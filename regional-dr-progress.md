# Regional-DR Setup Progress — ACM 2.15 + ODF 4.21 + Submariner on AWS

A running log of the issues we hit while bringing up Regional-DR for the
`pacman01` workload across two AWS-hosted OpenShift clusters managed by an ACM
hub, and how each was resolved. Read in order — most issues compounded on
earlier ones.

---

## Environment

| Component | Identifier / Version |
|---|---|
| Hub cluster | `cluster-gr4bg` — OpenShift 4.21.10, ACM 2.15.1 (MultiClusterHub `multiclusterhub`), 1 controller / 3 workers |
| Primary managed cluster | `ocpaws1` — OpenShift 4.21.10, AWS `us-east-1`, ODF 4.21.2 |
| Secondary managed cluster | `ocpaws2` — OpenShift 4.21.10, AWS `eu-central-1`, ODF 4.21.2 |
| Submariner | v0.22.1 (via `advanced-cluster-management` addon) |
| Broker namespace | `aws-regional-dr-broker` on the hub |
| ManagedClusterSet | `aws-regional-dr` (binds ocpaws1 + ocpaws2) |
| Test workload | `pacman01` — Pac-Man front end + MongoDB back end |
| Source repo | `https://github.com/anatsheh84/k8s-pacman-app.git` |

---

## 1. Submariner gateway nodes never registered (16 GB root volume)

### Symptom

After applying `SubmarinerConfig` and `ManagedClusterAddOn` on both managed
clusters, the hub-side `ManagedClusterAddOn submariner` reported
`SubmarinerConnectionDegraded=True / ConnectionsNotEstablished` and
`SubmarinerAgentDegraded=True / NoScheduledGateways`. On both managed
clusters the auto-created `submariner-gw-*` MachineSet showed
`desired=1, ready=0`; the EC2 instance was running but stuck at `Provisioned`,
no node had ever joined the cluster, and there were no pending CSRs.

### Root cause

The ACM submariner-addon clones the worker MachineSet to create the gateway
MachineSet but **strips `spec.template.spec.providerSpec.value.blockDevices`**
when doing so. With `blockDevices` empty, AWS uses the AMI default —
**16 GB gp2** — for the root EBS volume. RHCOS + the MCO `rpm-ostree` pivot
cannot complete in 16 GB. The EC2 console output of the gateway instance
showed:

```
Ignition: ran on 2026/04/26 22:27:36 UTC (this boot)
Ignition: user-provided config was applied
[  900.752] systemd-journald: Failed to save stream data ...
            No space left on device
Failed to start rpm-ostree System Management Daemon
```

Because the pivot never completed, kubelet never started, no CSR was filed,
the node never registered, no `submariner.io/gateway=true` label was applied,
the gateway pod was never scheduled, and the IPsec tunnel could not come up.

The `SubmarinerConfig` CRD has no field for root-volume size — only
`instanceType` and `gateways` are exposed under `spec.gatewayConfig.aws`.
Switching instance types alone does not fix this; the root volume size is
governed by `blockDevices` regardless of the chosen instance type.

### Resolution

Patch the gateway MachineSet to add a real root volume:

```yaml
spec:
  template:
    spec:
      providerSpec:
        value:
          blockDevices:
            - ebs:
                encrypted: true
                volumeSize: 100
                volumeType: gp3
```

Then delete the stuck Machine so the MachineSet templates a new one **from
the patched spec** — once a Machine exists with `blockDevices` baked into
its `spec.providerSpec`, AWS provisions a 100 GB gp3 root regardless of any
subsequent MachineSet revert (Machine spec is immutable for this field).

### Race against the addon's reconciler

The submariner-addon controller actively reconciles the gateway MachineSet
and reverts manual `blockDevices` patches within seconds. A one-shot
`oc patch` often loses the race: in the first attempt on `ocpaws1`, the
controller wiped the patch *before* the new Machine was templated, so the
replacement Machine still had a 16 GB root.

We solved this with a small background watcher per cluster
(`/tmp/sub-debug/race-patch.sh`) that polls every 2 s and re-applies the
patch any time it finds the MachineSet's `blockDevices` empty, plus
deletes any Machine that was templated without it, until a Machine exists
with `volumeSize: 100` baked into its spec. Run the watcher *before*
re-applying the SubmarinerConfig, so the moment the addon creates the
MachineSet, the patch lands within ~3 s.

