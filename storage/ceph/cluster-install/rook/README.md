# Rook Ceph Installation Guide

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

- **Helm**: `>= 3.0`
- **Chart Versions**:
  - rook-ceph: `v1.17.7`
  - rook-ceph-cluster: `v1.17.7`
- **Repository Access**: Access to the `rook-ceph` and `rook-ceph-cluster` Helm and image repository or a local mirror (e.g., Nexus) for air-gapped setups.
- **Kubernetes**: Version compatible with Rook v1.17.7 (v1.28).
- **Nodes**: Disks attached to nodes designated for Ceph storage.

### Adding the Helm Repository

The release channel is the most recent release of Rook that is considered stable for the community. Add and update the `rook-release` Helm repository (or configure a local mirror for offline setups).

```bash
helm repo add rook-release https://charts.rook.io/release --force-update
```

### Prepare nodes

Label Kubernetes nodes with attached disks for Ceph storage to enable node selection by the Rook operator:

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

1. Review and customize the [rook-ceph-values.yaml](./rook-ceph-values.yaml) file for your environment.
1. Install the chart:

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

1. Review and customize the [rook-ceph-cluster-values.yaml](./rook-ceph-cluster-values.yaml) file, ensuring:
   - `operatorNamespace` matches the operator's namespace (e.g., `rook-ceph`).
   - Storage configurations align with your disk setup.
   - Air-gapped repository settings point to your Nexus instance.
1. Install the chart:

   ```bash
   helm install rook-ceph-cluster rook-release/rook-ceph-cluster \
     --namespace rook-ceph \
     --create-namespace \
     --version v1.17.7 \
     -f rook-ceph-cluster-values.yaml
   ```

## Post-install

### Change RGW service count

To scale the Ceph Object Store (RGW) services:

1. Retrieve RGW details:

   ```bash
   kubectl -n rook-ceph exec -ti deployments.apps/rook-ceph-tools -- radosgw-admin realm list
   kubectl -n rook-ceph exec -ti deployments.apps/rook-ceph-tools -- radosgw-admin zonegroup list
   kubectl -n rook-ceph exec -ti deployments.apps/rook-ceph-tools -- radosgw-admin zone list
   ```

1. Check current RGW services:

   ```bash
   kubectl -n rook-ceph exec -ti deployments.apps/rook-ceph-tools -- ceph orch ps --daemon-type rgw
   ```

1. Scale RGW to 3 replicas on specified nodes:

   ```bash
   kubectl -n rook-ceph exec -ti deployments.apps/rook-ceph-tools -- ceph orch apply rgw ceph-objectstore --realm=ceph-objectstore --zone=ceph-objectstore --zonegroup=ceph-objectstore --placement="3 master1 master2 master3"
   ```

1. Verify the updated RGW services:

   ```bash
   kubectl -n rook-ceph exec -ti deployments.apps/rook-ceph-tools -- ceph orch ps --daemon-type rgw
   ```

### Retrieve Dashboard Credentials

After you connect to the dashboard you will need to login for secure access. Rook creates a default user named `admin` and generates a secret called `rook-ceph-dashboard-password` in the namespace where the Rook Ceph cluster is running. To retrieve the generated password, you can run the following:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

### Verify Cluster Health

Check the Ceph cluster status using the toolbox:

```bash
kubectl -n rook-ceph exec -ti deployments.apps/rook-ceph-tools -- ceph status
```

Ensure the cluster reports `HEALTH_OK` or address any warnings.

## Resources

- [Rook Ceph docs](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)

- [Rook Ceph helm charts](https://rook.io/docs/rook/latest-release/Helm-Charts/helm-charts/)
