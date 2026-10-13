# Durability Node Affinity OR Bug Reproduction

This document records the commands used on the `hrk-aks-nvme` Kubernetes
context to reproduce the durability node-affinity bug in the hyperconverged
webhook.

**Status:** Resolved and verified on `hrk-aks-nvme` on October 9, 2026.

The initial reproduction used the original source. A second cluster test used
the fixed `patchPodWithNodeAffinity` implementation. All test resources,
temporary container images, and temporary Helm configuration were removed
afterward.

## Bug Summary

When a Pod already has required node affinity, the webhook appends the storage
node requirement as a new `NodeSelectorTerm`.

Kubernetes evaluates separate `NodeSelectorTerms` using OR. The resulting rule
is therefore:

```text
Pod author's rule OR storage node rule
```

The required storage node can be bypassed whenever the Pod author's rule
matches another node.

## Environment Discovery

The following commands confirmed the target cluster and its initial state:

```powershell
kubectl config get-contexts

kubectl --context hrk-aks-nvme get nodes -o wide
kubectl --context hrk-aks-nvme get storageclass
kubectl --context hrk-aks-nvme get pods -A -o wide |
    Select-String 'local-csi|acstor|webhook'
kubectl --context hrk-aks-nvme get mutatingwebhookconfigurations
```

Relevant output:

```text
NAME                                STATUS   VERSION
aks-nodepool1-74017372-vmss000000   Ready    v1.35.7

NAME    PROVISIONER
local   localdisk.csi.acstor.io

NAME                                        WEBHOOKS
aks-node-mutating-webhook                   1
aks-webhook-admission-controller            1
gatekeeper-mutating-webhook-configuration   1
```

The local CSI Helm release was present, but the driver DaemonSet selected a
label that the node did not have. The hyperconverged webhook was disabled:

```powershell
kubectl --context hrk-aks-nvme get daemonsets,deployments -A
kubectl --context hrk-aks-nvme get csidrivers
kubectl --context hrk-aks-nvme get pv,pvc -A
helm list --kube-context hrk-aks-nvme -A
helm get values local-csi-driver `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --all
kubectl --context hrk-aks-nvme get nodes --show-labels
```

Relevant output:

```text
daemonset.apps/csi-local-node   0   0   0   0   0

localdisk.csi.acstor.io   false   true   true

NAME               NAMESPACE     STATUS     CHART
local-csi-driver   kube-system   deployed   local-csi-driver-0.0.1-latest

daemonset:
  nodeSelector:
    local-csi-nvme-poc: "true"

webhook:
  enforceEphemeral:
    enabled: false
  hyperconverged:
    enabled: false
```

The sole node and disabled webhook meant an actual wrong-node scheduling event
could not be demonstrated. The bug was instead verified from the Pod mutation
produced by the real webhook running in the cluster.

## Build Prerequisite Check

```powershell
docker version --format 'Client={{.Client.Version}} Server={{.Server.Version}}'
az account show --query '{name:name,state:state}' -o table
az acr show `
    --name hrkashyapacr `
    --query '{name:name,loginServer:loginServer}' `
    -o table
```

Relevant output:

```text
Client=29.7.2 Server=29.7.2

Name          LoginServer
------------  -----------------------
hrkashyapacr  hrkashyapacr.azurecr.io
```

## Manager Image Build

The first build used the working directory directly:

```powershell
az acr login --name hrkashyapacr

docker build `
    --pull `
    --build-arg GIT_COMMIT=bug-repro `
    --build-arg BUILD_ID=bug-repro-20261008 `
    -t hrkashyapacr.azurecr.io/acstor/local-csi-manager:bug-repro-20261008 `
    -f Dockerfile.manager `
    .
```

It failed because the Windows checkout had CRLF line endings in
`hack/generate-notice.sh`:

```text
/usr/bin/env: 'bash\r': No such file or directory
Dockerfile.manager:63
ERROR: failed to build
```

A `git archive` build context was also attempted, but it retained the committed
line endings and failed at the same step:

```powershell
$ctx = Join-Path $env:TEMP 'local-csi-bug-repro-context'
$archive = Join-Path $env:TEMP 'local-csi-bug-repro-context.tar'

New-Item -ItemType Directory -Path $ctx | Out-Null
git archive --format=tar HEAD -o $archive
tar -xf $archive -C $ctx

docker build `
    --pull `
    --build-arg GIT_COMMIT=bug-repro `
    --build-arg BUILD_ID=bug-repro-20261008 `
    -t hrkashyapacr.azurecr.io/acstor/local-csi-manager:bug-repro-20261008 `
    -f (Join-Path $ctx 'Dockerfile.manager') `
    $ctx
```

The successful build normalized only the temporary build-context copy of the
script. It did not modify the repository:

