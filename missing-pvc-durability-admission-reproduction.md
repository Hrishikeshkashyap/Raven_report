# Missing PVC Durability Admission Bug Reproduction

This document records the October 12, 2026 reproduction of RAVEN-002 on the
`hrk-aks-scsi-poc` Kubernetes context.

**Status:** Resolved and verified on `hrk-aks-scsi-poc` on October 12, 2026.

## Bug Summary

A Pod references two claims:

- D is bound to a durability-mode local volume on node A.
- X does not exist when the Pod is admitted.

The Pod also supplies required node affinity for node B. When the webhook
cannot find X, it allows the entire Pod without applying D's required storage
affinity. Creating X later does not cause the existing Pod to pass through Pod
admission again, so its node affinity remains restricted to B.

Durability mode does not replicate the volume. Scheduling on a node other than
the node containing D can cause staging to create an empty replacement and can
make the original copy eligible for later cleanup.

The impact is namespace-scoped integrity and durability-policy enforcement. It
does not demonstrate a cross-tenant authorization bypass.

## Vulnerable Code

The early return occurs in `handler.Handle` while it resolves the Pod's claims:

```go
if err := retry.OnError(retry.DefaultRetry, isRetriableError, getPVC); err != nil {
    if apierrors.IsNotFound(err) {
        log.Info(
            "allowed pod with pvc not found",
            "pvc", pvcName,
            "pod", pod.Name,
            "namespace", pod.Namespace,
        )
        return admission.Allowed("pvc not found")
    }
    return admission.Errored(http.StatusBadRequest, err)
}
```

Any local PV names collected from earlier claims are discarded by this return,
so `injectNodeAffinity` is never called for D.

## Environment

The active cluster and release were:

```text
Kubernetes context: hrk-aks-scsi-poc
Helm release:        local-csi-driver
Release namespace:  kube-system
Chart version:       0.4.0
Manager image:       localcsidriver.azurecr.io/acstor/local-csi-manager:v0.4.0
Node A:              aks-scsipoc-34422432-vmss000000
Synthetic node B:    node-b
```

The installed hyperconverged webhook was enabled:

```text
NAME                               WEBHOOKS
csi-local-hyperconverged-webhook   1
```

The test used the installed image without modifying the Helm release.

## Safety Controls

The reproduction exercised admission only:

- The Pod used the nonexistent scheduler `raven-002-no-scheduler`.
- Node B was a synthetic hostname.
- No volume was staged or mounted.
- Both PVs used `Retain`.
- All resources were removed after the test.

The cluster's enforce-ephemeral webhook was also enabled. The test PVCs used
its supported `localdisk.csi.acstor.io/accept-ephemeral-storage: "true"`
annotation so the PVC requests could reach the state needed for this isolated
hyperconverged-webhook test.

## Create Bound Durability Claim D

The following resources represented existing durability data on node A:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: raven-002-repro
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: raven-002-local
provisioner: localdisk.csi.acstor.io
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: raven-002-d-pv
  annotations:
    localdisk.csi.acstor.io/selected-node:
      aks-scsipoc-34422432-vmss000000
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: raven-002-local
  csi:
    driver: localdisk.csi.acstor.io
    volumeHandle: raven-002-d-volume
    volumeAttributes:
      localdisk.csi.acstor.io/failover-mode: durability
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: durability-data
  namespace: raven-002-repro
  annotations:
    localdisk.csi.acstor.io/accept-ephemeral-storage: "true"
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: raven-002-local
  volumeName: raven-002-d-pv
```

The claim bound successfully:

```text
PVC               STATUS   VOLUME
durability-data   Bound    raven-002-d-pv

PV               STATUS   NODE
raven-002-d-pv   Bound    aks-scsipoc-34422432-vmss000000
```

## Create the Pod While X Is Missing

The Pod referenced D first and absent X second. Its caller-supplied affinity
required node B:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: raven-002-pod
  namespace: raven-002-repro
spec:
  schedulerName: raven-002-no-scheduler
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node-b
  containers:
  - name: pause
    image: mcr.microsoft.com/oss/kubernetes/pause:3.6
    volumeMounts:
    - name: durability-data
      mountPath: /durability
    - name: missing-data
      mountPath: /missing
  volumes:
  - name: durability-data
    persistentVolumeClaim:
      claimName: durability-data
  - name: missing-data
    persistentVolumeClaim:
      claimName: missing-x
```

The API server created the Pod. Its stored affinity contained only node B:

