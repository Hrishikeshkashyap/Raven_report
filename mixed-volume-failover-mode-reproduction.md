# Mixed-Volume Failover Mode Order Bug Reproduction

This document records the October 9, 2026 reproduction of the mixed-volume
failover-mode bug on the `hrk-aks-nvme` Kubernetes context.

**Status:** Resolved and verified on `hrk-aks-nvme` on October 12, 2026.

This is separate from the required-affinity OR-term bug documented in
`affinity-bug-reproduction.md`.

## Bug Summary

A Pod uses two bound local CSI claims:

- D is a durability-mode volume assigned to node A.
- V is an availability-mode volume assigned to node B.

The webhook currently:

1. Appends every PV node to one shared node list.
2. Replaces the selected failover mode on every loop iteration.
3. Applies the last PV's mode to the union of all PV nodes.

The resulting mutation depends on the order of the Pod's volumes:

| Volume order | Current behavior             | Required behavior           |
|--------------|------------------------------|-----------------------------|
| D, then V    | Preferred affinity to A or B | Required affinity to A only |
| V, then D    | Required affinity to A or B  | Required affinity to A only |

Both current results permit placement on B. Staging can then recreate D empty
on B and make its old copy on A eligible for cleanup.

The impact is namespace-scoped integrity and data retention. It does not grant
an additional authorization privilege.

## Relevant Code

The behavior comes from `injectNodeAffinity`:

```go
var pvNodes []string
var failoverMode string

for _, pvName := range pvNames {
    var pvNodeList []string
    pvNodeList, failoverMode, response =
        h.getPvNodesAndFailoverMode(ctx, pvName)

    pvNodes = append(pvNodes, pvNodeList...)
}

return h.patchPodWithNodeAffinity(pod, pvNodes, failoverMode)
```

`pvNodes` accumulates nodes from all volumes, but `failoverMode` is overwritten
for each volume. The final value therefore comes from the last volume.

## Environment

The cluster initially had one real node:

```text
aks-nodepool1-74017372-vmss000000
```

The reproduction used:

```text
Node A: aks-nodepool1-74017372-vmss000000
Node B: node-b
```

Node B was a synthetic selected-node value. Both Pods used a nonexistent
scheduler, so no volume was staged or mounted. The test exercised only the
admission mutation.

The original Helm values had the hyperconverged webhook disabled:

```json
{
  "cleanup": {
    "pvCleanup": {
      "enabled": false
    }
  },
  "daemonset": {
    "nodeSelector": {
      "local-csi-nvme-poc": "true"
    }
  },
  "image": {
    "driver": {
      "pullPolicy": "Always",
      "repository": "hrkashyapacr.azurecr.io/acstor/local-csi-driver",
      "tag": "scsi-poc-v1"
    }
  },
  "webhook": {
    "enforceEphemeral": {
      "enabled": false
    },
    "hyperconverged": {
      "enabled": false
    }
  }
}
```

## Build the Reproduction Image

The image was built from `HEAD` plus the current working-tree copy of
`internal/webhook/hyperconverged/handler.go`. This included the earlier
OR-term fix while leaving the mixed-volume behavior unchanged.

```powershell
az acr login --name hrkashyapacr

$ctx = Join-Path $env:TEMP 'local-csi-mixed-mode-context'
$archive = Join-Path $env:TEMP 'local-csi-mixed-mode-context.tar'

if (Test-Path $ctx) {
    Remove-Item -Recurse -Force $ctx
}
if (Test-Path $archive) {
    Remove-Item -Force $archive
}

New-Item -ItemType Directory -Path $ctx | Out-Null
git archive --format=tar HEAD -o $archive
tar -xf $archive -C $ctx

Copy-Item `
    .\internal\webhook\hyperconverged\handler.go `
    (Join-Path $ctx 'internal\webhook\hyperconverged\handler.go') `
    -Force

$script = Join-Path $ctx 'hack\generate-notice.sh'
$crlf = [string][char]13 + [char]10
$lf = [string][char]10
$content = [IO.File]::ReadAllText($script).Replace($crlf, $lf)
[IO.File]::WriteAllText(
    $script,
    $content,
    [Text.UTF8Encoding]::new($false)
)

docker build `
    --pull `
    --build-arg GIT_COMMIT=mixed-mode-repro `
    --build-arg BUILD_ID=mixed-mode-repro-20261009 `
    -t hrkashyapacr.azurecr.io/acstor/local-csi-manager:mixed-mode-repro-20261009 `
    -f (Join-Path $ctx 'Dockerfile.manager') `
    $ctx

