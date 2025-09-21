# Kubernetes user management

This document provides guidance on managing users in Kubernetes, including authentication and authorization using Role-Based Access Control (RBAC). It covers creating user certificates, configuring kubeconfig files, and applying RBAC policies.

## Authentication

To authenticate users, generate certificates and create a kubeconfig file. Follow these steps:

1. Export user variables:

    ```bash
    export USER="jane"
    export GROUP="finops:masters"
    export KUBE_API="https://apiserver.testbed.moh:6443"
    export CLUSTER_NAME="cluster.local"
    # ROLE can be "admin", "edit" or "view"
    export ROLE="admin"
    export NS="finance"
    ```

1. Create a private key:

    ```bash
    openssl genrsa -out ${USER}.key 2048
    ```

1. Generate a Certificate Signing Request (`CSR`) using the private key, including the user name and group:

    ```bash
    openssl req -new -key ${USER}.key -subj "/CN=${USER}/O=${GROUP}" -out ${USER}.csr
    ```

1. Create a `CertificateSigningRequest` object and submit it:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: ${USER}
    spec:
      groups:
        - system:authenticated
      usages:
        - digital signature
        - key encipherment
        - client auth
      request: $(base64 -w0 ${USER}.csr)
      signerName: kubernetes.io/kube-apiserver-client
      expirationSeconds: 86400
    EOF
    ```

1. Identify the new request and approve it:

    ```bash
    kubectl get csr ${USER}
    kubectl certificate approve ${USER}
    ```

1. Extract the signed certificate (Kubernetes signs the certificate using the CA key pair):

    ```bash
    kubectl get csr ${USER} -o jsonpath='{.status.certificate}'| base64 -d > ${USER}.crt
    ```

1. Create a ServiceAccount for the user (for accessing the Kubernetes Dashboard):

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: ${USER}
      namespace: kube-system
    EOF
    ```

1. Create a Secret for the ServiceAccount token:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Secret
    metadata:
      name: ${USER}
      namespace: kube-system
      annotations:
        kubernetes.io/service-account.name: ${USER}
    type: kubernetes.io/service-account-token
    EOF
    ```

1. Create the kubeconfig file:

    ```bash
    cat <<EOF > config
    apiVersion: v1
    kind: Config
    current-context: ${USER}@${CLUSTER_NAME}
    clusters:
      - name: ${CLUSTER_NAME}
        cluster:
          certificate-authority-data: $(kubectl get cm kube-root-ca.crt -o jsonpath='{.data.ca\.crt}' | base64 -w0)
          server: ${KUBE_API}
    contexts:
      - name: ${USER}@${CLUSTER_NAME}
        context:
          user: ${USER}
          cluster: ${CLUSTER_NAME}
          namespace: ${NS}
    users:
      - name: ${USER}
        user:
          client-certificate-data: $(kubectl get csr ${USER} -o jsonpath='{.status.certificate}')
          client-key-data: $(base64 -w0 ${USER}.key)
          token: $(kubectl get secret ${USER} -n kube-system -o jsonpath={".data.token"} | base64 -d)
    EOF
    ```

1. View and test the kubeconfig:

    ```bash
    kubectl --kubeconfig=config config view
    kubectl --kubeconfig=config api-resources
    ```

## Authorization

Role-based access control (RBAC) is a method of regulating access to computer or network resources based on the roles of individual users within your organization.

### API objects

- `Roles` and `RoleBindings` apply to namespaced resources. A `Role` is scoped to a specific namespace. List namespaced resources with:

    ```bash
    kubectl api-resources --namespaced=true
    ```

- `ClusterRoles` and `ClusterRoleBindings` apply to cluster-scoped resources. List cluster-scoped resources with:

    ```bash
    kubectl api-resources --namespaced=false
    ```

For fine-grained access control examples, refer to the [official Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) documentation.

### Default Roles

There are some default clusterroles in kubernetes that you can use. They include super-user roles (`cluster-admin`), roles intended to be granted cluster-wide using `ClusterRoleBindings`, and roles intended to be granted within particular namespaces using `RoleBindings` (`admin`, `edit`, `view`).

- User-based access (via RoleBinding):

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: ${USER}-${NS}-${ROLE}
      namespace: ${NS}
    subjects:
    - kind: User
      name: ${USER}
      apiGroup: rbac.authorization.k8s.io
    - kind: ServiceAccount
      name: ${USER}
      namespace: kube-system
    roleRef:
      kind: ClusterRole
      name: ${ROLE}
      apiGroup: rbac.authorization.k8s.io
    EOF
    ```

- Group based access (via RoleBinding):

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: ${GROUP}-${NS}-${ROLE}
      namespace: ${NS}
    subjects:
    - kind: Group
      name: ${GROUP}
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: ${ROLE}
      apiGroup: rbac.authorization.k8s.io
    EOF
    ```

### Aggregation

Aggregate multiple `ClusterRoles` into one combined `ClusterRole` to extend default roles, such as adding permissions for custom resources.

Example aggregated `ClusterRole`:

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: aggregate-cron-tabs-edit
  labels:
    # Add these permissions to the "admin" and "edit" default roles.
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-view
  labels:
    # Add these permissions to the "view" default role.
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch"]
```

### Access chack

Verify a user's access to a resource:

```bash
kubectl auth can-i create cm --as jane --namespace=frontend
kubectl auth can-i create cm/redis-config --as-group cluster:masters --as jane --namespace=frontend
```

## Linux user for kube client VM

To manage Kubernetes access on a client VM:

1. Create a group for kubectl users:

    ```bash
    groupadd kubeusers
    ```

1. Create the user:

    ```bash
    adduser -m -g kubeusers -s /usr/bin/zsh -c "jane user for accessing kubectl" jane
    passwd jane
    chage -d 0 jane
    ```

1. Configure the user for `kubectl`:

    ```bash
    mkdir /home/jane/.kube
    chgrp kubeusers /usr/bin/kubectl
    chmod 750 /usr/bin/kubectl
    ```

## Resources

- [Certificates and Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)

- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)