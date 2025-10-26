# GitLab ArgoCD Integration

This guide outlines the steps to integrate GitLab with ArgoCD for GitOps workflows. It assumes you have GitLab and ArgoCD installed and accessible. Follow these steps in sequence for a successful setup.

## Prerequisites
- GitLab instance (self-hosted or SaaS).
- ArgoCD installed in a Kubernetes cluster (e.g., via Helm).
- Administrative access to both GitLab and the Kubernetes cluster.

## Step 1: Create GitLab Group and Project
Create a group named `devops` in GitLab. Then, within the `devops` group, create a project named `argocd`. This project will serve as the repository for ArgoCD configurations.

## Step 2: Create Dedicated GitLab User for ArgoCD
Create a user in GitLab named `argocd` (or similar). Assign this user the `Developer` role for the `devops` group to allow read/write access to the repositories.

## Step 3: Handle Self-Signed Certificates (If Applicable)
If your GitLab instance uses a self-signed certificate or a non-valid domain, add GitLab's hostname to the ArgoCD pods using the `values.global.hostAliases` in your ArgoCD Helm values file. This ensures proper name resolution within the pods.

Example YAML snippet:

```yaml
hostAliases:
  - ip: 192.168.1.10
    hostnames:
      - gitlab.example.com
      - repo.example.com
```

> [!NOTE]
> If ArgoCD also uses a self-signed certificate, you may need to configure GitLab to trust it for webhook calls (see Step 5 for disabling SSL verification). Restart the ArgoCD deployment after changes if necessary.

## Step 4: Configure GitLab Webhook Secret in ArgoCD
To enable webhooks from GitLab to ArgoCD, set up a shared secret using one of the following methods:

- Edit the `argocd-secret` Kubernetes secret and add `webhook.gitlab.secret` with your secret value.
  
- In your ArgoCD Helm values file, add:
  ```yaml
  configs:
    secret:
      gitlabSecret: your-secret-value
  ```
  Then upgrade the Helm release.

After configuring the secret, restart the ArgoCD server deployment:
```shell
kubectl -n argocd rollout restart deployment/argocd-server
```

## Step 5: Set Up Webhook in GitLab Project
In the `argocd` project settings under `Webhooks`, add a new webhook with the following details:
- URL: `https://argocd.example.com/api/webhook` (replace with your ArgoCD webhook endpoint).
- Secret Token: The value from Step 4 (e.g., `your-secret-value`).
- Triggers: Select `Push events`.
- SSL Verification: Disable if using self-signed certificates.

> [!IMPORTANT]
> Test the webhook by triggering a push event. It should return `HTTP 200 OK`. If you encounter an "URL is invalid" error, go to GitLab's `Admin area > Settings > Network > Outbound requests` and enable `Allow requests to the local network from webhooks and integrations`.

## Step 6: Generate SSH Key Pair for Authentication
Create an SSH key pair for ArgoCD to authenticate with GitLab:

```shell
ssh-keygen -t ed25519 -C "gitlab.example.com" -f ~/.ssh/gitlab -N ''
```

This generates a private key (`gitlab`) and a public key (`gitlab.pub`) with no passphrase.

## Step 7: Add SSH Public Key to GitLab and Configure ArgoCD Repository
- Add the contents of the public key (`gitlab.pub`) to the `argocd` user's SSH keys in GitLab (under User Settings > SSH Keys).

- In ArgoCD, connect to the GitLab repository using SSH. Use the private key (`gitlab`) to create a repository connection in ArgoCD's UI or via CLI. For example, add the repository with URL `git@gitlab.example.com:devops/argocd.git` and upload the private key.

## Step 8: Create App-of-Apps Application in ArgoCD
Create an ArgoCD `Application` resource to manage other applications (app-of-apps pattern). This watches the `./applications` path in your GitLab repository.

Apply the following YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  source:
    repoURL: git@gitlab.example.com:devops/argocd.git  # Update with your GitLab SSH URL
    path: applications  # Assuming the path is 'applications/' (adjust if needed)
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ApplyOutOfSyncOnly=true
```

> [!NOTE] 
> Replace placeholders like domains and paths with your actual values. Ensure the `repoURL` uses the SSH format for secure access.