```powershell
$ctx = Join-Path $env:TEMP 'local-csi-bug-repro-context'
$archive = Join-Path $env:TEMP 'local-csi-bug-repro-context.tar'

if (Test-Path $ctx) {
    Remove-Item -Recurse -Force $ctx
}
if (Test-Path $archive) {
    Remove-Item -Force $archive
}

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
    --build-arg GIT_COMMIT=bug-repro `
    --build-arg BUILD_ID=bug-repro-20261008 `
    -t hrkashyapacr.azurecr.io/acstor/local-csi-manager:bug-repro-20261008 `
    -f (Join-Path $ctx 'Dockerfile.manager') `
    $ctx

docker push `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:bug-repro-20261008

Remove-Item -Recurse -Force $ctx
Remove-Item -Force $archive
```

Successful build and push output:

```text
naming to hrkashyapacr.azurecr.io/acstor/local-csi-manager:
bug-repro-20261008

The push refers to repository
[hrkashyapacr.azurecr.io/acstor/local-csi-manager]

bug-repro-20261008:
digest: sha256:c6f5f065561b645054790fa5f65c8e07ea977bcf8f82002ecae25b9b2dba8f3d
```

## Enable the Test Webhook

The existing Helm values were reused. Only the hyperconverged webhook, manager
image, and manager replica count were overridden:

```powershell
helm upgrade local-csi-driver `
    .\charts\latest `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --reuse-values `
    --set webhook.hyperconverged.enabled=true `
    --set `
image.manager.repository=hrkashyapacr.azurecr.io/acstor/local-csi-manager `
    --set image.manager.tag=bug-repro-20261008 `
    --set manager.deployment.replicas=1 `
    --atomic `
    --wait `
    --timeout 5m

kubectl --context hrk-aks-nvme `
    -n kube-system `
    get deployment,pods,service |
    Select-String 'csi-local-manager|csi-local-webhook'

kubectl --context hrk-aks-nvme `
    get mutatingwebhookconfiguration `
    csi-local-hyperconverged-webhook

kubectl --context hrk-aks-nvme `
    -n kube-system `
    logs deployment/csi-local-manager `
    --tail=100
```

Relevant output:

```text
Release "local-csi-driver" has been upgraded.
STATUS: deployed
REVISION: 2

deployment.apps/csi-local-manager   1/1
pod/csi-local-manager-5c86c4744-fjjl4   1/1   Running
service/csi-local-webhook-service   ClusterIP

NAME                               WEBHOOKS
csi-local-hyperconverged-webhook   1

"registering hyperconverged webhook"
"Registering webhook" path="/mutate-pod"
"hyperconverged webhook registered"
"Serving webhook server" port=9443
```

## Test Resources

The test Pod used a nonexistent scheduler. This prevented it from running or
mounting the synthetic volume. Only admission-time mutation was tested.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: csi-affinity-bug-repro
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-affinity-bug-repro
provisioner: localdisk.csi.acstor.io
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-affinity-bug-repro-pv
  annotations:
    localdisk.csi.acstor.io/selected-node: aks-nodepool1-74017372-vmss000000
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: csi-affinity-bug-repro
  csi:
    driver: localdisk.csi.acstor.io
    volumeHandle: csi-affinity-bug-repro-volume
    volumeAttributes:
      localdisk.csi.acstor.io/failover-mode: durability
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: csi-affinity-bug-repro
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-affinity-bug-repro
  volumeName: csi-affinity-bug-repro-pv
---
apiVersion: v1
kind: Pod
metadata:
  name: affinity-bug-repro
  namespace: csi-affinity-bug-repro
spec:
  schedulerName: bug-repro-no-scheduler
  containers:
  - name: pause
    image: mcr.microsoft.com/oss/kubernetes/pause:3.6
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: bug-repro/author-rule
            operator: In
            values:
            - allowed
```

Save the manifest as `affinity-bug-repro.yaml`, then run:

```powershell
kubectl --context hrk-aks-nvme `
    apply -f .\affinity-bug-repro.yaml

kubectl --context hrk-aks-nvme `
    -n csi-affinity-bug-repro `
    get pvc,pod

kubectl --context hrk-aks-nvme `
    -n csi-affinity-bug-repro `
    get pod affinity-bug-repro `
    -o jsonpath='{.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution}'

kubectl --context hrk-aks-nvme `
    -n kube-system `
    logs deployment/csi-local-manager `
    --since=2m
```

## Initial False Start

The first manifest used the standard Kubernetes selected-node annotation:

```yaml
volume.kubernetes.io/selected-node:
  aks-nodepool1-74017372-vmss000000
```

The driver expects its own annotation:

```yaml
localdisk.csi.acstor.io/selected-node:
  aks-nodepool1-74017372-vmss000000
```

