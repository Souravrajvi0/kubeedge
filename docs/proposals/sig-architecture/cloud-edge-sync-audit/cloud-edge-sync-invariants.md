# Cloud–edge sync invariants

This document records the *stated* and *implemented* consistency model for
KubeEdge CloudCore ↔ EdgeCore object and membership sync. It is the
companion to `device-twin-state-map.md`.

## Components

| Plane | Module | Role |
| --- | --- | --- |
| CloudCore | CloudHub | Node sessions, ACK / no-ACK queues, ObjectSync checkpoints |
| CloudCore | EdgeController / DynamicController | Reflect Kubernetes lists/watches into downstream messages |
| CloudCore | DeviceController | Device / DeviceModel / DeviceStatus CRDs, membership, twin upstream |
| CloudCore | SyncController | Reconcile ObjectSync / ClusterObjectSync vs live API objects |
| CloudCore | Router | Rule / RuleEndpoint execution |
| EdgeCore | EdgeHub | Connection, keepalive, cert rotation, dispatch by message group |
| EdgeCore | MetaManager | SQLite `meta` + `meta_v2` (metaserver imitator), ACK to cloud |
| EdgeCore | DeviceTwin | Membership, MQTT twins, DMI mapper cache |
| EdgeCore | EventBus / ServiceBus | MQTT and HTTP edge buses |

## Message transport

CloudHub documents two modes (`cloud/pkg/cloudhub/dispatcher/message_dispatcher.go`):

1. **ACK mode.** Edge must ACK after MetaManager stores the object. If the
   session drops before ACK, CloudHub resends until ACK. Messages are keyed by
   object UID. A newer resourceVersion replaces an older queued message; a
   delete always wins.
2. **NO-ACK mode.** CloudHub assumes delivery. Used for podlist snapshots,
   membership, `twin/cloud_updated`, CSI volume ops, application responses,
   user-group, task manager, and several node/pod status patches.

`TrimMessage` strips `node/{nodeID}/` before the bytes hit the wire so EdgeCore
sees `{namespace}/{type}/{name}`.

## Kubernetes reflection

- Downstream: watch/informers → ACK messages with `Header.ResourceVersion`.
- Checkpoint: ObjectSync / ClusterObjectSync `status.objectResourceVersion`.
- SyncController: if live RV > synced RV, replay an update. Empty synced RV
  must be treated as never-acked (`0`), otherwise a Create+failed UpdateStatus
  race black-holes the object forever (CloudHub enqueue *and* SyncController
  both used to `return false` / `return` on empty status).
- Deletes are always enqueued, including objects with a deletionTimestamp.

**Eventual consistency (ACK objects):** after a partition heals, unacked
messages retry; SyncController also replays if the API object moved ahead of
ObjectSync. Convergence is expected; delay is not a correctness failure.

**Not guaranteed:** total order of no-ACK messages; exactly-once delivery
(replays are expected); metaserver watch events for writes that never reached
`meta_v2`.

## Edge MetaManager

On insert/update/delete from the cloud:

1. `imitator.Inject` writes `meta_v2` and triggers watches.
2. `handleMessage` writes the legacy `meta` SQLite table.
3. Response `OK` is sent to CloudHub.

`meta_v2` must be monotonic in resourceVersion: an incoming object with a
smaller RV than the stored row is skipped. (Previously a TODO; stale full
resyncs / retries could otherwise roll the metaserver cache backwards.)

Known unfixed sibling (tracked upstream, not re-fixed here): Inject write
failures were ACKed as OK — kubeedge/kubeedge#7070 / PR #7082.

## Partition → autonomy → heal

1. EdgeHub publishes `CloudDisconnected` to twin/meta/bus/task groups.
2. Edged and MetaManager keep serving from SQLite; pods keep running.
3. DeviceTwin **must not drop** reported twin/state updates: they stay in
   `ConfirmMap` and retry after reconnect (previously `dealSendToCloud`
   returned without storing while disconnected).
4. On reconnect, DeviceTwin sends `membership/detail`. Cloud must answer with
   the current device set **and** a parent-ID so `ConfirmMap` is cleared
   (previously the upstream case was empty — kubeedge/kubeedge#7439 plus
   membership never reconverged).
5. ACK object queues drain; SyncController fills remaining gaps.

## Version skew

CloudCore and EdgeCore of mixed minor versions are not a supported
consistency domain. ObjectSync CRDs, DeviceStatus split, and message resource
shapes have changed across releases; a permanent desync after a partial
upgrade should be treated as an operations failure, not as delayed sync.

## Tests

- Unit: CloudHub dispatcher, SyncController `needsResourceSync`, imitator RV
  filter, DeviceTwin ConfirmMap while disconnected, membership snapshot.
- Integration: beehive channel tests under `edge/pkg/devicetwin` and
  `cloud/pkg/cloudhub`.
- E2E: `tests/e2e` (requires a cluster; not required to reproduce the
  handler-level bugs above).
