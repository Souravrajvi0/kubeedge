# Findings (classified)

## This PR (fixes + tests)

| ID | Class | Invariant | Repro | Dedup |
| --- | --- | --- | --- | --- |
| F1 | B | ObjectSync / ClusterObjectSync with **empty** `status.objectResourceVersion` must still enqueue and later replay. Empty status is never-acked, not “already synced”. | Dispatcher test: ObjectSync status `""` + pod RV `1` must land in the ACK store. `needsResourceSync("1","") == true`. | Not filed as this race. Related: reliable-message-delivery proposal. |
| F2 | B | `meta_v2` must not apply a smaller resourceVersion over a larger one. | `shouldApplyIncomingResourceVersion(11,10) == false`. | Local TODO in `imitator.go`; not an issue. |
| F3 | B | Twin reported updates during partition must survive until reconnect. | `dealSendToCloud` while `Disconnected` stores `ConfirmMap`. | Not filed (distinct from #7439). |
| F4 | B | After reconnect, `membership/detail` must return the node’s devices and ACK the request (parent ID). | `TestHandleMembershipDetail`. | Completes #7439 (cloud never answered; ConfirmMap resend storm) **and** restores membership convergence. |
| F5 | B | TwinCloudSync with existing key and nil `metadata` must not panic TwinWorker. | `TestDealMsgTwinSyncNilMetadataDoesNotPanic`. | #6950 (open, unfixed). |
| F6 | A | DMI `default` log used `resources[3]` after requiring `len==3` (index panic). | Static. | Not filed. |
| F7 | A | Rule / RuleEndpoint **update** must rebind listeners. | Code path `model.UpdateOperation`. | Not filed. |

## Deduped, not re-implemented here

| Upstream | Class | Notes |
| --- | --- | --- |
| #7070 / PR #7082 (lgtm, unmerged) | B | Inject / processVolume ACK-on-failure. |
| #6856 | A | `DeleteMetaByKeyAndPodUID` LIKE vs exact UID. |
| #4582 | B | MetaServer namespace watch ordering (older, still open). |
| #6950 | B | Fixed in this PR as F5. |
| #7439 | B | Fixed in this PR as F4. |

## Not bugs (acceptable delay)

- ACK retries and 5s CloudHub resend are delay, not divergence, as long as ObjectSync moves forward.
- MQTT twin EdgeVersion increment on a REST update even when the string is unchanged is noisy but still converges via `edge_updated`.
- CloudCore vs EdgeCore skew is unsupported; not treated as a code defect.

## Classification key

- **A** — localized handler / parser / log / store.
- **B** — sync, twin, or partition-recovery correctness.
- **C** — systemic meta-sync architecture inconsistency (none newly claimed beyond the documented ACK vs no-ACK split, which is by design).
