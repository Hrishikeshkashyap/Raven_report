# RAVEN-010: Failover cleanup cluster test

## Problem statement

Cleanup can act on an outdated ownership decision:

1. A Pod uses its original, data-bearing logical volume (LV) on node A.
2. The Pod moves to B. Staging records B as selected node and can create an
   empty local replacement. A's original data stays on A.
3. Cleanup on A reads selected-node B and decides A's LV is obsolete.
4. Before cleanup retires it, the Pod returns to A. Staging reuses the original
   LV and records A as selected node again.
5. Cleanup continues using its earlier decision, without coordinating with
   staging or checking current ownership before retirement.

**The concern is concurrent reuse and retirement, not disk/data movement.**
Default retirement quarantines/renames the LV for later wiping; the reaper
also checks open state. Immediate destruction of an actively reused LV is
not a guaranteed outcome.

## Verdict

**NOT REPRODUCED in three real A -> B -> A round trips.**

Selected-node changes, empty replacement creation, and old-node cleanup were
observed. However, every return to A used a **new LV UUID**, not the original
LV. These runs exercised the availability-policy replacement path, not the
critical concurrent-reuse interleaving. This does not prove RAVEN-010 is fixed.

## Setup and method

- **Date/context:** 2026-10-20, `hrk-aks-nvme`.
- **A:** `aks-nodepool1-74017372-vmss000000`;
  **B:** `aks-nodepool1-74017372-vmss000003`.
- Both nodes Ready; Kubernetes `v1.35.7`, Ubuntu 24.04.5, `Standard_L8s_v3`.
- **Driver:** `hrkashyapacr.azurecr.io/acstor/local-csi-driver:scsi-poc-v1`;
  binary reports commit `4473ac8-dirty`. Not rebuilt from this checkout;
  exact source equivalence is unverified.
- **Resolved image digest:**
  `sha256:a26b17d93967dacc67da14a477194b9453cd83e8c8782732899f9adce78af55a`.
- With explicit approval, initialized `/dev/nvme0n1` on both nodes after
  inspection found no reported filesystem, partitions, or LVM volumes.
  OS disks were excluded.
- Temporarily enabled `--run-alongside-webhook=true`, restricted discovery to
  `/dev/nvme0n1`, and added the node labels needed to start the driver.
  Failover/orphan GC and wiping remained enabled.
- Used a disposable **128Mi ReadWriteOnce PVC**, availability-mode
  StorageClass, and BusyBox Pod in `raven010-test-20261020`.
- Wrote `ORIGINAL_DATA_ON_A` to `/data/raven010-marker`, then normally deleted
  and recreated the Pod on B and A three times, waiting for Ready each time.
  Recorded ownership, LV UUIDs, and marker contents at each placement.
- Explicit hostname selectors controlled placement; no hyperconverged webhook
  was installed. No artificial cleanup pause or fault injection was used.

**Test PV:** `pvc-1bbf80c1-75d1-4c1a-85ca-8a4bffd9aabb`  
**Volume handle:** `containerstorage#pvc-1bbf80c1-75d1-4c1a-85ca-8a4bffd9aabb`

## Results

Snapshot times are IST (UTC+05:30). UUID prefixes below distinguish the LVs;
the first-cycle excerpt below includes full UUIDs.

| Time | Placement | Selected node | LV UUID prefix | Original marker |
| --- | --- | --- | --- | --- |
| 17:01:17 | Initial A | Initial-node fallback A | `g6VMbQ` | Present |
| 17:01:29 | Cycle 1 B | B | `mM49Sg` | Absent |
| 17:01:36 | Cycle 1 return A | A | `NecKep` | Absent |
| 17:01:43 | Cycle 2 B | B | `w9PhNH` | Absent |
| 17:01:50 | Cycle 2 return A | A | `TNuSpN` | Absent |
| 17:01:57 | Cycle 3 B | B | `euTM4Q` | Absent |
| 17:02:04 | Cycle 3 return A | A | `E7votD` | Absent |

All seven placements reached Ready. The PV had no node affinity throughout.
At every post-move inventory, the other node had no listed test LV.

### Important log evidence

**First-cycle inventory/marker excerpts** from the test transcript:

```text
17:01:17 IST - initial A
Selected: null; Initial: aks-nodepool1-74017372-vmss000000
A LV UUID: g6VMbQ-qYyd-Rg2K-4CVK-SwOB-Sd1L-dpTDq0
ORIGINAL_DATA_ON_A

17:01:29 IST - on B
Selected: aks-nodepool1-74017372-vmss000003
B LV UUID: mM49Sg-qCUs-ajLY-gvyP-skCw-eSVx-xhBL7h
A LV inventory: empty
MARKER_ABSENT

17:01:36 IST - return to A
Selected: aks-nodepool1-74017372-vmss000000
A LV UUID: NecKep-6AB1-zuJh-dbQu-K80x-rRP3-FYNQLb
MARKER_ABSENT
```

**A-side cleanup log excerpts** (UTC; repetitive metadata omitted):

```text
11:31:23.688Z failover_pv_cleanup.go:100
  "Detected node annotation mismatch, checking if LVM volume should be cleaned up"
11:31:23.783Z failover_pv_cleanup.go:142
  "Successfully cleaned up orphaned LVM volume"
11:31:23.836Z reaper.go:216
  "wiping quarantined volumes" due=1 quarantined=1
11:31:23.988Z reaper.go:228
  "wiped and removed quarantined volume"
  vg="containerstorage"
  lv="local-csi-wipe-fea411fc-0bb0-4b0d-ac32-8ca39a0f17e5"
```

This cleanup finished at **17:01:23 IST**, before the B snapshot and the
return to A. A's original LV was already retired; the missing marker on return
therefore **does not demonstrate stale-decision cleanup**. No retirement of
an actively reused original LV was observed.

## Cleanup and remaining side effect

Test Pod/PVC/PV, StorageClass, and both temporary namespaces were removed.
Original driver arguments were restored and compared exactly; temporary node
labels were removed. Both nodes remained Ready.

```text
Final LV inventory on node B: empty
Final LV inventory on node A: empty
VERIFIED: original driver arguments restored
csi-local-node: DESIRED=0 CURRENT=0 READY=0 (matches pre-test state)
Final PVC query: No resources found
Final PV query:  No resources found
```

**NVMe initialization and `containerstorage` VG provisioning remain.**
Restoration did not return disks to their original raw state. No existing
workload or repository source was changed.

## Limits and next step

Two PowerShell argument-quoting errors interrupted initial harness commands
before failover. They were corrected; the complete three-cycle run then
finished with exit code 0. Both failures remain in the transcript.

A conclusive reproduction needs a test-only pause **after cleanup decides B
owns the PV**, verified reuse of the original LV by staging on A, and then
release of that cleanup. This run did not force that ordering.

## Full evidence (optional)

- [Test transcript](./raven010-test-output.txt)
  and [cleanup transcript](./raven010-cleanup-output.txt).
- Driver logs: [A](./aks-nodepool1-74017372-vmss000000-raven010-driver.log),
  [B](./aks-nodepool1-74017372-vmss000003-raven010-driver.log).
- Snapshots: [PV](./raven010-pv.yaml), [events](./raven010-events.json),
  [driver Pods/image IDs](./raven010-driver-pods.json),
  [original DaemonSet](./raven010-original-ds.json).
- Scripts: [inspection](./raven010-inspect.ps1),
  [enablement](./raven010-enable.ps1), [test](./raven010-test.ps1),
  [cleanup](./raven010-cleanup.ps1).