Within 5 minutes of a properly-templated gateway Machine, the node
registered, was labeled `submariner.io/gateway=true`, the gateway pod
scheduled, and the IPsec tunnel came up. Both addons reached
`Available=True / SubmarinerAgentDeployed / ConnectionsEstablished`.

### Lesson / prevention

The MachineSet template still gets reverted to empty `blockDevices` by the
addon — the live Machine on disk is fine, but if it's ever recreated (HA
event, addon re-template, manual scale), the disk problem returns. Worth an
upstream bug against `submariner-addon` to preserve `blockDevices` when
cloning the worker MachineSet on AWS, or to expose `volumeSize` /
`volumeType` on `SubmarinerConfig.spec.gatewayConfig.aws`.

---

## 2. Cross-cluster TLS trust for ramen-hub-operator → MCG S3

### Symptom

The Ramen DR hub operator stores DR metadata in MCG (NooBaa) S3 buckets on
each managed cluster, accessed via
`https://s3-openshift-storage.apps.<managed-cluster>...`. The managed
clusters' default ingress certs are issued by an in-cluster
`CN=ingress-operator@…` and are not in the hub's trust store, so the S3 client
TLS handshake from ramen-hub would fail with x509 errors and DRPolicy
validation would hang at `Validating`.

### Resolution

Built a combined `user-ca-bundle` ConfigMap from the two managed clusters'
`default-ingress-cert` ConfigMaps:

```sh
oc --kubeconfig=$KC1 get cm default-ingress-cert -n openshift-config-managed \
   -o jsonpath="{.data.ca-bundle\.crt}" > /tmp/dr-ssl/primary.crt
oc --kubeconfig=$KC2 get cm default-ingress-cert -n openshift-config-managed \
   -o jsonpath="{.data.ca-bundle\.crt}" > /tmp/dr-ssl/secondary.crt
```

Concatenated both into a single `ConfigMap user-ca-bundle` in
`openshift-config`, applied to **all three clusters** (hub + both managed),
then patched each cluster's `Proxy/cluster`:

```sh
oc patch proxy cluster --type=merge \
  --patch='{"spec":{"trustedCA":{"name":"user-ca-bundle"}}}'
```

OpenShift's cluster-network-operator merges `user-ca-bundle` into the
system trust store distributed to all pods. Verified: each cluster's
`user-ca-bundle` ConfigMap contains 4 cert blocks (leaf + self-signed CA for
each managed cluster), and `proxy/cluster.spec.trustedCA.name` is
`user-ca-bundle`.

The hub's own ingress cert (Let's Encrypt) was untouched — only its trust
*store* was extended.

### Then: restart ramen-hub-operator to pick up the new bundle

```sh
oc rollout restart deployment/ramen-hub-operator -n openshift-operators
```

This appeared to succeed (`successfully rolled out`) but the new pod
crash-looped. Investigation: the deployment is owned by CSV
`odr-hub-operator.v4.21.2-rhodf`. OLM saw the
`kubectl.kubernetes.io/restartedAt` annotation as drift, reverted it, and the
new ReplicaSet was scaled to 0. The fix for OLM-managed operators is:

```sh
oc delete pod -n openshift-operators -l name=ramen-hub-operator
```

Plain pod delete — the deployment controller spawns a fresh pod with the
*current* (operator-blessed) spec, which mounts the in-place-updated
`openshift-trusted-cabundle` ConfigMap. Verified the new pod has volume
`ramen-manager-trustedca-vol` referencing ConfigMap
`openshift-trusted-cabundle`.

### Lesson / prevention

For OLM-managed operators, never use `rollout restart` — OLM treats the
annotation as drift. Use `oc delete pod` and let the deployment controller
recreate.

---

## 3. Two default StorageClasses on managed clusters

### Symptom

`oc get sc` on both managed clusters showed both `gp3-csi` and
`ocs-storagecluster-ceph-rbd` annotated as default
(`storageclass.kubernetes.io/is-default-class: "true"`). Kubernetes does
not error on this — it picks one arbitrarily — which means a PVC created
without an explicit `storageClassName` could land on either, and EBS-backed
PVCs are not replicated by ODF Regional-DR.

