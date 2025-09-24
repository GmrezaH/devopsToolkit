# Argocd Installation and Configuration

<div align="center">
  <img src="../../images/argo.png" alt="Argo logo" />
</div>

[Argo CD](https://argo-cd.readthedocs.io) is a declarative, GitOps continuous delivery tool for Kubernetes.

## Prerequisites

- Kubernetes: `>=1.30`
- Helm: `>=3`
- Argo-cd helm chart (CHART VERSION): `8.5.6`
- Access to the `Argo` Helm and image repository or a local mirror (for air-gapped environments).
- [Cert-manager](../../kubernetes/addons/cert-manager/README.md) installed and an issuer configured for issuing certificate.
- [Ingress controller](../../kubernetes/addons/ingress-nginx/README.md) installed.

## Preparation

Argo expects the password in the secret to be **bcrypt hashed**. You can create Admin password hash with:

```bash
sudo dnf install -y httpd-tools
htpasswd -nbBC 10 "" $ARGO_PWD | tr -d ':\n' | sed 's/$2y/$2a/'
```

## Adding the Helm Repository

Add and update the `Argo` Helm repository (or configure a local mirror for offline setups)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

## Installing the Helm Chart

Use Helm 3 to install `Argocd` with the provided [values.yaml](./values.yaml) file:

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 8.5.6 \
  -f values.yaml
```

> [!NOTE]
> Note that the `nginx.ingress.kubernetes.io/ssl-passthrough` annotation requires that the `--enable-ssl-passthrough` flag be added to the command line arguments to `nginx-ingress-controller`.

This command deploys `Argocd`. Verify the deployment:

```bash
kubectl get pods -n argocd
```

## Resources

- [argo-helm repository](https://github.com/argoproj/argo-helm/tree/main)

- [argo-cd installation](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)
