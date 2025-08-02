# Create Vault certificates

## Create key & certificate using kubernetes CA

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

# Create a key for Kubernetes to sign

```bash
mkdir -p $WORKDIR

openssl genrsa -out ${WORKDIR}/vault.key 2048

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

openssl req -new \
            -key ${WORKDIR}/vault.key \
            -out ${WORKDIR}/server.csr \
            -config ${WORKDIR}/csr.conf

```

# Create the certificate

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

kubectl create -f ${WORKDIR}/csr.yaml

kubectl certificate approve ${CSR_NAME}

kubectl get csr ${CSR_NAME}
```

# Store key, cert, and kubernetes CA into kubernetes secrets store

```bash
kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}' | openssl base64 -d -A -out ${WORKDIR}/vault.crt

kubectl get cm -o jsonpath='{.items[0].data.ca\.crt}'> ${WORKDIR}/vault.ca

# or

kubectl config view \
  --raw \
  --minify \
  --flatten \
  -o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
| base64 -d > ${WORKDIR}/vault.ca

kubectl create namespace ${NAMESPACE}

kubectl create secret generic ${SECRET_NAME} \
    --namespace ${NAMESPACE} \
    --from-file=vault.key=${WORKDIR}/vault.key \
    --from-file=vault.crt=${WORKDIR}/vault.crt \
    --from-file=vault.ca=${WORKDIR}/vault.ca
```

# Install Vault helm chart

Deploy the cluster

```bash
helm install vault nexus/vault -n vault -f values-vault-0.29.1-minimal.yaml
```

# Initialize and Unseal

Wait for a few minutes and initialize and unseal `vault-0` pod

```bash
# Initialize vault-0 with 5 key share and 3 key threshold
kubectl exec -n $NAMESPACE vault-0 -- vault operator init \
    -key-shares=5 \
    -key-threshold=3 \
    -format=json > ${WORKDIR}/cluster-keys.json

# Create variables to capture the Vault unseal key
VAULT_UNSEAL_KEY_1=$(jq -r ".unseal_keys_b64[0]" ${WORKDIR}/cluster-keys.json)
VAULT_UNSEAL_KEY_2=$(jq -r ".unseal_keys_b64[1]" ${WORKDIR}/cluster-keys.json)
VAULT_UNSEAL_KEY_3=$(jq -r ".unseal_keys_b64[2]" ${WORKDIR}/cluster-keys.json)

# Unseal Vault running on the `vault-0` pod until it reaches the key threshold (in this case 3 times)
kubectl -n $NAMESPACE exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_1
kubectl -n $NAMESPACE exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_2
kubectl -n $NAMESPACE exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_3
```

> [!IMPORTANT]
> Store Unseal keys and Root key SECURELY

# Join `vault-1` and `vault-2` pods to Raft cluster

Join Remaining pods to Raft cluster and unseal them

```bash
# Start an interactive shell session on the `vault-1` pod
kubectl exec -n $NAMESPACE -it vault-1 -- /bin/sh

# Join the `vault-1` pod to the Raft cluster.
/ $ vault operator raft join -address=https://vault-1.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-server-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-server-tls/vault.crt)" -leader-client-key="$(cat /vault/userconfig/vault-server-tls/vault.key)" https://vault-0.vault-internal:8200

# Exit the `vault-1` pod
/ $ exit

# Unseal `vault-1` until it reaches the key-threshold (in this case 3 times)
kubectl -n $NAMESPACE exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_1
kubectl -n $NAMESPACE exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_2
kubectl -n $NAMESPACE exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_3

# Start an interactive shell session on the `vault-2` pod
kubectl exec -n $NAMESPACE -it vault-2 -- /bin/sh

# Join the `vault-2` pod to the Raft cluster.
/ $ vault operator raft join -address=https://vault-2.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-server-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-server-tls/vault.crt)" -leader-client-key="$(cat /vault/userconfig/vault-server-tls/vault.key)" https://vault-0.vault-internal:8200

# Exit the `vault-2` pod
/ $ exit

# Unseal `vault-2` until it reaches the key-threshold (in this case 3 times)
kubectl -n $NAMESPACE exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY_1
kubectl -n $NAMESPACE exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY_2
kubectl -n $NAMESPACE exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY_3
```

# Verify the initialization

```bash
# Export the cluster `root token`
export CLUSTER_ROOT_TOKEN=$(cat ${WORKDIR}/cluster-keys.json | jq -r ".root_token")

# Login to `vault-0` with the `root token`
kubectl -n vault exec vault-0 -- vault login $CLUSTER_ROOT_TOKEN  # Store login tokens

# List the raft peers
kubectl -n vault exec vault-0 -- vault operator raft list-peers

# Print HA status
kubectl -n vault exec vault-0 -- vault status
```