### Resolution

On each managed cluster:

```sh
oc patch storageclass gp3-csi -p \
  '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Verified: each cluster now has exactly one default StorageClass,
`ocs-storagecluster-ceph-rbd`. No `ReclaimPolicy`, `VolumeBindingMode`, or
provisioner field was changed; existing PVCs are unaffected.

### Lesson / prevention

For DR-protected workloads, the default StorageClass on every managed
cluster must be the one mirrored by ODF (`ocs-storagecluster-ceph-rbd`). A
dual-default situation is a silent footgun: a PVC author who forgets
`storageClassName` has a 50/50 chance of landing on EBS and silently
falling outside DR coverage.

---

## 4. GitOps deployment of pacman01 — the placement chain

### Symptom

User created `ApplicationSet pacman01` (with associated
`Placement pacman01-placement`) targeting cluster set `default` with
predicate `pacman=true`. Application never deployed.

### Root cause

`Placement.spec.clusterSets: [default]`, but `ocpaws1` (the only cluster
labeled `pacman=true`) lives in cluster set `aws-regional-dr`. Result: zero
matched clusters → ApplicationSet generated zero children → no Argo
Application → nothing deployed.

### Resolution

Switched the Placement to the right cluster set and added the matching
binding in `openshift-gitops`:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
  name: aws-regional-dr
  namespace: openshift-gitops
spec:
  clusterSet: aws-regional-dr
---
spec.clusterSets: [aws-regional-dr]   # on pacman01-placement
```

After this, `Placement` reported `SUCCEEDED=True / SELECTEDCLUSTERS=1` with
decision `ocpaws1`.

### But still no Argo Application — second issue

Controller log:

```
matched cluster in ArgoCD ... unmatched cluster in ArgoCD clusterName=ocpaws1
generated 0 applications
```

ACM placement matched ocpaws1, but **Argo CD on the hub had no Cluster
Secret for ocpaws1**. Argo refuses to template a destination it doesn't
know. The OpenShift-GitOps operator only auto-registered `local-cluster`.

### Resolution

Created a `GitOpsCluster` covering the `aws-regional-dr` set:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata: { name: aws-regional-dr-placement, namespace: openshift-gitops }
spec:    { clusterSets: [aws-regional-dr] }
---
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata: { name: aws-regional-dr, namespace: openshift-gitops }
spec:
  argoServer: { argoNamespace: openshift-gitops }
  placementRef:
    apiVersion: cluster.open-cluster-management.io/v1beta1
    kind: Placement
    name: aws-regional-dr-placement
```

The GitOpsCluster controller then minted
`ocpaws1-application-manager-cluster-secret` and
`ocpaws2-application-manager-cluster-secret` — Argo accepted the destinations
and the Application was generated.

### Lesson / prevention

For ACM-driven GitOps deployments, three pieces must align:

1. The `Placement.spec.clusterSets` must match a `ManagedClusterSetBinding`
   in the same namespace as the Placement.
2. A `GitOpsCluster` must cover the same cluster set so the chosen managed
   clusters become Argo destinations.
3. The selected ManagedClusters must actually carry the
   predicate-matching labels.

A Placement returning `SELECTEDCLUSTERS > 0` is necessary but not
sufficient — you also need an Argo cluster secret for the destination.

---

## 5. Sync failed: namespace mismatch (`pacman01` vs `pacman-app`)

### Symptom

`Application pacman01-ocpaws1` synced from Git but every resource was
`OutOfSync / Missing` with:

```
namespaces "pacman-app" not found. Retrying attempt #4
```

### Root cause

Git manifests have `metadata.namespace: pacman-app` hardcoded. The
ApplicationSet template specified `destination.namespace: pacman01` with
`syncOptions: CreateNamespace=true`, so Argo created `pacman01` (which sat
empty) but then tried to apply manifests targeting `pacman-app` — Argo's
`CreateNamespace=true` only creates the *destination* namespace, not
arbitrary namespaces referenced inside manifests.

### Resolution

Patched the ApplicationSet so destination namespace matches the
hardcoded namespace in the Git manifests:

```sh
oc patch applicationset pacman01 -n openshift-gitops --type=merge \
  -p '{"spec":{"template":{"spec":{"destination":{"namespace":"pacman-app"}}}}}'
