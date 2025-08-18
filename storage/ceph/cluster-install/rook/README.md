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

### Adding the Helm Repository

The release channel is the most recent release of Rook that is considered stable for the community.
Add and update the `rook-release` Helm repository (or configure a local mirror for offline setups)

```bash
helm repo add rook-release https://charts.rook.io/release --force-update
```

### Prepare nodes

Before installing rook ceph operator you need to add labels to kubernetes node (that have disks attached and going to host ceph nodes) for operator to use it in nodeSelector:

```bash
kubectl label nodes master1 disktype=ssd
kubectl label nodes master2 disktype=ssd
kubectl label nodes master3 disktype=ssd
```

Verify node labels:

```bash
kubectl get nodes --show-labels
```

### Ceph Operator Helm Chart

Installs [rook](https://github.com/rook/rook) to create, configure, and manage Ceph clusters on Kubernetes.

**Before installing, review the [rook-ceph-values.yaml](./rook-ceph-values.yaml) to confirm if the default settings need to be updated.**

```bash
helm install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --create-namespace \
  --version v1.17.7 \
  -f rook-ceph-values.yaml
```

> [!NOTE]
> It is recommended that the rook operator be installed into the `rook-ceph` namespace. The clusters can be installed into the same namespace as the operator or a separate namespace.

### Ceph Cluster Helm Chart

Creates Rook resources to configure a [Ceph](https://ceph.io/en/) cluster using the [Helm](https://helm.sh/) package manager. This chart is a simple packaging of templates that will optionally create Rook resources such as:

- CephCluster, CephFilesystem, and CephObjectStore CRs
- Storage classes to expose Ceph RBD volumes, CephFS volumes, and RGW buckets
- Ingress for external access to the dashboard
- Toolbox

**Before installing, review the [rook-ceph-cluster-values.yaml](./rook-ceph-cluster-values.yaml) to confirm if the default settings need to be updated.**

- If the operator was installed in a namespace other than `rook-ceph`, the namespace must be set in the `operatorNamespace`variable.

- The default values for cephBlockPools, cephFileSystems, and CephObjectStores will create one of each, and their corresponding storage classes.

```bash
helm install rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  --create-namespace \
  --version v1.17.7 \
  -f rook-ceph-cluster-values.yaml
```

## Resources

- [Rook Ceph docs](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)

- [Rook Ceph helm charts](https://rook.io/docs/rook/latest-release/Helm-Charts/helm-charts/)
