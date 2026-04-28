# Deploying a DR-Protected Application on ACM + ODF Regional-DR

Practical runbook for taking a Helm-packaged application and bringing it under
ODF Regional-DR protection across two managed clusters. Assumes the underlying
infrastructure (ACM hub, ODF on both managed clusters, ODF Multicluster
Orchestrator, MirrorPeer, DRClusters) is already installed and healthy.

Validated end-to-end against `globex` (Helm chart, two PostgreSQL PVCs)
deployed on `ocpaws1`/`ocpaws2`. Round-trip relocate and failover tested with
zero data loss for replicated writes.

---

## Environment

| Component | Identifier |
|---|---|
| Hub | `cluster-gr4bg` — ACM 2.16, ODF Multicluster Orchestrator 4.21.2, GitOps 1.20.2 |
| Primary cluster | `ocpaws1` — ODF 4.21.2, default SC `ocs-storagecluster-ceph-rbd` |
| Secondary cluster | `ocpaws2` — ODF 4.21.2, same default SC |
| ManagedClusterSet | `aws-regional-dr` (binds ocpaws1 + ocpaws2) |
| Workload | `globex` Helm chart at `https://github.com/anatsheh84/globex-dr.git` |
| PVC label selector | `app=globex` (matches both `catalog-database` and `inventory-database`) |

---

## Pre-flight: gaps to fix before enabling DR

Even a fully installed ODF + MCO stack typically has three configuration
gaps that block DR. Fix all three on the hub before creating the DRPolicy.

### 1. DRClusters need distinct `spec.region`

Required so the DRPolicy can validate that two distinct sites exist. The
values are arbitrary labels — they don't have to match real cloud regions,
just be different strings.

```bash
oc patch drcluster ocpaws1 --type=merge -p '{"spec":{"region":"aws-us-east-1"}}'
oc patch drcluster ocpaws2 --type=merge -p '{"spec":{"region":"aws-us-east-2"}}'
```

Verify:

```bash
oc get drcluster -A -o custom-columns='NAME:.metadata.name,REGION:.spec.region,PHASE:.status.phase'
# Both should show distinct REGION and PHASE=Available
```

### 2. Placement (used by GitOpsCluster) needs DR tolerations

When a managed cluster goes unreachable, ACM applies taints. Without these
tolerations, the Placement controller would silently drop the cluster from
the decision while DRPC was still trying to orchestrate failover deliberately
— the two controllers would race. Tolerations make Placement "sticky" so DRPC
is the only thing that moves the workload.

```bash
oc patch placement aws-regional-dr-placement -n openshift-gitops \
  --type=merge -p '{"spec":{"tolerations":[
    {"key":"cluster.open-cluster-management.io/unreachable","operator":"Exists"},
    {"key":"cluster.open-cluster-management.io/unavailable","operator":"Exists"}
  ]}}'
```

### 3. ClusterRoleBinding for the local ArgoCD on each managed cluster

Pull-model ApplicationSets are reconciled by the *managed cluster's* local
ArgoCD. The default GitOps install grants its application-controller SA only
narrow namespace permissions. Cluster-admin is heavy-handed but is the
documented baseline because DR-orchestrated workloads can include
cluster-scoped resources over time. Apply on **both** managed clusters.

```yaml
# argocd-crb.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshift-gitops-argocd-application-controller-admin
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

```bash
KUBECONFIG=<ocpaws1> oc apply -f argocd-crb.yaml
KUBECONFIG=<ocpaws2> oc apply -f argocd-crb.yaml
```

---

## Step 1: Create the DRPolicy (hub)

```yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata:
  name: async-3min
spec:
  drClusters: [ocpaws1, ocpaws2]
  schedulingInterval: 3m
```

`schedulingInterval` is the RPO target. 1m is the floor; 3–5m is the
practical baseline for Regional-DR with RBD. Verify:

```bash
oc get drpolicy async-3min -o jsonpath='{.status.conditions[0]}{"\n"}'
# Expect: type=Validated status=True reason=Succeeded
```

---

## Step 2: Deploy the app via ApplicationSet (Pull model)

Use the ACM console wizard (Applications → Create application set → **Pull
model**). Key wizard choices:

| Step | Value |
|---|---|
| Argo server | The GitOpsCluster bound to the DR ManagedClusterSet |
| Generators | Default (`Cluster Decision Resource Generator`, requeue 180s) |
| Repository | Git, public repo URL, `main`, path `.` |
| Sync policy | Defaults are fine: prune, selfHeal, automated, CreateNamespace |
| Placement | New placement, cluster set = `aws-regional-dr`, label `name In <primary>`, limit 1 |

**Critical YAML edit before submitting** — the wizard generates
`destination.server: "{{server}}"`, which is wrong for Pull model.
The local ArgoCD on the managed cluster only knows about itself as
`https://kubernetes.default.svc`. Toggle YAML view and change:

```yaml
destination:
  namespace: globex
  server: https://kubernetes.default.svc   # <— hardcoded, not {{server}}
```

