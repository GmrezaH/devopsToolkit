# cert-manager Installation and Configuration

cert-manager creates TLS certificates for workloads in your Kubernetes or OpenShift cluster and renews the certificates before they expire.

cert-manager can obtain certificates from a variety of certificate authorities, including: [Let's Encrypt](https://cert-manager.io/docs/configuration/acme/), [HashiCorp Vault](https://cert-manager.io/docs/configuration/vault/), [Venafi](https://cert-manager.io/docs/configuration/venafi/), [private PKI](https://cert-manager.io/docs/configuration/ca/) and [SelfSigned](https://cert-manager.io/docs/configuration/selfsigned/).

With cert-manager's [Certificate resource](https://cert-manager.io/docs/usage/certificate/), the private key and certificate are stored in a Kubernetes Secret which is mounted by an application Pod or used by an Ingress controller. With csi-driver, csi-driver-spiffe, or istio-csr , the private key is generated on-demand, before the application starts up; the private key never leaves the node and it is not stored in a Kubernetes Secret.

## Requirements

- Kubernetes: `>=1.29`
- Helm: `>=3`
- cert-manager (CHART VERSION): `v1.18.2`
- Access to the Jetstack Helm repository or a local mirror (for air-gapped environments).
- For air-gapped setups, ensure the `cert-manager` Helm chart and images are mirrored to the local Nexus repository.

## Adding the Helm Repository

Add and update the Jetstack Helm repository (or configure a local mirror for offline setups)

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
```

## Installing the Helm Chart

Use Helm 3 to install Cert-Manager with the provided [values.yaml](./values.yaml) file:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.18.2 \
  -f values.yaml
```

This command deploys `cert-manager` **without monitoring enabled**, as specified in the installation checklist. Verify the deployment:

```bash
kubectl get pods -n cert-manager
```

## Bootstrapping CA Issuers

A common use case for Cert-Manager is to bootstrap a private PKI using a self-signed root certificate, which can then be used with a `CA` or `ClusterIssuer` to sign certificates.

### Option 1: Namespace-Scoped CA Issuer

The YAML below will create a `SelfSigned` issuer, issue a root certificate and configures a namespace-scoped `Issuer`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sandbox
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: sandbox
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: my-ca-issuer
  namespace: sandbox
spec:
  ca:
    secretName: root-secret
```

### Option 2: Cluster-Wide CA Issuer

Alternatively, if you are looking to use `ClusterIssuer` for signing Certificates across all namespaces with the `SelfSigned Certificate CA`, use a `ClusterIssuer` with the root CA stored in the `cert-manager` namespace:

```yaml
# `sandbox` namespace is no longer needed.
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  # Create CA root secret in `cert-manager` namespace instead of `sandbox` namespace.
  namespace: cert-manager
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    # `ClusterIssuer` resource is not namespaced, so `secretName` is assumed to reference secret in `cert-manager` namespace.
    secretName: root-secret
```

The selfsigned-issuer `ClusterIssuer` is used to issue the Root CA Certificate. Then, `my-ca-issuer` ClusterIssuer is used to issue but also sign certificates using the newly created Root CA `Certificate`, which is what you will use for future certificates cluster-wide.

## Obtain CA certificates

To enable browser or client trust for certificates issued by Cert-Manager's CA, you must extract and distribute the root CA certificate (`tls.crt`) from the CA secret (`root-secret`). This ensures secure communication without SSL/TLS warnings for services using Cert-Manager-issued certificates.

1. Extract the CA certificate and save it as `cluster-ca.crt`:

   ```bash
   kubectl get secrets -n cert-manager root-secret -o jsonpath='{.data.tls\.crt}' | base64 -d > cluster-ca.crt
   ```

1. Import the `cluster-ca.crt` into the trust store:

   - **Chrome/Edge**: Settings > Privacy and Security > Manage Certificates > Authorities > Import `cluster-ca.crt`.
   - **Firefox**: Settings > Privacy & Security > Certificates > View Certificates > Authorities > Import `cluster-ca.crt`.
   - **Linux**: Copy `cluster-ca.crt` to `/usr/local/share/ca-certificates/` and run `sudo update-ca-certificates`.
   - **macOS**: Open Keychain Access, import `cluster-ca.crt` into the "System" keychain, and set it to "Always Trust" for SSL.

1. Restart the browser or application to apply changes. Verify by accessing a service secured by a Cert-Manager certificate.

## Upgrading for Monitoring

To enable monitoring after `kube-prometheus-stack` installation (e.g., Prometheus integration):

1. Update `values.yaml` to include monitoring configurations.
1. Upgrade the Helm release:

   ```bash
   helm upgrade cert-manager jetstack/cert-manager \
     --namespace cert-manager \
     -f values.yaml
   ```

## Resources

- [Official cert-manager docs](https://cert-manager.io/docs/)

- [Official cert-manager installation docs](https://cert-manager.io/docs/installation/helm/)

- [Cert-manager configuring SelfSigned issuer docs](https://cert-manager.io/docs/configuration/selfsigned/)
