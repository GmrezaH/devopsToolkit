# Install ingress-nginx

[ingress-nginx](https://github.com/kubernetes/ingress-nginx) Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer.
This chart bootstraps an ingress-nginx deployment on a Kubernetes cluster using the Helm package manager.

## Requirements

- Kubernetes: `>=1.21.0-0`

## Add the repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

## Install chart

**Important:** only helm3 is supported

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx -f values-ingress-nginx-4.12.0.yaml --create-namespace
```