The new placement won't have the DR tolerations from Pre-flight #2 — patch it
afterwards (the wizard doesn't expose tolerations in the form):

```bash
oc patch placement globex-placement -n openshift-gitops \
  --type=merge -p '{"spec":{"tolerations":[
    {"key":"cluster.open-cluster-management.io/unreachable","operator":"Exists"},
    {"key":"cluster.open-cluster-management.io/unavailable","operator":"Exists"}
  ]}}'
```

Verify the app lands on the primary:

```bash
KUBECONFIG=<primary> oc get pods -n globex
KUBECONFIG=<primary> oc get pvc -n globex
# PVCs must be Bound on the RBD storage class
```

---

## Step 3: Enroll the app in DR (apply DRPlacementControl)

Use the ACM console (Applications → ⋮ → Manage disaster recovery → Enroll)
or apply directly:

```yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPlacementControl
metadata:
  name: globex-placement-drpc
  namespace: openshift-gitops
  labels:
    cluster.open-cluster-management.io/backup: ramen
spec:
  preferredCluster: ocpaws1
  drPolicyRef:
    name: async-3min
  placementRef:
    kind: Placement
    name: globex-placement
  pvcSelector:
    matchLabels:
      app: globex
```

`pvcSelector` must uniquely match this app's PVCs. If you select a label
shared with other workloads, ODF will manage all of them under this DRPC and
you'll get conflicts.

---

## Verification checklist

After the DRPC is applied, expect within ~1 minute:

```bash
# Hub: DRPC reconciled
oc get drpc -A -o wide
# Expect: PHASE=Deployed, PROGRESSION=Completed, PEER READY=True

# Primary: VRG created
KUBECONFIG=<primary> oc get vrg -n globex
# Expect: state=Primary, replicationState=primary

# Primary: VolumeReplication / VolumeGroupReplication populated
KUBECONFIG=<primary> oc get volumereplication,volumegroupreplication -n globex
```

Then wait one full sync interval (3 min for `async-3min`) and verify
replication is actively flowing:

```bash
# RBD-mirror sees images replaying
KUBECONFIG=<primary> oc rsh -n openshift-storage <toolbox-pod> \
  rbd mirror pool status ocs-storagecluster-cephblockpool
# Expect: "X images" and "X replaying", health: OK

# DRPC lastGroupSyncTime is recent
oc get drpc <name> -n openshift-gitops -o jsonpath='{.status.lastGroupSyncTime}'
# Should be within the last 2 × schedulingInterval
```

---

## DR operations

| | Relocate | Failover |
|---|---|---|
| Use when | Both clusters healthy, planned move | Source cluster is down/lost |
| Source-side behavior | Quiesces app, waits for final sync | Skips quiesce, assumes source is gone |
| Data loss | Zero | Up to one sync interval (writes after `lastGroupSyncTime` are lost) |
| Reversibility | Symmetric — relocate back any time | Asymmetric — needs cleanup once source returns |

ACM console: Applications → ⋮ → Manage disaster recovery → choose action.

---

## Known transient state during relocate

After a successful relocate, the DRPC's `Protected` condition flips to
`False` for up to one sync interval *after* `Available` flips to `True`. The
ACM console may briefly show "DR protection failed" during this window.

**This is not a failure.** The app is running on the new primary and writes
are being mirrored to the old primary in real time. The `Protected`
condition only flips back to `True` once Ramen has *verified* the first sync
cycle from the new primary completes — it's a verification gap, not a
replication gap.

The Red Hat `WorkloadUnprotected` Prometheus alert has a 10-minute grace
period (`for: 10m`) for exactly this reason. If alerting on `Protected:
False`, suppress for at least `2 × schedulingInterval`.

---

## Validating with marker rows

The cleanest way to prove end-to-end replication actually preserves
application data is to write distinctively-tagged rows before triggering DR:

```sql
-- On primary, before DR action
INSERT INTO catalog (itemid, name, description, price)
VALUES ('DR-TEST-001', 'DR Marker',
        'Written on <primary> at <timestamp>', 99.99);
```

After relocate/failover, query the same tables on the new primary:

```sql
SELECT * FROM catalog WHERE itemid='DR-TEST-001';
SELECT COUNT(*) FROM catalog;   -- expect: original_count + 1
```

If both the marker row and the row count match, the full path is
verified: app write → PostgreSQL WAL → block write → RBD mirror →
secondary cluster's RBD → Postgres mount → query.

---

## Common gotchas

- **Wizard picks Push model by default.** Push model isn't compatible with
  DRPC orchestration — always select Pull model.
- **Wizard sets `server: "{{server}}"` for Pull model.** Must be hardcoded
  to `https://kubernetes.default.svc` for the local ArgoCD to recognize the
  destination.
- **GitOps must be installed on every managed cluster** that may host the
  workload. Pull model relies on the local ArgoCD; if it's missing, the
  ManifestWork lands but never reconciles.
- **Don't reuse the GitOpsCluster's Placement for the app.** DRPC mutates
  the placement decision during failover; reusing the GitOpsCluster's
  placement breaks cluster registration. Always create a new per-app
  Placement.
- **`mirroringStatus: {}` and missing VolumeReplicationClass before DRPC** are
  not bugs. Both populate lazily once a workload is enrolled.
- **`MultiClusterHub` may show `Pending`** while all sub-components show
  `True`. This is a stale top-level condition, not a real problem.