The first attempt therefore did not mutate the Pod. Its log correctly reported:

```text
"found failover mode in PV volume attributes"
pvName="csi-affinity-bug-repro-pv"
failoverMode="durability"

"no node name found"
err="pv is not assigned to a node"
```

The annotation was corrected and the Pod was recreated:

```powershell
kubectl --context hrk-aks-nvme `
    delete pod affinity-bug-repro `
    -n csi-affinity-bug-repro `
    --wait=true

kubectl --context hrk-aks-nvme `
    apply -f .\affinity-bug-repro.yaml
```

## Confirmed Bug Output

The mutated Pod contained two separate `NodeSelectorTerms`:

```json
{
  "nodeSelectorTerms": [
    {
      "matchExpressions": [
        {
          "key": "bug-repro/author-rule",
          "operator": "In",
          "values": [
            "allowed"
          ]
        }
      ]
    },
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
```

The webhook logs confirmed that the durability branch generated the patch:

```text
"pod handling request"
request="affinity-bug-repro"
operation="CREATE"

"starting to inject storage affinity for pod"
pod="affinity-bug-repro"

"getting nodes for volume"
volume name="csi-affinity-bug-repro-pv"

"found failover mode in PV volume attributes"
pvName="csi-affinity-bug-repro-pv"
failoverMode="durability"

"list of nodes available for pod"
nodes=["aks-nodepool1-74017372-vmss000000"]
failoverMode="durability"

"applying required node affinity for durability mode"
pod="affinity-bug-repro"

"patching pod with node affinity"
pod="affinity-bug-repro"
failoverMode="durability"
```

Kubernetes evaluates the two terms as:

```text
bug-repro/author-rule = allowed
OR
kubernetes.io/hostname = aks-nodepool1-74017372-vmss000000
```

This confirms the bug. The storage hostname is not mandatory when another term
matches.

The correct structure would contain both expressions inside every existing
term:

```json
{
  "nodeSelectorTerms": [
    {
      "matchExpressions": [
        {
          "key": "bug-repro/author-rule",
          "operator": "In",
          "values": [
            "allowed"
          ]
        },
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
```

Expressions inside one term are evaluated using AND.

## Fix Verification

The fix was built into a temporary manager image:

```powershell
$ctx = Join-Path $env:TEMP 'local-csi-affinity-fix-context'
$archive = Join-Path $env:TEMP 'local-csi-affinity-fix-context.tar'

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
    --build-arg GIT_COMMIT=affinity-fix `
    --build-arg BUILD_ID=affinity-fix-20261009 `
    -t hrkashyapacr.azurecr.io/acstor/local-csi-manager:affinity-fix-20261009 `
    -f (Join-Path $ctx 'Dockerfile.manager') `
    $ctx

docker push `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:affinity-fix-20261009
```

Build and push output:

```text
affinity-fix-20261009:
digest: sha256:e44159294d0dd7b5855e9bf70744cb3709a99f6f2eb548257393e9c8da38574d
```

The existing Helm revision was recorded before enabling the fixed webhook:

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

helm upgrade local-csi-driver `
    .\charts\latest `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --reuse-values `
    --set webhook.hyperconverged.enabled=true `
    --set `
image.manager.repository=hrkashyapacr.azurecr.io/acstor/local-csi-manager `
    --set image.manager.tag=affinity-fix-20261009 `
    --set manager.deployment.replicas=1 `
    --rollback-on-failure `
    --wait `
    --timeout 5m
```

Relevant deployment output:

```text
Recorded rollback revision: 3
STATUS: deployed
REVISION: 4

deployment.apps/csi-local-manager   1/1
pod/csi-local-manager-7bc584c68b-fjhlm   1/1   Running
service/csi-local-webhook-service   ClusterIP

csi-local-hyperconverged-webhook   1
```

### Conflicting Affinity Test

The verification Pod supplied this existing required affinity:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-b
```

The synthetic durability PV was assigned to the real cluster node:

```yaml
metadata:
  annotations:
    localdisk.csi.acstor.io/selected-node:
      aks-nodepool1-74017372-vmss000000
spec:
  csi:
    volumeAttributes:
      localdisk.csi.acstor.io/failover-mode: durability
```

The Pod was created with the default Kubernetes scheduler. The mutated selector
was retrieved with:

```powershell
$affinityJson = kubectl `
    --context hrk-aks-nvme `
    -n csi-affinity-fix-verification `
    get pod affinity-fix-verification `
    -o jsonpath='{.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution}'

