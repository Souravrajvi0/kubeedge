# Device twin state map

## Objects

| Object | Writer | Edge store | Cloud store |
| --- | --- | --- | --- |
| Device CR (`devices.kubeedge.io`) | apiserver / mapper | MetaManager `meta` + DMI cache | etcd |
| DeviceModel CR | apiserver | MetaManager `meta` + DMI cache | etcd |
| DeviceStatus CR | DeviceController upstream | not authoritative | etcd (`status.twins`, `state`) |
| Twin MQTT document | mapper / MQTT client | DeviceTwin in-memory + SQLite device twin rows | DeviceStatus.twins |
| Membership | DeviceController downstream | DeviceTwin `DeviceList` | DeviceController `Device` map |

Desired vs reported:

- **Desired (expected):** cloud DeviceStatus / MQTT `$hw/events/device/{ns}/{id}/twin/cloud_updated` / Device spec properties for DMI.
- **Reported (actual):** MQTT `$hw/events/device/{ns}/{id}/twin/update` → DeviceTwin `RestDealType` → `twin/edge_updated` to cloud → patch DeviceStatus.

## Twin worker state machine (`DealMsgTwin`)

`dealType`:

| Value | Name | Source |
| --- | --- | --- |
| 0 | RestDealType | MQTT / mapper update |
| 1 | SyncDealType | cloud `TwinCloudSync` |
| 2 | DetailDealType | cloud detail |
| 3 | SyncTwinDeleteDealType | delete while syncing |

Per property key:

1. Unknown key → `dealTwinAdd` (validate type, bump edge or cloud version).
2. Delete: REST `msgTwin == nil`, or sync metadata.type `deleted` → `dealTwinDelete`.
3. Else `dealTwinCompare` expected then actual.

Versions (`dealVersion`):

- REST: `EdgeVersion++`.
- Sync: reject if local CloudVersion > request CloudVersion **or** local EdgeVersion > request EdgeVersion; else copy both counters. On reject, edge sends `edge_updated` with its copy (`SyncResult`) so the cloud can catch up.

Delta MQTT topic is published when expected ≠ actual after the deal.

**Invariant:** a TwinCloudSync payload that omits `metadata` must not panic the TwinWorker. Operator precedence in the old delete check evaluated `msgTwin.Metadata.Type` even when `Metadata` was nil (kubeedge/kubeedge#6950).

## Topics and ACKs

| Resource / topic | ACK? | Notes |
| --- | --- | --- |
| `{ns}/device/{name}` insert/update/delete | yes (resource group) | MetaManager then DMI (`MetaDeviceOperation`) |
| `node/{id}/membership` add/remove | **no** | fire-and-forget; heal via `membership/detail` |
| `membership/detail` get | response with parent ID | full snapshot of devices for that node |
| `twin/cloud_updated` | **no** | desired apply; loss is healed by Device CR + detail |
| `device/{id}/twin/edge_updated` | retry via ConfirmMap | reported / expected observed |

Cloud `Device` messages are routed MetaGroup → DeviceTwin with source rewritten to `meta`, so `classifyMsg` uses `MetaDeviceOperation` (DMI), not the legacy `DeviceUpdated` path.

## Partition behavior

- While disconnected, reported twins stay in `ConfirmMap` and are resent after connect + `checkConfirm` (60s) or the next send.
- Membership adds/removes that were no-ACK during the partition are reconstructed from `devicesForNode` on `membership/detail`.
- DMI cache is rebuilt from MetaManager SQLite on EdgeCore restart (`initDeviceInfoFromDB`). Twin `DeviceList` is rebuilt from SQLite (`SyncSqlite`) plus membership snapshot after connect.

## Rules

Rule and RuleEndpoint messages only honored insert/delete. An update of a live Rule left the old source listener registered and the new target unused. Updates now unregister then re-add.