docker push `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:mixed-mode-repro-20261009

Remove-Item -Recurse -Force $ctx
Remove-Item -Force $archive
```

Build output:

```text
mixed-mode-repro-20261009:
digest: sha256:a94b59d86cd077246e9b3e1f82c3fe508742ed366a5d96558b97074843c97c56
```

## Deploy the Reproduction Webhook

The active Helm revision was recorded before the upgrade:

```powershell
$release = (
    helm list `
        --kube-context hrk-aks-nvme `
        -n kube-system `
        -o json |
    ConvertFrom-Json |
    Where-Object name -EQ 'local-csi-driver'
)

$release.revision
```

The recorded rollback revision was `5`.

The temporary image and webhook were deployed with:

```powershell
helm upgrade local-csi-driver `
    .\charts\latest `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --reuse-values `
    --set webhook.hyperconverged.enabled=true `
    --set `
image.manager.repository=hrkashyapacr.azurecr.io/acstor/local-csi-manager `
    --set image.manager.tag=mixed-mode-repro-20261009 `
    --set manager.deployment.replicas=1 `
    --rollback-on-failure `
    --wait `
    --timeout 5m
```

Deployment output:

```text
STATUS: deployed
REVISION: 6

deployment.apps/csi-local-manager   1/1
pod/csi-local-manager-cb55f95b4-zwbfn   1/1   Running

NAME                               WEBHOOKS
csi-local-hyperconverged-webhook   1
```

## Reproduction Resources

The following manifest creates:

- Durability PV D on node A.
- Availability PV V on node B.
- One bound PVC for each PV.
- A Pod listing D before V.
- A Pod listing V before D.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: csi-mixed-mode-repro
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-mixed-mode-durability
provisioner: localdisk.csi.acstor.io
volumeBindingMode: Immediate
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-mixed-mode-availability
provisioner: localdisk.csi.acstor.io
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-mixed-mode-durability-pv
  annotations:
    localdisk.csi.acstor.io/selected-node:
      aks-nodepool1-74017372-vmss000000
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: csi-mixed-mode-durability
  csi:
    driver: localdisk.csi.acstor.io
    volumeHandle: csi-mixed-mode-durability-volume
    volumeAttributes:
      localdisk.csi.acstor.io/failover-mode: durability
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-mixed-mode-availability-pv
  annotations:
    localdisk.csi.acstor.io/selected-node: node-b
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: csi-mixed-mode-availability
  csi:
    driver: localdisk.csi.acstor.io
    volumeHandle: csi-mixed-mode-availability-volume
    volumeAttributes:
      localdisk.csi.acstor.io/failover-mode: availability
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: durability-data
  namespace: csi-mixed-mode-repro
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-mixed-mode-durability
  volumeName: csi-mixed-mode-durability-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: availability-data
  namespace: csi-mixed-mode-repro
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-mixed-mode-availability
  volumeName: csi-mixed-mode-availability-pv
---
apiVersion: v1
kind: Pod
metadata:
  name: durability-then-availability
  namespace: csi-mixed-mode-repro
spec:
  schedulerName: mixed-mode-no-scheduler
  containers:
  - name: pause
    image: mcr.microsoft.com/oss/kubernetes/pause:3.6
    volumeMounts:
    - name: durability-data
      mountPath: /durability
    - name: availability-data
      mountPath: /availability
  volumes:
  - name: durability-data
    persistentVolumeClaim:
      claimName: durability-data
  - name: availability-data
    persistentVolumeClaim:
      claimName: availability-data
---
apiVersion: v1
kind: Pod
metadata:
  name: availability-then-durability
  namespace: csi-mixed-mode-repro
spec:
  schedulerName: mixed-mode-no-scheduler
  containers:
  - name: pause
    image: mcr.microsoft.com/oss/kubernetes/pause:3.6
    volumeMounts:
    - name: availability-data
      mountPath: /availability
    - name: durability-data
      mountPath: /durability
  volumes:
  - name: availability-data
    persistentVolumeClaim:
      claimName: availability-data
  - name: durability-data
    persistentVolumeClaim:
      claimName: durability-data
```

Save the manifest as `mixed-mode-repro.yaml`, then apply it:

```powershell
kubectl --context hrk-aks-nvme `
    apply -f .\mixed-mode-repro.yaml
```

## Confirm Both Claims Are Bound

```powershell
kubectl --context hrk-aks-nvme `
    -n csi-mixed-mode-repro `
    get pvc