```text
creationTimestamp: 2026-10-12T08:53:27Z
resourceVersion:   5313661
phase:             Pending

requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: kubernetes.io/hostname
      operator: In
      values:
      - node-b
```

Node A was absent. The durability placement requirement had not been injected.

The manager log identifies the short-circuit:

```text
handler.go:71 "pod handling request"
  kind="Pod" request="raven-002-pod" operation="CREATE"
handler.go:112 "allowed pod with pvc not found"
  pvc="missing-x" pod="raven-002-pod" namespace="raven-002-repro"
```

There was no subsequent `starting to inject storage affinity` log for this
Pod.

## Create X After Pod Admission

X was then created and bound to a synthetic volume on node B:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: raven-002-x-pv
  annotations:
    localdisk.csi.acstor.io/selected-node: node-b
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: raven-002-local
  csi:
    driver: localdisk.csi.acstor.io
    volumeHandle: raven-002-x-volume
    volumeAttributes:
      localdisk.csi.acstor.io/failover-mode: availability
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: missing-x
  namespace: raven-002-repro
  annotations:
    localdisk.csi.acstor.io/accept-ephemeral-storage: "true"
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: raven-002-local
  volumeName: raven-002-x-pv
```

X reached `Bound`, but the existing Pod remained unchanged:

```text
creationTimestamp: 2026-10-12T08:53:27Z
resourceVersion:   5313661
phase:             Pending

requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: kubernetes.io/hostname
      operator: In
      values:
      - node-b
```

The identical creation timestamp and resource version confirm that creating X
did not mutate or recreate the Pod. In a schedulable version of this setup, the
Pod could proceed using its original node B affinity without another Pod
admission request.

## Control Request

A server-side dry run created a control Pod referencing only D and without
caller affinity:

```powershell
kubectl create --dry-run=server -f .\durability-control.yaml `
    -o jsonpath='{.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution}'
```

The same live webhook injected node A:

```json
{
  "nodeSelectorTerms": [
    {
      "matchExpressions": [
        {
          "key": "kubernetes.io/hostname",
          "operator": "In",
          "values": [
            "aks-scsipoc-34422432-vmss000000"
          ]
        }
      ]
    }
  ]
}
```

The corresponding logs showed normal durability processing:

```text
request="raven-002-control" operation="CREATE"
pod="raven-002-control"
nodes=["aks-scsipoc-34422432-vmss000000"]
failoverMode="durability"
"applying required node affinity for durability mode"
```

This control confirms that D was resolvable and that the missing X path, rather
than invalid D metadata or an inactive webhook, prevented the mutation.

## Result

RAVEN-002 was reproduced:

1. D was bound in durability mode on node A.
2. A Pod referenced D and absent X while requiring node B.
3. The webhook allowed the Pod without injecting D's node A restriction.
4. The Pod was stored with only the node B requirement.
5. X was created and bound.
6. The existing Pod retained the same resource version and node B affinity.
7. A control request using only D correctly received required affinity to A.

## Cleanup

The following commands removed all reproduction resources:

```powershell
kubectl delete namespace raven-002-repro `
    --ignore-not-found `
    --wait=true

kubectl delete persistentvolume `
    raven-002-d-pv `
    raven-002-x-pv `
    --ignore-not-found

kubectl delete storageclass raven-002-local `
    --ignore-not-found
```

Cleanup verification:

```text
namespace "raven-002-repro" deleted
persistentvolume "raven-002-d-pv" deleted
persistentvolume "raven-002-x-pv" deleted
storageclass.storage.k8s.io "raven-002-local" deleted
PASS: all RAVEN-002 reproduction resources removed.
```

The Helm release and installed images were not changed during the original
reproduction.

## Fixed-Image Verification

The working-tree fix was built into temporary driver and manager images:

```text
Driver image:
  hrkashyapacr.azurecr.io/acstor/local-csi-driver:
  raven002-fix-20261012-1917
Driver digest:
  sha256:9a62ffc82139ac915f39c434a2fe4af926d67f9e78fb06afcb7c57fafab7026b

Manager image:
  hrkashyapacr.azurecr.io/acstor/local-csi-manager:
  raven002-fix-20261012-1917
Manager digest:
  sha256:7f9001f32dd2ed75d88d4d7f039df031db476815081408cbe1707d79350a658f
```

The temporary images were deployed as Helm revision 3. Both manager replicas
and the single driver DaemonSet Pod became Ready before testing.

### Missing Ordinary PVC

