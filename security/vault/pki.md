# PKI secrets engine

The PKI secrets engine generates dynamic X.509 certificates. With this secrets engine, services can get certificates without going through the usual manual process of generating a private key and CSR, submitting to a CA, and waiting for a verification and signing process to complete. Vault's built-in authentication and authorization mechanisms provide the verification functionality.

<div align="center">
  <img src="../../images/vault-pki.png" alt="Hashicorp Vault PKI architecture" />
</div>

## Prerequisites

- A Vault environment. Refer to the [Vault install guide](./README.md) to install Vault.
- `jq` and `openssl` installed.

### Policy requirements

To perform all tasks demonstrated in this tutorial, your policy must include the following capabilities:

```sh
# Enable secrets engine
path "sys/mounts/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# List enabled secrets engine
path "sys/mounts" {
  capabilities = [ "read", "list" ]
}

# Work with pki secrets engine
path "pki*" {
  capabilities = [ "create", "read", "update", "delete", "list", "sudo", "patch" ]
}
```

## Generate root certificate authority

Generate a self-signed root certificate using PKI secrets engine.

1. Start an interactive shell session on the `vault-0` pod.

   ```shell
   kubectl -n vault exec -ti vault-0 -- sh
   ```

1. Enable the `pki` secrets engine at the `pki` path.

   ```sh
   vault secrets enable pki
   ```

1. Tune the `pki` secrets engine to issue certificates with a maximum time-to-live (TTL) of 87600 hours.

   ```sh
   vault secrets tune -max-lease-ttl=87600h pki
   ```

1. Generate the **example.com** root CA, give it an issuer name, and save its certificate in the file `root_2025_ca.crt`.

   ```sh
   vault write -field=certificate pki/root/generate/internal \
        common_name="example.com" \
        issuer_name="root-2025" \
        ttl=87600h > /tmp/root_2025_ca.crt
   ```

   This generates a new self-signed CA certificate and private key. Vault automatically revokes the generated root at the end of its lease period (TTL). The CA certificate signs its own Certificate Revocation List (CRL).

1. List the issuer information for the root CA.

   ```sh
   vault list pki/issuers/
   ```

   Copy this key, you'll use this key in the next step.

1. You can read the issuer with its ID to get the certificates and other metadata about the issuer. Skip the certificate output, but list the issuer metadata and usage information.

   ```sh
   vault read pki/issuer/$(vault list -format=json pki/issuers/ | jq -r '.[]') | tail -n 6
   ```

1. Create a role for the root CA. Creating this role allows for specifying an issuer when necessary for the purposes of this scenario. This also provides a simple way to transition from one issuer to another by referring to it by name.

   ```sh
   vault write pki/roles/2025-servers allow_any_name=true
   ```

1. Configure the CA and CRL URLs.

   ```sh
   vault write pki/config/urls \
      issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
      crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
   ```

## Generate intermediate CA

Create an intermediate CA using the root CA you regenerated in the first step.

1. Enable the `pki` secrets engine at the `pki_int` path.

   ```sh
   vault secrets enable -path=pki_int pki
   ```

1. Tune the `pki_int` secrets engine to issue certificates with a maximum time-to-live (TTL) of 43800 hours.

   ```sh
   vault secrets tune -max-lease-ttl=43800h pki_int
   ```

1. Execute the following command to generate an intermediate and save the CSR as `pki_intermediate.csr`.

   ```sh
   exit

   kubectl exec vault-0 -it -n vault -- \
      vault write -format=json pki_int/intermediate/generate/internal \
        common_name="example.com Intermediate Authority" \
        issuer_name="example-dot-com-intermediate" \
        | jq -r '.data.csr' > pki_intermediate.csr

   kubectl cp pki_intermediate.csr vault/vault-0:/tmp
   ```

1. Sign the intermediate certificate with the root CA private key, and save the generated certificate as `intermediate.cert.pem`.

   ```sh
   kubectl exec vault-0 -it -n vault -- \
      vault write -format=json pki/root/sign-intermediate \
        issuer_ref="root-2025" \
        csr=@/tmp/pki_intermediate.csr \
        format=pem_bundle ttl="43800h" \
        | jq -r '.data.certificate' > intermediate.cert.pem

   kubectl cp intermediate.cert.pem vault/vault-0:/tmp
   ```

1. After signing the CSR and the root CA returns a certificate, it can be imported back into Vault.

   ```sh
   kubectl exec vault-0 -it -n vault -- \
      vault write pki_int/intermediate/set-signed certificate=@/tmp/intermediate.cert.pem
   ```

## Create a role