```

Output:

```text
NAME                STATUS   VOLUME
availability-data   Bound    csi-mixed-mode-availability-pv
durability-data     Bound    csi-mixed-mode-durability-pv
```

The PVs were also bound:

```text
NAME                             STATUS   CLAIM
csi-mixed-mode-durability-pv     Bound    csi-mixed-mode-repro/durability-data
csi-mixed-mode-availability-pv   Bound    csi-mixed-mode-repro/availability-data
```

## D, Then V Result

The mutation was retrieved with:

```powershell
kubectl --context hrk-aks-nvme `
    -n csi-mixed-mode-repro `
    get pod durability-then-availability `
    -o jsonpath='{.spec.affinity.nodeAffinity}'
```

Observed output:

```json
{
  "preferredDuringSchedulingIgnoredDuringExecution": [
    {
      "preference": {
        "matchExpressions": [
          {
            "key": "kubernetes.io/hostname",
            "operator": "In",
            "values": [
              "aks-nodepool1-74017372-vmss000000",
              "node-b"
            ]
          }
        ]
      },
      "weight": 100
    }
  ]
}
```

Webhook logs:

```text
pod="durability-then-availability"
pvName="csi-mixed-mode-durability-pv"
failoverMode="durability"

pvName="csi-mixed-mode-availability-pv"
failoverMode="availability"

nodes=["aks-nodepool1-74017372-vmss000000","node-b"]
failoverMode="availability"

"applying preferred node affinity for availability mode"
```

The last volume V changed the final mode to availability. The resulting
preferred affinity allows A or B instead of requiring A.

## V, Then D Result

The mutation was retrieved with:

```powershell
kubectl --context hrk-aks-nvme `
    -n csi-mixed-mode-repro `
    get pod availability-then-durability `
    -o jsonpath='{.spec.affinity.nodeAffinity}'
```

Observed output:

```json
{
  "requiredDuringSchedulingIgnoredDuringExecution": {
    "nodeSelectorTerms": [
      {
        "matchExpressions": [
          {
            "key": "kubernetes.io/hostname",
            "operator": "In",
            "values": [
              "node-b",
              "aks-nodepool1-74017372-vmss000000"
            ]
          }
        ]
      }
    ]
  }
}
```

Webhook logs:

```text
pod="availability-then-durability"
pvName="csi-mixed-mode-availability-pv"
failoverMode="availability"

pvName="csi-mixed-mode-durability-pv"
failoverMode="durability"

nodes=["node-b","aks-nodepool1-74017372-vmss000000"]
failoverMode="durability"

"applying required node affinity for durability mode"
```

The last volume D changed the final mode to durability, but the required node
list still contains both A and B.

## Automated Reproduction Assertion

The following check verifies both order-dependent results:

```powershell
$dv = kubectl `
    --context hrk-aks-nvme `
    -n csi-mixed-mode-repro `
    get pod durability-then-availability `
    -o jsonpath='{.spec.affinity.nodeAffinity}'

$vd = kubectl `
    --context hrk-aks-nvme `
    -n csi-mixed-mode-repro `
    get pod availability-then-durability `
    -o jsonpath='{.spec.affinity.nodeAffinity}'

if (
    $dv -notmatch 'preferredDuringSchedulingIgnoredDuringExecution' -or
    $dv -match 'requiredDuringSchedulingIgnoredDuringExecution'
) {
    throw 'Expected D,V to be preferred-only'
}

if (
    $vd -notmatch 'requiredDuringSchedulingIgnoredDuringExecution' -or
    $vd -match 'preferredDuringSchedulingIgnoredDuringExecution'
) {
    throw 'Expected V,D to be required-only'
}

foreach ($json in @($dv, $vd)) {
    if (
        $json -notmatch 'aks-nodepool1-74017372-vmss000000' -or
        $json -notmatch 'node-b'
    ) {
        throw 'Expected both A and B in each mutation'
    }
}
```

Result:

```text
PASS: bug reproduced with both claims Bound.
D,V produced preferred affinity to A or B.
V,D produced required affinity to A or B.
```

## Required Behavior

For either declaration order, the Pod uses durability data D on A. The safe
result is:

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: kubernetes.io/hostname
      operator: In
      values:
      - aks-nodepool1-74017372-vmss000000
```

The availability node B must not be included in the required storage-node set.

## Fix Verification

The fix was verified from these production commits:

```text
d55332f fix(webhook): enforce storage affinity in existing terms (RAVEN-005)
197bce9 fix(webhook): aggregate durability placement nodes
        (RAVEN-003, RAVEN-004)
```

The aggregation fix:

