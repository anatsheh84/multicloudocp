# multicloudocp

Evaluation project for cross-region and cross-cloud disaster recovery
architectures for OpenShift workloads, using OpenShift Data Foundation
Regional-DR with Red Hat Advanced Cluster Management.

## Scope

- Self-managed OpenShift Container Platform (not ROSA, ARO, or OSD)
- Containerized applications and VMs (via OpenShift Virtualization)
- State lives in PVCs; managed data services are out of scope
- Active-passive DR; whole-region failure is the reference scenario

## Topology

- **Hub (AWS, us-east-1)** — SNO, ACM 2.15, ODF Multicluster Orchestrator. Installed.
- **ocpaws1 (AWS, us-east-1)** — installed
- **ocpaws2 (AWS, eu-central-1)** — provisioning, planned failover target for ocpaws1
- **ocpazure (Azure, eastus)** — planned

## Operations notes

### Recovering after long hibernation

If a Hive-managed cluster is hibernated for more than 24 hours, kubelet client certs expire. Symptoms: `oc login` fails (`EOF` or `401`), apps routes return TLS errors, ELB targets show `OutOfService`.

Fix:

1. Extract the admin kubeconfig from the hub:
   ```
   oc extract secret/<infraID>-admin-kubeconfig -n <cluster-namespace> --to=- --keys=kubeconfig > admin.kubeconfig
   ```
2. Approve all pending CSRs (run twice — a second wave appears once kubelets reconnect):
   ```
   KUBECONFIG=admin.kubeconfig oc adm certificate approve $(KUBECONFIG=admin.kubeconfig oc get csr -o name)
   ```
3. Wait a few minutes for OVN, ingress, and OAuth to converge. `oc login` will work again.

## Status

Work in progress. See `docs/` for architecture notes and decisions as they are added.
