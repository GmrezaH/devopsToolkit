# ingress-nginx Installation and Configuration

The [ingress-nginx](https://github.com/kubernetes/ingress-nginx) Helm chart deploys an NGINX-based Ingress controller for Kubernetes, acting as a reverse proxy and load balancer to manage external traffic to services. This guide outlines the steps to install and configure ingress-nginx in a Kubernetes cluster, with considerations for air-gapped environments.

## Requirements

- Kubernetes: `>=1.21.0-0`
- Helm: `>=3`
- ingress-nginx (CHART VERSION): `4.13.1`
- Access to the ingress-nginx Helm repository or a local mirror (for air-gapped environments)
- For air-gapped setups, ensure the `ingress-nginx` Helm chart and images are mirrored to the local Nexus repository.

## Adding the Helm Repository

Add and update the `ingress-nginx` Helm repository. In air-gapped environments, use a local Nexus repository with mirrored charts:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

## Install chart

Install the ingress-nginx chart using Helm 3 with the provided [values.yaml](./values.yaml) file:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --version 4.13.1 \
  -f values.yaml
```

This command deploys `ingress-nginx` **without monitoring enabled**, as specified in the installation checklist. Verify the deployment:

```bash
kubectl get pods -n ingress-nginx
```

## Configuring Ingress-Nginx

After installation, configure the Ingress-Nginx controller to handle external traffic effectively and ensure compatibility with your cluster's requirements, such as integration with Cert-Manager for TLS or custom load balancer settings.

1. **Verify Controller Status**:
   Check that the Ingress-Nginx controller is running and accessible:

   ```bash
   kubectl get svc -n ingress-nginx
   ```

   Identify the service's external IP or LoadBalancer IP (if applicable). For clusters using a local HAProxy/Keepalived setup, ensure the service is reachable via the virtual IP (VIP).

2. **Create an Ingress Resource**:
   Define an Ingress resource to route traffic to a service. Example for a simple application:

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: example-ingress
     namespace: default
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
     ingressClassName: nginx
     rules:
       - host: example.test.local
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: example-service
                   port:
                     number: 80
   ```

   Apply the resource:

   ```bash
   kubectl apply -f example-ingress.yaml
   ```

3. **Integrate with Cert-Manager for TLS**:
   To secure Ingress resources with TLS certificates, configure Cert-Manager integration. Add an annotation to the Ingress resource to reference a Cert-Manager issuer (e.g., `cluster-ca-issuer`):

   ```yaml
   metadata:
     annotations:
       cert-manager.io/cluster-issuer: cluster-ca-issuer
   spec:
     tls:
       - hosts:
           - example.test.local
         secretName: example-tls
   ```

   Ensure the `cluster-ca-issuer` is configured (see `cert-manager/README.md`).

4. **Test Connectivity**:
   Verify that the Ingress is routing traffic correctly. If using a local DNS (e.g., `nexus.test.local`), add the Ingress host to `/etc/hosts` or your DNS server:

   ```bash
   echo "192.168.200.10 example.test.local" >> /etc/hosts
   ```

   Test with:

   ```bash
   curl http://example.test.local
   ```

## Upgrading for Monitoring

To enable monitoring after `kube-prometheus-stack` installation (e.g., Prometheus integration):

1. Update [values.yaml](./values.yaml) to include monitoring configurations (e.g., enable prometheus metrics endpoint).

1. Upgrade the Helm release:

   ```bash
   helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
     --namespace ingress-nginx \
     -f values.yaml
   ```

## Resources

- [ingress-nginx repository docs](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx)