- Gives durability volumes precedence over availability volumes.
- Intersects the resolved node sets of multiple durability volumes.
- Uses availability nodes only when the Pod has no durability volumes.
- Denies admission when durability placement is missing or has no common node.

### Build and Deployment

A temporary manager image was built from commit `197bce9` and pushed to ACR:

```powershell
$ctx = Join-Path $env:TEMP 'local-csi-mixed-mode-fix-context'
$archive = Join-Path $env:TEMP 'local-csi-mixed-mode-fix-context.tar'

New-Item -ItemType Directory -Path $ctx | Out-Null
git archive --format=tar HEAD -o $archive
tar -xf $archive -C $ctx

$script = Join-Path $ctx 'hack\generate-notice.sh'
$crlf = [string][char]13 + [char]10
$lf = [string][char]10
$content = [IO.File]::ReadAllText($script).Replace($crlf, $lf)
[IO.File]::WriteAllText(
    $script,
    $content,
    [Text.UTF8Encoding]::new($false)
)

docker build `
    --pull `
    --build-arg GIT_COMMIT=197bce9 `
    --build-arg BUILD_ID=mixed-mode-fix-20261012 `
    -t hrkashyapacr.azurecr.io/acstor/local-csi-manager:mixed-mode-fix-20261012 `
    -f (Join-Path $ctx 'Dockerfile.manager') `
    $ctx

docker push `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:mixed-mode-fix-20261012

Remove-Item -Recurse -Force $ctx
Remove-Item -Force $archive
```

Image output:

```text
mixed-mode-fix-20261012:
digest: sha256:16d817aa0e0b639bdbed3a46181a6543564ae59ed118280009f2f58f9ddf4ded
```

The existing release revision was recorded as `7`. The fixed webhook was
deployed as revision `8`:

```powershell
helm upgrade local-csi-driver `
    .\charts\latest `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --reuse-values `
    --set webhook.hyperconverged.enabled=true `
    --set `
image.manager.repository=hrkashyapacr.azurecr.io/acstor/local-csi-manager `
    --set image.manager.tag=mixed-mode-fix-20261012 `
    --set manager.deployment.replicas=1 `
    --rollback-on-failure `
    --wait `
    --timeout 5m
```

Deployment output:

```text
STATUS: deployed
REVISION: 8

deployment.apps/csi-local-manager   1/1
pod/csi-local-manager-6db99f99b7-w2dqk   1/1   Running

NAME                               WEBHOOKS
csi-local-hyperconverged-webhook   1
```

### Bound-Claim Precondition

The PVs and PVCs were created before either Pod. The test waited for both claims
to become bound:

```powershell
kubectl --context hrk-aks-nvme `
    -n csi-mixed-mode-fix-verification `
    wait `
    --for=jsonpath='{.status.phase}'=Bound `
    pvc/durability-data `
    pvc/availability-data `
    --timeout=60s
```

Output:

```text
NAME                STATUS   VOLUME
availability-data   Bound    csi-mixed-fix-availability-pv
durability-data     Bound    csi-mixed-fix-durability-pv
```

The verification used:

```text
D: durability on aks-nodepool1-74017372-vmss000000
V: availability on node-b
```

The two Pods differed only in volume order.

### D, Then V Fixed Result

```powershell
kubectl --context hrk-aks-nvme `
    -n csi-mixed-mode-fix-verification `
    get pod durability-then-availability `
    -o jsonpath='{.spec.affinity.nodeAffinity}'
```

Output:

```json
{
  "requiredDuringSchedulingIgnoredDuringExecution": {
    "nodeSelectorTerms": [
      {
        "matchExpressions": [
          {
            "key": "kubernetes.io/hostname",
            "operator": "In",
            "values": [
              "aks-nodepool1-74017372-vmss000000"
            ]
          }
        ]
      }
    ]
  }
}
```

### V, Then D Fixed Result

```powershell
kubectl --context hrk-aks-nvme `
    -n csi-mixed-mode-fix-verification `
    get pod availability-then-durability `
    -o jsonpath='{.spec.affinity.nodeAffinity}'