A role is a logical name that maps to a policy used to generate those credentials. It allows [configuration parameters](https://developer.hashicorp.com/vault/api-docs/secret/pki#create-update-role) to control certificate common names, alternate names, the key uses that they are valid for, and more.

1. Create a role named `example-dot-com` which allows subdomains, and specify the default issuer ref ID as the value of `issuer_ref`.

   ```sh
   kubectl exec vault-0 -it -n vault -- sh

   vault write pki_int/roles/example-dot-com \
        issuer_ref="$(vault read -field=default pki_int/config/issuers)" \
        allowed_domains="example.com,svc.cluster.local,svc" \
        allow_subdomains=true \
        allow_wildcard_certificates=true \
        max_ttl="720h"
   ```

## Request a certificate

Execute the following command to request a new certificate for the `test.example.com` domain based on the `example-dot-com` role.

```sh
vault write pki_int/issue/example-dot-com common_name="test.example.com" ttl="24h"
```

The response has the PEM-encoded private key, key type and certificate serial number.

> [!NOTE]
> Keep certificate lifetimes short to align with Vault's philosophy of short-lived secrets.

## Automate leaf certificate renewal

### Cert-manager integration

In order to request signing of certificates by `Vault`, the issuer must be able to properly authenticate against it. `cert-manager` provides multiple approaches to authenticating to Vault which are detailed below.

Vault auth type cert-manager issuer configuration

- A. Authenticating with a Vault `AppRole`
- B. Authenticating with a Vault `Token`
- C. Authenticating with Kubernetes Service Accounts > Use `JWT/OIDC` Auth
- C. Authenticating with Kubernetes Service Accounts > Use `Kubernetes` Auth

#### Vault Authentication Method: Use Kubernetes Auth

The Kubernetes auth method should be used when:

- Your Vault server is running inside the Kubernetes cluster.
- Or, your Kubernetes' cluster OIDC discovery endpoint is not reachable from the Vault server, but Vault can reach the Kubernetes API server.

1. Enable the `kubernetes` secrets engine at the `kubernetes` path and configure it to use the kubernetes cluster.

   ```sh
   kubectl config view --minify --flatten -ojson \
     | jq -r '.clusters[].cluster."certificate-authority-data"' \
     | base64 -d > /tmp/cacrt

   kubectl cp /tmp/cacrt vault/vault-0:/tmp

   kubectl -n vault exec -ti vault-0 -- \
      vault auth enable -path=kubernetes kubernetes

   kubectl -n vault exec -ti vault-0 -- \
      vault write auth/kubernetes/config \
        kubernetes_host=https://kubernetes.default:443 \
        kubernetes_ca_cert=@/tmp/cacrt
   ```

1. Create a Kubernetes Service Account and a matching `Vault` role.

   ```sh
   kubectl create serviceaccount -n cert-manager vault-issuer
   ```

1. Then add an RBAC Role so that cert-manager can get tokens for the `ServiceAccount`.

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: vault-issuer
     namespace: cert-manager
   rules:
     - apiGroups: [""]
       resources: ["serviceaccounts/token"]
       resourceNames: ["vault-issuer"]
       verbs: ["create"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: vault-issuer
     namespace: cert-manager
   subjects:
     - kind: ServiceAccount
       name: cert-manager
       namespace: cert-manager
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: vault-issuer
   ```

1. Create a policy named `pki_int` that enables read access to the PKI secrets engine paths.

   ```sh
   vault policy write pki_int - <<EOF
   path "pki_int*"                        { capabilities = ["read", "list"] }
   path "pki_int/sign/example-dot-com"    { capabilities = ["create", "update"] }
   path "pki_int/issue/example-dot-com"   { capabilities = ["create"] }
   EOF
   ```

1. Then, create the Vault role.

   ```sh
   vault write auth/kubernetes/role/vault-issuer-role \
       bound_service_account_names=vault-issuer \
       bound_service_account_namespaces=cert-manager \
       audience="vault://vault-issuer" \
       policies=pki_int \
       ttl=1m
   ```

   It is recommended to use a different Vault role each per Issuer or ClusterIssuer. The `audience` allows you to restrict the Vault role to a single Issuer or ClusterIssuer. The syntax is the following:

   - **Issuer**: "vault://namespace/issuer-name"
   - **ClusterIssuer**: vault://cluster-issuer-name"

1. Finally, you can create your Issuer:

   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: vault-issuer
     namespace: cert-manager
   spec:
     vault:
       path: pki_int/sign/example-dot-com
       server: https://vault-active.vault:8200
       # base64 encoded CA Bundle PEM file
       caBundle: LS0tL...
       auth:
         kubernetes:
           role: vault-issuer-role
           mountPath: /v1/auth/kubernetes
           serviceAccountRef:
             name: vault-issuer
   ```

   Replace `ClusterIssuer` with `Issuer` if that is what you want to deploy.

#### Verifying the issuer Deployment

The Vault issuer tests your Vault instance by querying the `v1/sys/health` endpoint, to ensure your Vault instance is unsealed and initialized before requesting certificates. The result of that query will populate the `STATUS` column.

```sh
kubectl get clusterissuers vault-issuer -o wide
```

Certificates are now ready to be requested by using the Vault issuer named `vault-issuer` within the cluster.

### Vault agent templates

## Resources

- [Build your own certificate authority (CA)](https://developer.hashicorp.com/vault/tutorials/pki/pki-engine)

- [PKI design white paper](https://www.hashicorp.com/en/vault-pki)

- [PKI secrets engine capabilities](https://developer.hashicorp.com/vault/docs/secrets/pki)

- [Vault PKI tutorials](https://developer.hashicorp.com/vault/docs/secrets/pki#tutorial)

- Reference the [PKI secrets engine API](https://developer.hashicorp.com/vault/api-docs/secret/pki)

- Vault Agent [support for automatically renewing requested certificates](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template#certificates)

- [Cert Manager Vault issuer configuration](https://cert-manager.io/docs/configuration/vault/)
