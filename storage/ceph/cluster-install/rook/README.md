# Rook

Rook is an open source cloud-native storage orchestrator, providing the platform, framework, and support for Ceph storage to natively integrate with cloud-native environments.

[Ceph](https://ceph.com/) is a distributed storage system that provides file, block and object storage and is deployed in large scale production clusters.

Rook automates deployment and management of Ceph to provide self-managing, self-scaling, and self-healing storage services. The Rook operator does this by building on Kubernetes resources to deploy, configure, provision, scale, upgrade, and monitor Ceph.

The Ceph operator was declared stable in December 2018 in the Rook v0.9 release, providing a production storage platform for many years. Rook is hosted by the [Cloud Native Computing Foundation (CNCF)](https://www.cncf.io/) as a [graduated](https://www.cncf.io/announcements/2020/10/07/cloud-native-computing-foundation-announces-rook-graduation/) level project.

## Helm Charts

Rook has published the following Helm charts for the Ceph storage provider:

- [Rook Ceph Operator](https://rook.io/docs/rook/latest-release/Helm-Charts/operator-chart/): Starts the Ceph Operator, which will watch for Ceph CRs (custom resources)
- [Rook Ceph Cluster](https://rook.io/docs/rook/latest-release/Helm-Charts/ceph-cluster-chart/): Creates Ceph CRs that the operator will use to configure the cluster

The Helm charts are intended to simplify deployment and upgrades. Configuring the Rook resources without Helm is also fully supported by creating the [manifests](https://github.com/rook/rook/tree/release-1.17/deploy/examples) directly.

### Prerequisites

- Helm: `>=3`
- rook-ceph (CHART VERSION): `v1.17.7`
- rook-ceph-cluster (CHART VERSION): `v1.17.7`
- Access to the `rook-ceph` and `rook-ceph-cluster` Helm and image repository or a local mirror (for air-gapped environments).

### Ceph Operator Helm Chart

Installs [rook](https://github.com/rook/rook) to create, configure, and manage Ceph clusters on Kubernetes.

### Ceph Cluster Helm Chart

Creates Rook resources to configure a [Ceph](https://ceph.io/en/) cluster using the [Helm](https://helm.sh/) package manager. This chart is a simple packaging of templates that will optionally create Rook resources such as:

- CephCluster, CephFilesystem, and CephObjectStore CRs
- Storage classes to expose Ceph RBD volumes, CephFS volumes, and RGW buckets
- Ingress for external access to the dashboard
- Toolbox

## Resources

- [Rook Ceph docs](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)

- [Rook Ceph helm charts](https://rook.io/docs/rook/latest-release/Helm-Charts/helm-charts/)
