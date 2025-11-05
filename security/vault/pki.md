# PKI secrets engine

The PKI secrets engine generates dynamic X.509 certificates. With this secrets engine, services can get certificates without going through the usual manual process of generating a private key and CSR, submitting to a CA, and waiting for a verification and signing process to complete. Vault's built-in authentication and authorization mechanisms provide the verification functionality.

## Prerequisites

- A Vault environment. Refer to the [Vault install guide](./README.md) to install Vault.
- `jq` and `openssl` installed.

### Policy requirements

To perform all tasks demonstrated in this tutorial, your policy must include the following capabilities:

```hcl
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
        ttl=87600h > root_2025_ca.crt
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

1. Use the pki issue command to handle the intermediate CA generation process.

   ```sh
   vault pki issue \
         --issuer_name=example-dot-com-intermediate \
         /pki/issuer/a13b7db8-58f2-d926-64fc-bd39c9e6950c \
         /pki_int/ \
         common_name="example.com Intermediate Authority" \
         o="example" \
         ou="education" \
         key_type="rsa" \
         key_bits="4096" \
         max_depth_len=1 \
         permitted_dns_domains="test.example.com" \
         ttl="43800h"
   ```

## Create a role

A role is a logical name that maps to a policy used to generate those credentials. It allows [configuration parameters](https://developer.hashicorp.com/vault/api-docs/secret/pki#create-update-role) to control certificate common names, alternate names, the key uses that they are valid for, and more.

1. Create a role named `example-dot-com` which allows subdomains, and specify the default issuer ref ID as the value of `issuer_ref`.

   ```sh
   vault write pki_int/roles/example-dot-com \
        issuer_ref="$(vault read -field=default pki_int/config/issuers)" \
        allowed_domains="example.com" \
        allow_subdomains=true \
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

### Vault agent templates

### Cert-manager integration

## Resources

- [Build your own certificate authority (CA)](https://developer.hashicorp.com/vault/tutorials/pki/pki-engine)
- [PKI design white paper](https://www.hashicorp.com/en/vault-pki)
- [PKI secrets engine capabilities](https://developer.hashicorp.com/vault/docs/secrets/pki)
- [Vault PKI tutorials](https://developer.hashicorp.com/vault/docs/secrets/pki#tutorial)
- Reference the [PKI secrets engine API](https://developer.hashicorp.com/vault/api-docs/secret/pki)
- Vault Agent [support for automatically renewing requested certificates](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template#certificates)