The fixed webhook was tested with D followed by missing X and with missing X
followed by D. Both CREATE requests were rejected:

```text
D then X:
admission webhook "hyperconverged.localdisk.csi.acstor.io" denied the request:
PVC raven-002-fixed/missing-x does not exist; create it before retrying Pod
creation

X then D:
admission webhook "hyperconverged.localdisk.csi.acstor.io" denied the request:
PVC raven-002-fixed/missing-x does not exist; create it before retrying Pod
creation

stored_pods=0
```

This forces callers to retry Pod creation after an ordinary claim exists.
Placement is recalculated during that new admission request, so a missing
claim can no longer discard another volume's durability restriction.

### Missing Generic-Ephemeral Claim

A server-side dry run used a generic-ephemeral volume before Kubernetes had
created its generated claim. The webhook derived the expected claim name from
the Pod and volume names, continued processing D, and injected node A:

```text
generated_claim_name=raven-002-ephemeral-generated-data
generated_claim_persisted=False
required_hostnames=aks-scsipoc-34422432-vmss000000
```

The dry run confirms that a legitimately absent generated claim does not
short-circuit placement for an already-bound durability volume.

### Durability Topology Retention

A `WaitForFirstConsumer` StorageClass was created with:

```yaml
parameters:
  volumeGroup: ssdcontainerstorage
  localdisk.csi.acstor.io/failover-mode: durability
```

The dynamically provisioned claim bound successfully, its consumer reached
`Running`, and the resulting PV retained both the initial owner and hard
topology:

```text
pv=pvc-05a7b410-ea1d-4336-81c0-4bf0e682c075
failover_mode=durability
selected_initial_node=aks-scsipoc-34422432-vmss000000
pv_topology=aks-scsipoc-34422432-vmss000000
consumer_phase=Running
```

This verifies the binding-aware placement backstop for claims that are
unbound when the Pod first passes admission.

### Missing Durability LV

A bound synthetic durability PV referenced an LV that did not exist. The Pod
was assigned to the recorded owner node. `NodeStageVolume` returned
`NotFound`, and an LVM query before and after staging confirmed that no
replacement LV was allocated:

```text
pod_phase=Pending
event_reason=FailedMount
event_message=MountVolume.MountDevice failed for volume
"raven-002-missing-lv": rpc error: code = NotFound desc = volume not found:
logical volume ssdcontainerstorage/raven-002-missing-lv does not exist
lv_created=False
```

This confirms that durability staging no longer repairs missing data by
creating an empty volume. Availability-mode recovery behavior was not changed.

### Combined Regression Checks

The same temporary images were also checked for RAVEN-003, RAVEN-004, and
RAVEN-005:

```text
D_then_V_required=aks-scsipoc-34422432-vmss000000
V_then_D_required=aks-scsipoc-34422432-vmss000000

conflict_A_then_B_exit=1
conflict_B_then_A_exit=1
denial=durability volumes have no common storage node

same_node_required=aks-scsipoc-34422432-vmss000000

caller_term_count=1
caller_hostname_requirements=
node-b AND aks-scsipoc-34422432-vmss000000
```

Mixed availability and durability order no longer changes the result.
Different-node durability volumes are denied in both orders, same-node
durability volumes require only that node, and caller affinity is ANDed with
storage affinity in one selector term.

### One-Node Limitation

The cluster has one real node. A Pod forced onto that node while its
durability volume recorded synthetic node B was rejected by kubelet's
NodeAffinity predicate before CSI staging:

```text
Reason:  NodeAffinity
Message: Pod was rejected: Predicate NodeAffinity failed:
node(s) didn't match Pod's node affinity/selector
```

No LV was created. This validates the admission and kubelet placement barrier,
but the live cluster could not directly invoke the wrong-owner
`NodeStageVolume` branch. That branch is covered by focused node unit tests.

## Fixed-Image Cleanup

The test namespace, six static PVs, dynamic PV, two StorageClasses, and all
test LVs were removed. The release was rolled back to revision 1, producing
revision 4:

```text
REVISION  STATUS    DESCRIPTION
4         deployed  Rollback to 1

DaemonSet/csi-local-node:
  localcsidriver.azurecr.io/acstor/local-csi-driver:v0.4.0
Deployment/csi-local-manager:
  localcsidriver.azurecr.io/acstor/local-csi-manager:v0.4.0
```

The original DaemonSet and both manager replicas were Ready after rollback.
No `raven-002` namespace, PV, StorageClass, or LV remained. The temporary ACR
tags and local Docker images were also deleted.