Write-Host $affinityJson
```

The fixed webhook produced one term containing both expressions:

```json
{
  "nodeSelectorTerms": [
    {
      "matchExpressions": [
        {
          "key": "kubernetes.io/hostname",
          "operator": "In",
          "values": [
            "node-b"
          ]
        },
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
```

Kubernetes evaluates expressions in one term using AND:

```text
hostname = node-b
AND
hostname = aks-nodepool1-74017372-vmss000000
```

No node can satisfy both hostname requirements. The real scheduler left the Pod
pending and emitted:

```text
NAME                        READY   STATUS    NODE
affinity-fix-verification   0/1     Pending   <none>

Warning  FailedScheduling
0/1 nodes are available:
1 node(s) didn't match Pod's node affinity/selector.
```

The successful verification command checked both the mutation shape and the
scheduler event:

```powershell
$selector = $affinityJson | ConvertFrom-Json
$terms = @($selector.nodeSelectorTerms)
$expressions = @($terms[0].matchExpressions)

$events = kubectl `
    --context hrk-aks-nvme `
    -n csi-affinity-fix-verification `
    get events `
    --field-selector involvedObject.name=affinity-fix-verification `
    --sort-by='.lastTimestamp'

$eventText = $events -join [Environment]::NewLine

if ($terms.Count -ne 1 -or $expressions.Count -ne 2) {
    throw 'Expected one term with two expressions'
}
if ($eventText -notmatch 'node affinity/selector') {
    throw 'Expected scheduler rejection event'
}
```

Result:

```text
PASS: fixed webhook ANDs both hostname requirements and the scheduler rejects
the conflict.
```

This confirms that the reported issue is resolved by the current fix.

### Fix Verification Cleanup

The release was restored to revision 3, and all verification resources and the
temporary image were deleted:

```powershell
helm rollback local-csi-driver 3 `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --wait `
    --timeout 5m

kubectl --context hrk-aks-nvme `
    delete namespace csi-affinity-fix-verification `
    --ignore-not-found `
    --wait=true

kubectl --context hrk-aks-nvme `
    delete persistentvolume csi-affinity-fix-verification-pv `
    --ignore-not-found

kubectl --context hrk-aks-nvme `
    delete storageclass csi-affinity-fix-verification `
    --ignore-not-found

az acr repository delete `
    --name hrkashyapacr `
    --image acstor/local-csi-manager:affinity-fix-20261009 `
    --yes

docker image rm `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:affinity-fix-20261009
```

Cleanup output:

```text
Rollback was a success!
namespace "csi-affinity-fix-verification" deleted
persistentvolume "csi-affinity-fix-verification-pv" deleted
storageclass.storage.k8s.io "csi-affinity-fix-verification" deleted
Deleted: sha256:e44159294d0dd7b5855e9bf70744cb3709a99f6f2eb548257393e9c8da38574d
```

The restored values again showed:

```yaml
webhook:
  enforceEphemeral:
    enabled: false
  hyperconverged:
    enabled: false
```

## Original Reproduction Cleanup

The Helm release was rolled back to revision 1. All test resources and the
temporary image were then removed:

```powershell
helm rollback local-csi-driver 1 `
    --kube-context hrk-aks-nvme `
    -n kube-system `
    --wait `
    --timeout 5m

kubectl --context hrk-aks-nvme `
    delete namespace csi-affinity-bug-repro `
    --ignore-not-found `
    --wait=true

kubectl --context hrk-aks-nvme `
    delete persistentvolume csi-affinity-bug-repro-pv `
    --ignore-not-found

kubectl --context hrk-aks-nvme `
    delete storageclass csi-affinity-bug-repro `
    --ignore-not-found

az acr repository delete `
    --name hrkashyapacr `
    --image acstor/local-csi-manager:bug-repro-20261008 `
    --yes

docker image rm `
    hrkashyapacr.azurecr.io/acstor/local-csi-manager:bug-repro-20261008
```

Cleanup output:

```text
Rollback was a success!
namespace "csi-affinity-bug-repro" deleted
persistentvolume "csi-affinity-bug-repro-pv" deleted
storageclass.storage.k8s.io "csi-affinity-bug-repro" deleted

Deleted:
sha256:c6f5f065561b645054790fa5f65c8e07ea977bcf8f82002ecae25b9b2dba8f3d
```

The following verification commands produced no output for the deleted
resources:

```powershell
kubectl --context hrk-aks-nvme `
    get namespace csi-affinity-bug-repro `
    --ignore-not-found

kubectl --context hrk-aks-nvme `
    get pv csi-affinity-bug-repro-pv `
    --ignore-not-found

kubectl --context hrk-aks-nvme `
    get storageclass csi-affinity-bug-repro `
    --ignore-not-found

kubectl --context hrk-aks-nvme `
    get mutatingwebhookconfiguration `
    csi-local-hyperconverged-webhook `
    --ignore-not-found
```

The restored release values were:

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
