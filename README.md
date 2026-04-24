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

Three OCP clusters:

- **Hub (AWS)** — ACM 2.15 hub, ODF Multicluster Orchestrator
- **Managed cluster 1 (AWS)** — planned
- **Managed cluster 2 (Azure, eastus)** — planned

## Status

Work in progress. See `docs/` for architecture notes and decisions as they are added.
