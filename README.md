# HA Vault cluster with Raft and TLS

This deployment guide covers the steps required to install and configure a HA HashiCorp Vault cluster with integrated storage and TLS enabled in Kubernetes.

## Install the Vault Helm chart

1. Add the HashiCorp Helm repository.

   ```bash
   helm repo add hashicorp https://helm.releases.hashicorp.com
   ```

2. Update all the repositories to ensure `helm` is aware of the latest versions.

   ```bash
   helm repo update
   ```

3. To verify, search repositories for vault in charts.

   ```bash
   helm search repo hashicorp/vault
   ```

## Create Vault certificate using kubernetes CA

1. Export the working directory location and the naming variables.

   ```bash
   # SERVICE is the name of the Vault service in kubernetes.
   # It does not have to match the actual running service, though it may help for consistency.
   export SERVICE=vault-internal

   # NAMESPACE where the Vault service is running.
   export NAMESPACE=vault

   # SECRET_NAME to create in the kubernetes secrets store.
   export SECRET_NAME=vault-server-tls

   # WORKDIR is a temporary working directory.
   export WORKDIR=tmp

   # CSR_NAME will be the name of our certificate signing request as seen by kubernetes.
   export CSR_NAME=vault-csr

   # K8S_CLUSTER_NAME is the domain name of the kubernetes cluster
   export K8S_CLUSTER_NAME="cluster.local"
   ```

1. Create a working directory.

   ```bash
   mkdir -p $WORKDIR
   ```

1. Generate the private key for Kubernetes to sign.

   ```bash
   openssl genrsa -out ${WORKDIR}/vault.key 2048
   ```

### Create the Certificate Signing Request (CSR)

1. Create the CSR configuration file.

   ```bash
   cat <<EOF >${WORKDIR}/csr.conf
   [req]
   default_bits = 2048
   prompt = no
   encrypt_key = yes
   default_md = sha256
   distinguished_name = kubelet_serving
   req_extensions = v3_req
   [ kubelet_serving ]
   O = system:nodes
   CN = system:node:*.${NAMESPACE}.svc.${K8S_CLUSTER_NAME}
   [ v3_req ]
   basicConstraints = CA:FALSE
   keyUsage = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
   extendedKeyUsage = serverAuth, clientAuth
   subjectAltName = @alt_names
   [alt_names]
   DNS.1 = *.${SERVICE}
   DNS.2 = *.${SERVICE}.${NAMESPACE}
   DNS.3 = *.${SERVICE}.${NAMESPACE}.svc
   DNS.4 = *.${SERVICE}.${NAMESPACE}.svc.${K8S_CLUSTER_NAME}
   DNS.5 = vault-active
   DNS.6 = vault-active.${NAMESPACE}
   DNS.7 = vault-active.${NAMESPACE}.svc
   DNS.8 = vault-active.${NAMESPACE}.svc.${K8S_CLUSTER_NAME}
   DNS.9 = *.${NAMESPACE}
   IP.1 = 127.0.0.1
   EOF
   ```

1. Generate the CSR.

   ```bash
   openssl req -new \
               -key ${WORKDIR}/vault.key \
               -out ${WORKDIR}/server.csr \
               -config ${WORKDIR}/csr.conf
   ```

### Issue the certificate

1. Create the CSR YAML file to send it to Kubernetes.

   ```bash
   cat <<EOF >${WORKDIR}/csr.yaml
   apiVersion: certificates.k8s.io/v1
   kind: CertificateSigningRequest
   metadata:
     name: ${CSR_NAME}
   spec:
     signerName: kubernetes.io/kubelet-serving
     groups:
     - system:authenticated
     request: $(base64 ${WORKDIR}/server.csr | tr -d '\n')
     signerName: kubernetes.io/kubelet-serving
     usages:
     - digital signature
     - key encipherment
     - server auth
   EOF
   ```

1. Send the CSR to Kubernetes.

   ```bash
   kubectl create -f ${WORKDIR}/csr.yaml
   ```

1. Approve the CSR in Kubernetes.

   ```bash
   kubectl certificate approve ${CSR_NAME}
   ```

1. Verify that the certificate was approved and issued.

   ```bash
   kubectl get csr ${CSR_NAME}
   ```

## Store key, cert, and kubernetes CA into kubernetes secrets store

1. Retrieve the certificate.

   ```bash
   kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}' | openssl base64 -d -A -out ${WORKDIR}/vault.crt
   ```

1. Retrieve Kubernetes CA certificate.

   ```bash
   kubectl get cm -o jsonpath='{.items[0].data.ca\.crt}'> ${WORKDIR}/vault.ca

   # or

   kubectl config view \
     --raw \
     --minify \
     --flatten \
     -o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
   | base64 -d > ${WORKDIR}/vault.ca
   ```

1. Create the Kubernetes namespace.

   ```bash
   kubectl create namespace ${NAMESPACE}
   ```

1. Store the key, cert, and Kubernetes CA into Kubernetes secrets.

   ```bash
   kubectl create secret generic ${SECRET_NAME} \
       --namespace ${NAMESPACE} \
       --from-file=vault.key=${WORKDIR}/vault.key \
       --from-file=vault.crt=${WORKDIR}/vault.crt \
       --from-file=vault.ca=${WORKDIR}/vault.ca
   ```

## Deploy the vault cluster via Helm with overrides

1. Create the vaules.yaml file from values file provided in this repository.

1. Deploy the cluster.

   ```bash
   helm install vault hashicorp/vault -n vault -f values.yaml
   ```

1. Display the pods in the namespace that you created for vault.

   ```bash
   kubectl -n $VAULT_K8S_NAMESPACE get pods
   ```

### Initialize and Unseal `vault-0`

Wait for a few minutes and initialize and unseal `vault-0` pod.

