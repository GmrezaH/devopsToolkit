# ingress-nginx Installation and Configuration

[k8tz](https://github.com/k8tz/k8tz) is a kubernetes admission controller and a CLI tool to inject timezones into Pods and CronJobs1.

Containers do not inherit timezones from host machines and have only accessed to the clock from the kernel. The default timezone for most images is UTC, yet it is not guaranteed and may be different from container to container. With `k8tz` it is easy to standardize selected timezone across pods and namespaces automatically with minimal effort.

## Prerequisites

- **Helm**: `>= 3.0`
- **Chart Versions**:
  - k8tz: `0.18.0`
- **Repository Access**: Access to the `k8tz` Helm and image repository or a local mirror (e.g., Nexus) for air-gapped setups.

## Adding the Helm Repository

Add and update the `k8tz` Helm repository. In air-gapped environments, use a local Nexus repository with mirrored charts:

```bash
helm repo add k8tz https://k8tz.github.io/k8tz/
helm repo update
```

## Installing the Helm Chart

Install the k8tz chart using Helm 3 with the provided [values.yaml](./values.yaml) file:

```bash
helm install k8tz k8tz/k8tz \
  --namespace k8tz \
  --create-namespace \
  -f values.yaml
```

Verify the deployment:

```bash
kubectl get pods -n k8tz
```

## Resources

- [k8tz repository docs](https://github.com/k8tz/k8tz)
