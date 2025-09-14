# kube-prometheus-stack Installation

<div align="center">
  <img src="../../images/prometheus.png" alt="kube-prometheus-stack logo" />
</div>

Installs core components of the [kube-prometheus](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) stack, a collection of Kubernetes manifests, Grafana dashboards, and Prometheus rules combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with Prometheus using the Prometheus Operator.

## Prerequisites

- **Kubernetes**: `>=1.19`
- **Helm**: `>= 3.0`
- **Chart Versions**:
  - kube-prometheus-stack: `77.0.1`
- **Repository Access**: Access to the `prometheus-community` Helm and image repository or a local mirror (e.g., Nexus) for air-gapped setups.

## Adding the Helm Repository

Add and update the `prometheus-community` Helm repository. In air-gapped environments, use a local Nexus repository with mirrored charts:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## Preparation

1. Create `monitoring` namespace with `privileged` pod security level:

   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Namespace
   metadata:
     name: monitoring
     labels:
          pod-security.kubernetes.io/audit: privileged
          pod-security.kubernetes.io/audit-version: v1.32
          pod-security.kubernetes.io/enforce: privileged
          pod-security.kubernetes.io/enforce-version: v1.32
          pod-security.kubernetes.io/warn: privileged
          pod-security.kubernetes.io/warn-version: v1.32
   EOF
   ```

1. Create and Apply the credentials secret for ingress-nginx basic authentication:

   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Secret
   metadata:
     name: prometheus-auth
     namespace: monitoring
   type: Opaque
   data:
     auth: $(htpasswd -nbm admin admin | base64)
   EOF
   ```

   Make sure `httpd-tools` package is installed.

1. **(Optional)** If you installed `etcd` as static pods, create secret for ETCD https serviceMonitor:

   ```bash
   kubectl -n monitoring create secret generic etcd-client-cert --from-file=etcd-ca=/etc/kubernetes/ssl/etcd/ca.crt --from-file=etcd-client=/etc/kubernetes/ssl/etcd/healthcheck-client.crt --from-file=etcd-client-key=/etc/kubernetes/ssl/etcd/healthcheck-client.key  --dry-run=client -o yaml
   ```

## Installing the Helm Chart

Install the `kube-prometheus-stack` chart using Helm 3 with the provided [values.yaml](./values.yaml) file:

```bash
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f values.yaml
```

Verify the deployment:

```bash
kubectl get pods -n monitoring
```

## Resources

- [kube-prometheus-stack repository](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