1. Initialize `vault-0` with 5 key share and 3 key threshold.

   > [!CAUTION]
   > Do not run an unsealed Vault in production with a single key share and a single key threshold.

   ```bash
   kubectl exec -n $NAMESPACE vault-0 -- vault operator init \
       -key-shares=5 \
       -key-threshold=3 \
       -format=json > ${WORKDIR}/cluster-keys.json
   ```

   > [!NOTE]
   > The `operator init` command generates a root key that it disassembles into key shares `-key-shares=5` and then sets the number of key shares required to unseal Vault `-key-threshold=1`. These key shares are written to the output as unseal keys in JSON format `-format=json`. Here the output is redirected to a file named `cluster-keys.json`.

   > [!IMPORTANT]
   > Store Unseal keys and Root key SECURELY

1. Create variables to capture the Vault unseal key.

   ```bash
   VAULT_UNSEAL_KEY_1=$(jq -r ".unseal_keys_b64[0]" ${WORKDIR}/    cluster-keys.json)
   VAULT_UNSEAL_KEY_2=$(jq -r ".unseal_keys_b64[1]" ${WORKDIR}/    cluster-keys.json)
   VAULT_UNSEAL_KEY_3=$(jq -r ".unseal_keys_b64[2]" ${WORKDIR}/    cluster-keys.json)
   ```

   After initialization, Vault is configured to know where and how to access the storage, but does not know how to decrypt any of it. `Unsealing` is the process of constructing the root key necessary to read the decryption key to decrypt the data, allowing access to the Vault.

1. Unseal Vault running on the vault-0 pod until it reaches the key threshold (in this case 3 times).

   ```bash
   kubectl -n $NAMESPACE exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_1
   kubectl -n $NAMESPACE exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_2
   kubectl -n $NAMESPACE exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_3
   ```

### Join `vault-1` and `vault-2` pods to Raft cluster

1. Start an interactive shell session on the `vault-1` pod

   ```bash
   kubectl exec -n $NAMESPACE -it vault-1 -- /bin/sh
   ```

   **Your system prompt is replaced with a new prompt / $.**

1. Join the `vault-1` pod to the Raft cluster.

   ```sh
   vault operator raft join -address=https://vault-1.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-server-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-server-tls/vault.crt)" -leader-client-key="$(cat /vault/userconfig/vault-server-tls/vault.key)" https://vault-0.vault-internal:8200
   ```

1. Exit the `vault-1` pod

   ```
   exit
   ```

1. Unseal `vault-1` until it reaches the key-threshold (in this case 3 times)

   ```bash
   kubectl -n $NAMESPACE exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_1
   kubectl -n $NAMESPACE exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_2
   kubectl -n $NAMESPACE exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_3
   ```

1. Start an interactive shell session on the `vault-2` pod

   ```bash
   kubectl exec -n $NAMESPACE -it vault-2 -- /bin/sh
   ```

   **Your system prompt is replaced with a new prompt / $.**

1. Join the `vault-2` pod to the Raft cluster.

   ```sh
   vault operator raft join -address=https://vault-2.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-server-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-server-tls/vault.crt)" -leader-client-key="$(cat /vault/userconfig/vault-server-tls/vault.key)" https://vault-0.vault-internal:8200
   ```

1. Exit the `vault-2` pod

   ```sh
   exit
   ```

1. Unseal `vault-2` until it reaches the key-threshold (in this case 3 times)

   ```bash
   kubectl -n $NAMESPACE exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY_1
   kubectl -n $NAMESPACE exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY_2
   kubectl -n $NAMESPACE exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY_3
   ```

### Verify the cluster is initialized and unsealed

Login to vault and confirm everything is working

1. Export the cluster `root token`

   ```bash
   export CLUSTER_ROOT_TOKEN=$(cat ${WORKDIR}/cluster-keys.json | jq -r ".root_token")
   ```

1. Login to `vault-0` with the `root token`

   ```bash
   kubectl -n vault exec vault-0 -- vault login $CLUSTER_ROOT_TOKEN  # Store login tokens
   ```

1. List the raft peers

   ```bash
   kubectl -n vault exec vault-0 -- vault operator raft list-peers
   ```

1. Print HA status

   ```bash
   kubectl -n vault exec vault-0 -- vault status
   ```

## Test

### Create a secret

1. Start an interactive shell session on the vault-0 pod.

   ```bash
   kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-0 -- /bin/sh
   ```

   **Your system prompt is replaced with a new prompt / $.**

1. Enable the kv-v2 secrets engine.

   ```sh
   vault secrets enable -path=secret kv-v2
   ```

1. Create a secret at the path `secret/tls/apitest` with a username and a password.

   ```sh
   vault kv put secret/tls/apitest username="apiuser" password="supersecret"
   ```

1. Verify that the secret is defined at the path `secret/tls/apitest`

   ```sh
   vault kv get secret/tls/apitest
   ```

1. Exit the `vault-0` pod.

   ```sh
   exit
   ```

### Expose the vault service and retrieve the secret via the API

The Helm chart defined a Kubernetes service named vault that forwards requests to its endpoints (i.e. The pods named vault-0, vault-1, and vault-2).

1. Confirm the Vault service configuration

   ```bash
   kubectl -n NAMESPACE get service vault
   ```

2. In `another terminal`, port forward the vault service.

   ```bash
   kubectl -n vault port-forward service/vault 8200:8200
   ```

3. In the original terminal, perform a `HTTPS` curl request to retrieve the secret you created in the previous section.

   ```bash
   curl --cacert $TMPDIR/vault.ca \
      --header "X-Vault-Token: $CLUSTER_ROOT_TOKEN" \
      https://127.0.0.1:8200/v1/secret/data/tls/apitest | jq .data.data
   ```