```

Output:

```json
{
  "requiredDuringSchedulingIgnoredDuringExecution": {
    "nodeSelectorTerms": [
      {
        "matchExpressions": [
          {
            "key": "kubernetes.io/hostname",
            "operator": "In",
            "values": [
              "aks-nodepool1-74017372-vmss000000"
            ]
          }
        ]
      }
    ]
  }
}
```

Both orders now produce the same required node set:

```text
{aks-nodepool1-74017372-vmss000000}
```

The availability node `node-b` is absent.

### Verification Assertion

The following checks were applied to both mutation results:

```powershell
foreach ($json in @($dv, $vd)) {
    if ($json -notmatch 'requiredDuringSchedulingIgnoredDuringExecution') {
        throw 'Expected required affinity'
    }
    if ($json -match 'preferredDuringSchedulingIgnoredDuringExecution') {
        throw 'Unexpected preferred affinity'
    }
    if ($json -notmatch 'aks-nodepool1-74017372-vmss000000') {
        throw 'Expected durability node A'
    }
    if ($json -match 'node-b') {
        throw 'Availability node B must not be permitted'
    }
}
```

Result:

```text
PASS: both volume orders require A only; B is excluded.
```

Webhook logs also showed that both volume modes were read but only A was sent
to required-affinity patching:

```text
pod="durability-then-availability"
pvName="csi-mixed-fix-durability-pv"
failoverMode="durability"
pvName="csi-mixed-fix-availability-pv"
failoverMode="availability"
nodes=["aks-nodepool1-74017372-vmss000000"]
failoverMode="durability"

pod="availability-then-durability"
pvName="csi-mixed-fix-availability-pv"
failoverMode="availability"
pvName="csi-mixed-fix-durability-pv"
failoverMode="durability"
nodes=["aks-nodepool1-74017372-vmss000000"]
failoverMode="durability"
```

This confirms that RAVEN-003 is resolved and volume declaration order no longer
changes placement behavior.

### Fix Verification Cleanup

The release was restored to revision `7`, and all verification resources and
the temporary image were removed:

```powershell
helm rollback local-csi-driver 7 `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --wait `
    --timeout 5m

kubectl --context hrk-aks-nvme `
    delete namespace csi-mixed-mode-fix-verification `
    --ignore-not-found `
    --wait=true

kubectl --context hrk-aks-nvme `
    delete persistentvolume `
    csi-mixed-fix-durability-pv `
    csi-mixed-fix-availability-pv `
    --ignore-not-found

kubectl --context hrk-aks-nvme `
    delete storageclass `
    csi-mixed-fix-durability `
    csi-mixed-fix-availability `
    --ignore-not-found

az acr repository delete `
    --name hrkashyapacr `
    --image acstor/local-csi-manager:mixed-mode-fix-20261012 `
    --yes

docker image rm `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:mixed-mode-fix-20261012
```

Cleanup output:

```text
Rollback was a success!
namespace "csi-mixed-mode-fix-verification" deleted
persistentvolume "csi-mixed-fix-durability-pv" deleted
persistentvolume "csi-mixed-fix-availability-pv" deleted
storageclass.storage.k8s.io "csi-mixed-fix-durability" deleted
storageclass.storage.k8s.io "csi-mixed-fix-availability" deleted
Deleted: sha256:16d817aa0e0b639bdbed3a46181a6543564ae59ed118280009f2f58f9ddf4ded
```

The original values were restored with the hyperconverged webhook disabled.

## Original Reproduction Cleanup

The release was restored to revision 5:

```powershell
helm rollback local-csi-driver 5 `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --wait `
    --timeout 5m
```

All reproduction resources and the temporary image were removed:

```powershell
kubectl --context hrk-aks-nvme `
    delete namespace csi-mixed-mode-repro `
    --ignore-not-found `
    --wait=true

kubectl --context hrk-aks-nvme `
    delete persistentvolume `
    csi-mixed-mode-durability-pv `
    csi-mixed-mode-availability-pv `
    --ignore-not-found

kubectl --context hrk-aks-nvme `
    delete storageclass `
    csi-mixed-mode-durability `
    csi-mixed-mode-availability `
    --ignore-not-found

az acr repository delete `
    --name hrkashyapacr `
    --image acstor/local-csi-manager:mixed-mode-repro-20261009 `
    --yes

docker image rm `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:mixed-mode-repro-20261009
```

Cleanup output:

```text
Rollback was a success!
namespace "csi-mixed-mode-repro" deleted
persistentvolume "csi-mixed-mode-durability-pv" deleted
persistentvolume "csi-mixed-mode-availability-pv" deleted
storageclass.storage.k8s.io "csi-mixed-mode-durability" deleted
storageclass.storage.k8s.io "csi-mixed-mode-availability" deleted
Deleted: sha256:a94b59d86cd077246e9b3e1f82c3fe508742ed366a5d96558b97074843c97c56
```

The final verification confirmed:

- The original Helm values were restored.
- The hyperconverged webhook was disabled.
- The namespace, PVs, and StorageClasses were absent.
- The temporary ACR tag and local image were deleted.