```

### Lesson / prevention

ApplicationSet's `destination.namespace` must match the namespaces declared
inside the Git manifests, OR the manifests should drop the hardcoded
namespace so they inherit it from the Application destination. Mixing the
two creates orphan namespaces.

---

## 6. Mongo `ImagePullBackOff` — Bitnami registry move

### Symptom

`mongo-686db6785-mlwgg` pod stuck in `ErrImagePull`:

```
Failed to pull image "bitnami/mongodb:5.0.10": ...
manifest unknown
reading manifest 5.0.10 in docker.io/bitnami/mongodb: manifest unknown
```

### Root cause

On 2025-08-28, Bitnami / Broadcom **removed almost all tagged images from
`docker.io/bitnami/*`**. The free read-only archive of the old tags now
lives at `docker.io/bitnamilegacy/*`. Pulls of the old paths return
"manifest unknown" because Docker Hub no longer holds them.

### Resolution

Updated the Git manifests to point at the legacy archive:

```yaml
image: bitnamilegacy/mongodb:5.0.10
```

One-line change, identical image build, no other adjustments required.

### Lesson / prevention

For workloads still pinned to `bitnami/*` images, expect to either
re-point to `bitnamilegacy/*` (free, read-only) or move to Bitnami Secure
Images (paid). Mirror critical images to your own registry to insulate
against vendor policy changes.

---

## 7. Mongo data was not on the PV (Bitnami `dbPath` override)

### Symptom

After the first failover (ocpaws2 → ocpaws1) appeared to succeed —
`Application Synced/Healthy`, mongo pod Running, high score visible in
`db.highscore` — closer inspection found that **the database was not on the
mirrored Ceph RBD volume at all**.

### Root cause

On the destination cluster:

| Check | Result |
|---|---|
| PVC `mongo-storage` (RBD `/dev/rbd0`, 8 Gi ext4) | mounted at `/data/db` |
| `/data/db` contents | empty — only `lost+found`, 20 KB used |
| mongod's actual `storage.dbPath` | **`/bitnami/mongodb/data/db`** |
| `/bitnami/mongodb/data/db` filesystem | **`overlay`** (container's writable layer) |
| `/bitnami/mongodb/data/db` size | 301 MB of WiredTiger files |

The Bitnami MongoDB image overrides the upstream MongoDB `dbPath` to
`/bitnami/mongodb/data/db` (Bitnami's convention). Vanilla `mongo:*` images
use `/data/db`, but Bitnami's wrapper redirects `dbPath`, so the PVC mount
at `/data/db` sat at a path mongod never touched. All mongo writes went to
the container overlay FS, which is local and ephemeral. The high score
seen on ocpaws2 after "failover" was a fresh write to the destination's
container disk, not data carried by Ceph RBD mirroring.

### Resolution

Mount the PVC at `/bitnami/mongodb` (preferred for this image):

```yaml
volumeMounts:
  - name: mongo-db
    mountPath: /bitnami/mongodb
```

This captures `data/db` plus Bitnami's other state. Verified after the fix:

```
$ df -hT /bitnami/mongodb/data/db /
/dev/rbd0   ext4     7.8G  301M  7.5G   4%   /bitnami/mongodb
overlay     overlay  100G   23G   77G  23%   /
```

### Lesson / prevention

For any image that wraps an upstream daemon with a custom data-path
convention (Bitnami, Confluent, Crunchy, etc.), **verify** the running
process's actual `dbPath`/`data-dir`/equivalent by querying the daemon
itself or reading the running config. Don't assume the upstream default
applies. A "successful" failover that only redeploys the app while data
silently lives on the container overlay is the classic false-positive.

---

## 8. Failover EPERM — UID drift across clusters

### Symptom

After the manifests were fixed to use `/bitnami/mongodb`, a failover from
ocpaws2 → ocpaws1 left the new mongo pod crash-looping with WiredTiger
fatal:

```
WiredTiger error ... /bitnami/mongodb/data/db/WiredTiger.wt:
handle-open: open: Operation not permitted
Terminating. reason: 1: Operation not permitted
```

### Root cause

`Operation not permitted` is `EPERM`, not `EACCES`. The distinction matters.

OpenShift assigns each namespace its own UID range
(`openshift.io/sa.scc.uid-range`) **independently per cluster**. The
`pacman-app` namespace was allocated:

- `ocpaws2`: uid range starts at `1000830000`
- `ocpaws1`: uid range starts at `1000840000`

After the Ceph RBD mirror replicated the volume to ocpaws1, the on-disk
file ownership was preserved at uid `1000830000` (the original writer's
uid on ocpaws2). The new mongo pod on ocpaws1 ran under `1000840000`. With
fsGroup applied, the file *group* matched (kubelet's recursive chgrp), and
mode `0660` granted group rw. So plain `read()`/`write()` would have
worked.

But WiredTiger opens its data files with `O_NOATIME` for performance:

```c
open(filename, O_RDWR | O_NOATIME, …)
```

From `open(2)`:

> `O_NOATIME` — only effective when the **owner** of the file matches the
> EUID of the caller, or the caller has the **CAP_FOWNER** capability.

The `restricted-v2` SCC strips all capabilities and the file owner uid did
not match the caller — so `open()` returned `EPERM`. Group access via
`fsGroup` does not satisfy `O_NOATIME`'s ownership requirement.

### Initial fix considered: per-pod runAsUser + custom SCC

We sketched a Deployment that pinned `runAsUser: 1001` (Bitnami's expected
mongo user), added an init container chowning `/bitnami/mongodb` to
`1001:1001`, and created a `ServiceAccount mongo` bound to the
`system:openshift:scc:anyuid` ClusterRole. This works but requires editing
every chart/manifest, every workload's SA, and an extra SCC binding to
maintain. It also fights operators that re-template their pods.

### Final fix: pin namespace SCC annotations

Better approach for namespaces we control: pin the three SCC range
annotations on the `Namespace` object itself, identical on both managed
clusters, so the default `restricted-v2` SCC assigns the same UID, fsGroup,
and SELinux MCS level on both sides. The Deployment stays vanilla — no
`runAsUser`, no custom SCC, no chown init container.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: pacman-app
  annotations:
    openshift.io/sa.scc.uid-range: "1000840000/10000"
    openshift.io/sa.scc.supplemental-groups: "1000840000/10000"
    openshift.io/sa.scc.mcs: "s0:c29,c14"
```

Procedure: pick one cluster's allocations as canonical, copy them into the
manifest, apply to both clusters via Argo. Existing pods/SAs on the
destination keep their old admission, but new pods admitted there after
the sync get the pinned values — matching the source's on-disk file
ownership/labels.

After a one-time chown of the existing PV contents (`chown -R 1000840000:1000840000`
via a privileged debug pod) — needed because the legacy data was written
under the old uid — failover/relocate became uid-stable across clusters.

### Lesson / prevention

UID drift is universal to ACM Regional-DR — not Mongo-specific. Different
workloads exhibit different failure modes:

- **MongoDB / WiredTiger** — immediate `EPERM` on `O_NOATIME` open. Loud.
- **PostgreSQL** — refuses to start: "data directory has wrong ownership".
- **MySQL / MariaDB** — usually starts but mixed-ownership accumulates.
- **Redis with persistence** — typically works (no `O_NOATIME`), until it
  doesn't.
- **Elasticsearch / Cassandra / Kafka** — start, then specific operations
  (compactions, segment merges) hit EPERM.
- **Generic apps that chmod/chown/utimes their own files** — EPERM.
- **Apps that write `0600` files** — survive failover only if the new uid
  matches; group access doesn't help.

For any DR-protected stateful workload, the namespace SCC pinning is the
default, and per-pod fixed UID + custom SCC is the fallback for namespaces
we don't control (operator-managed, third-party Helm charts).

---

## 9. Successful failover — high score data carried across

After all of the above:

- Argo Application `pacman01-ocpaws2 Synced / Healthy`
- High score inserted on ocpaws2.
- DRPC `Failover ocpaws2 → ocpaws1` issued.
- Failover declared `Available=True / FailedOver / Completed` in **~67 s**.
- Same PVC UID `pvc-de7cab3a-…` rebound on ocpaws1 to the same Ceph RBD
  image (mirrored).
- mongo pod on ocpaws1: `1/1 Running`, no UID errors.
- `db.highscore` on ocpaws1: contained the score set on ocpaws2 before the
  failover. **End-to-end DR data preservation confirmed.**

Failover end-to-end timeline (UTC):

| Time | Event | Δ from start |
|---|---|---|
| 18:27:42 | `Started failover to cluster ocpaws1` | 0 s |
| 18:39:10 (during earlier cycle) | VRG restored on target | +33 s |
| 18:28:26 | `Available=True / FailedOver / Completed` | **+67 s** |

---

## 10. Argo CD `prune=true` on DR-protected workloads is a footgun

### Symptom (during a Relocate)

ACM Placement controller transiently empties the PlacementDecision while
coordinating the Relocate. ApplicationSet sees zero matched clusters and
**deletes its child Argo Application** (which carries
`resources-finalizer.argocd.argoproj.io`). The cascade then deletes the
`Namespace`, which deletes the PVC, which — because
`ocs-storagecluster-ceph-rbd.reclaimPolicy: Delete` — destroys the
underlying Ceph RBD image.

Net result: source workload + namespace + PVC + RBD image all gone before
final-sync completed. The relocate then deadlocks at `RunningFinalSync`
because there's nothing left to mirror to the destination.

Failover (different code path) does **not** drain the Placement decision
and is unaffected.

### Required Argo Application syncPolicy for DR

| UI option | Field | Setting | Why |
|---|---|---|---|
| Delete resources that are no longer defined in source | `automated.prune` | **`false`** | Stops Argo from pruning when the Placement decision transiently empties during Relocate. |
| Allow applications to have empty resources | `automated.allowEmpty` | **`true`** | Lets the Application stay valid through the empty-decision window instead of erroring. |
| Automatically sync when cluster state changes | `automated.selfHeal` | **`true`** | selfHeal only re-applies missing resources from Git; it does not prune. |
| Automatically create namespace if it does not exist | `syncOptions: CreateNamespace=true` | **on** | Lets Argo create the namespace on the relocate target. |
| Prune propagation policy | `syncOptions: PrunePropagationPolicy=orphan` | **`orphan`** | If anything is ever deleted, `orphan` leaves managed resources behind instead of cascade-deleting. |

Equivalent YAML on the Application:

```yaml
spec:
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
      allowEmpty: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=orphan
```

### Required ApplicationSet setting

The Application sync policy above protects against pruning *during sync*.
The **ApplicationSet** can still delete the child Application when the
Placement empties; to prevent that cascade:

```yaml
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true
```

Documented in ODF's GitOps DR reference architecture; mandatory for any
DR-protected ApplicationSet.

### Lesson / prevention

Argo's `automated.prune: true` plus the default Application finalizer is
incompatible with the Placement-decision-drain behavior of ACM Regional-DR
during Relocate. Without these settings, every Relocate is a 50/50 chance
of destroying the source workload before the data plane can complete
mirror sync.

---

## 11. Azure cluster provisioning failed — wrong VM family / quota exhausted

### Symptom

A new managed cluster `ocpaz1` was created from the ACM hub UI on Azure
(`eastus`). After ~71 minutes the Hive `ClusterDeployment` flipped to
`ProvisionStopped` with reason `InstallAttemptsLimitReached`. The provision
pod `ocpaz1-0-zckm4-provision-5jnvx` exited with status `Error` (exit
code 6 from `openshift-install`).

The hub's `ClusterDeployment` status told us the high-level cause:

```
ProvisionFailed   reason=NoWorkerNodesReady
                  msg="0 worker nodes have joined the cluster"
ProvisionStopped  reason=InstallAttemptsLimitReached
                  msg="Install attempts limit reached"
```

### Root cause (Azure side, confirmed via `az` CLI)

Three master VMs `ocpaz1-pxn8k-master-{0,1,2}` came up cleanly on
`Standard_D4s_v3` (one per AZ). All three worker VM creations failed at
09:33:16–09:33:19 UTC with this Azure error:

```
{
  "code": "OperationNotAllowed",
  "message": "Operation could not be completed as it results in exceeding
              approved standardDCSv2Family Cores quota.
              Location: eastus, Current Limit: 2, Current Usage: 0,
              Additional Required: 4, (Minimum) New Limit Required: 4."
}
```

Two layered problems:

1. **Wrong VM family**: the install config / MachinePool selected a worker
   `vmSize` from the **`Standard_DCsv2`** family — Azure's
   *Confidential Computing* (Intel SGX) family. This is the wrong default
   for general-purpose OpenShift workers; it's a specialty SKU for
   enclave-isolated workloads. Most subscriptions have near-zero quota
   for it by default.
2. **Quota: 2 cores limit**: the subscription's `standardDCSv2Family Cores`
   quota in eastus was capped at **2 cores** — not even enough for a
   single 4-core worker. Three worker creates ran in parallel, all hit the
   same denial, none could proceed. Masters were unaffected because they
   used a *different* family (`standardDSv3Family`), which had sufficient
   headroom.

Cascade: 0 workers Ready → ingress controller has nowhere to schedule its
router pods → routes (console, oauth, image-registry) can't be admitted →
`authentication`, `console`, `ingress`, `machine-api`, `monitoring`,
`image-registry` all stay Degraded → `openshift-install` 40-minute
"wait-for-cluster-init" timer expires → exit 6 → Hive marks
`ProvisionStopped`.

The 3 masters were still billing in eastus after the install gave up,
since Hive's deprovision is only triggered by deleting the
`ClusterDeployment`.

### Resolution

For Azure on this subscription, **use `Standard Dv3 Family vCPUs` or
`Standard Dv4 Family vCPUs`** as the worker SKU family. These are the
general-purpose families OpenShift expects, both have substantial default
quota in `eastus`, and they match what the masters already successfully
used.

Concretely, set the worker `vmSize` to one of:

| Family | Recommended SKU | vCPU / RAM | Notes |
|---|---|---|---|
| Dv3 | `Standard_D4s_v3` | 4 / 16 GB | Same family as the masters; tested and works on this subscription |
| Dv3 | `Standard_D8s_v3` | 8 / 32 GB | If workers will run heavier workloads |
| Dv4 | `Standard_D4s_v4` | 4 / 16 GB | Newer hardware, slightly better price/perf |
| Dv4 | `Standard_D8s_v4` | 8 / 32 GB | Larger workers on Dv4 |

Avoid (without an explicit Confidential Computing requirement and a quota
increase request approved):

- `Standard_DC*s_v2` — Confidential Computing (the family that failed)
- `Standard_DC*s_v3` — Confidential Computing v3
- Any non-`s` SKU lacking premium-disk support (RHCOS needs premium SSD)

In ACM hub UI, configure this on the `MachinePool` for workers, or in the
install-config YAML used to create the `ClusterDeployment`. For
declarative use, `MachinePool.spec.platform.azure.osDisk.diskSizeGB` and
`MachinePool.spec.platform.azure.type` are the relevant fields:

```yaml
apiVersion: hive.openshift.io/v1
kind: MachinePool
metadata:
  name: ocpaz1-worker
  namespace: ocpaz1
spec:
  clusterDeploymentRef:
    name: ocpaz1
  name: worker
  platform:
    azure:
      type: Standard_D4s_v3        # ← Dv3 family; Dv4 also fine: Standard_D4s_v4
      zones: [ "1", "2", "3" ]
      osDisk:
        diskSizeGB: 128
  replicas: 3
```

### Lesson / prevention

1. **Verify VM family before any large Azure install.** Run
   `az vm list-usage --location <region> -o table | grep -E "vCPU|Cores"`
   to confirm there's headroom in whichever family you intend to use, in
   the target region, on the target subscription. Workshop / sandbox /
   trial subscriptions often have 2–10 core caps on non-default families.
2. **Match worker family to master family by default.** If masters use
   `Standard_D4s_v3` (Dv3) and you have no specific reason to differ,
   keeping workers in the same family eliminates the "two separate
   quota gauges" failure mode.
3. **Don't pick `DCsv2` / `DCsv3` unless you actually need Confidential
   Computing** for an enclave-protected workload. These families have
   different licensing, lower availability across zones, and almost
   always require a manual quota increase request in workshop
   subscriptions.
4. **Read Hive's `ClusterDeployment` conditions early.** The
   `ProvisionFailed=True / NoWorkerNodesReady` condition appears within
   ~30 minutes of an install starting; you don't need to wait the full
   71 minutes. A daily check (or alert) on
   `oc get clusterdeployment -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name} {range .status.conditions[?(@.type=="ProvisionFailed")]}{.status} {.reason}{"\n"}{end}{end}'`
   surfaces in-progress install issues before the install-attempts limit
   is exhausted.
5. **Activity log first when an Azure install fails.** Running
   `az monitor activity-log list --resource-group <rg> --status Failed`
   against the install resource group (`<cluster>-<infraID>-rg`) returns
   the exact Azure-side denial message — quota, SKU availability,
   networking, RBAC, etc. Faster than trawling the install pod log.
6. **Dispose of leftover masters when an install fails.** If you don't
   plan to retry the same `ClusterDeployment`, delete it on the hub —
   that triggers Hive's deprovision and stops the Azure billing on the
   3 master VMs that did succeed.

---

## 12. README documentation (in the workload repo)

The workload repo at `https://github.com/anatsheh84/k8s-pacman-app` now
carries a "Deploying under ACM Regional-DR" section in `README.md`
documenting:

- `bitnamilegacy/mongodb:5.0.10` and the Bitnami registry move.
- `/bitnami/mongodb` mount path (not `/data/db`).
- Namespace SCC annotation pinning to prevent UID drift.
- Argo CD Application syncPolicy + ApplicationSet
  `preserveResourcesOnDeletion: true`.
- Storage-class default requirement
  (`ocs-storagecluster-ceph-rbd` as single default).
- End-to-end DR validation flow.

Two commits on `main`:

```
df068ca docs: document Regional-DR settings for ACM + Argo CD
cf2a62d Pin namespace SCC annotations for Regional-DR
49818e2 Change MongoDB mount path to /bitnami/mongodb (pre-existing)
7207f0d Update MongoDB image to bitnamilegacy version (pre-existing)
```

---

## Summary — issues solved

| # | Layer | Class | Resolution |
|---|---|---|---|
| 1 | AWS / MachineSet | submariner-addon strips `blockDevices`, 16 GB AMI default insufficient | Patch MachineSet to 100 GB gp3, race the addon's reconciler with a watcher |
| 2 | TLS trust | Managed clusters' ingress CAs not trusted by hub | `user-ca-bundle` ConfigMap on all 3 clusters + `proxy/cluster.spec.trustedCA` |
| 2b | Operator restart | OLM reverts `rollout restart` annotation | `oc delete pod` instead of `rollout restart` |
| 3 | StorageClass | Two SCs annotated default | Patch `gp3-csi` annotation to `false` |
| 4a | ACM Placement | Wrong cluster set in `spec.clusterSets` | Add `ManagedClusterSetBinding` + switch placement to `aws-regional-dr` |
| 4b | Argo registration | Hub had no Argo cluster secret for managed clusters | Create `GitOpsCluster aws-regional-dr` |
| 5 | Argo / manifests | Destination namespace ≠ namespace inside manifests | Patch ApplicationSet `destination.namespace` |
| 6 | Container image | Bitnami removed `bitnami/mongodb:5.0.10` from Docker Hub | Use `bitnamilegacy/mongodb:5.0.10` |
| 7 | PV mount path | Bitnami overrides mongod `dbPath` to `/bitnami/mongodb` | Mount PVC at `/bitnami/mongodb` |
| 8 | UID drift across clusters | OpenShift assigns each namespace its own uid range; on-disk uid doesn't match new pod | Pin namespace SCC annotations identically on both clusters |
| 9 | DR validation | — | End-to-end failover succeeded, data preserved across cluster switch |
| 10 | Argo + DR Relocate | `prune=true` cascades during placement-decision drain | Disable prune, set `PrunePropagationPolicy=orphan`, set ApplicationSet `preserveResourcesOnDeletion=true` |
| 11 | Azure cluster install | Worker `vmSize` defaulted to `Standard_DCSv2` (Confidential Computing); subscription quota 2 cores → workers never created → install timed out | Use `Standard Dv3 Family vCPUs` or `Standard Dv4 Family vCPUs` for workers (e.g., `Standard_D4s_v3` / `Standard_D4s_v4`); verify quota with `az vm list-usage` before install |
| 12 | Documentation | Knowledge gap for next operator | README updates committed to workload repo |
