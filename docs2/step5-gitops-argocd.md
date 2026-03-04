# From Imperative Control to Declarative GitOps

In previous steps, autoscaling and rollouts were managed imperatively, using direct commands to the cluster to modify replicas or update Pods.

With **GitOps**, the entire desired state of the cluster — Deployments, ConfigMaps, HPAs, Services — is defined in YAML files within a Git repository. A controller like **ArgoCD** continuously compares the desired state with the actual state and automatically applies the necessary changes.

**This approach ensures idempotent, reproducible, and secure updates, with automatic rollbacks, self-healing, and continuous synchronization, eliminating the need for manual intervention on the cluster.**

**The Pillars of GitOps:**

1. **Git as the Single Source of Truth**: The terminal (`kubectl apply`) is no longer used for manual changes. If you need 5 replicas instead of 2, you update the file on GitHub.
2. **Desired State vs. Actual State**: You describe how you want the cluster to look (Desired State). The GitOps tool monitors the real-time status of the cluster (Actual State).
3. **Automated Reconciliation**: If the two states diverge, the tool automatically corrects the cluster to match Git.

## What is ArgoCD?

**Argo CD** is a **declarative, GitOps continuous delivery tool for Kubernetes**. It automates the deployment and lifecycle management of applications, ensuring consistency and reducing manual errors. As a CNCF graduated project, it guarantees security and reliability at scale.

* **Monitors GitHub**: Checks the repository every few seconds for changes.
* **Monitors the Cluster**: Tracks Pods and Services on K3s.
* **Compares & Synchronizes**: If a difference is detected (e.g., a port change or a new comment in Git), it pulls the updated manifests and applies them to the cluster.

# GitOps with ArgoCD: Setup and Configuration

## 1. Installing ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

### Optimization: Pinning ArgoCD to the Master VM

To prevent the Raspberry Pi (Worker Node) from being overwhelmed by management overhead, I forced all ArgoCD components to run exclusively on the VM (**k3s-master** node) using a `nodeSelector`.

```bash
# Example for the main server; repeated for all ArgoCD components (redis, repo-server, etc.)
kubectl patch deployment argocd-server -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'

```

### 2. Accessing the Dashboard

I exposed the service via **NodePort** and configured **Caddy** as a Reverse Proxy to provide secure HTTPS access.

**Caddyfile configuration:**

```text
argo.enrisox-devops.it {
    reverse_proxy https://192.168.1.X:30260 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}

```

**Retrieve the initial admin password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

```

## 3. Repository Connection

I created a private GitHub repository containing the application manifests:

* `apps/portfolio/deployment.yaml` (Deployment & Service)
* `apps/portfolio/configmap.yaml` (Portfolio HTML content)

Using a **GitHub Personal Access Token (PAT)**, I connected the repository to ArgoCD under **Settings -> Repositories**.

## 4. Creating the ArgoCD Application

Through the ArgoCD dashboard, I defined a new application with the following **Sync Policies**:

* **Automated Sync**: Automatically applies differences between Git and the cluster.
* **Prune**: If a YAML is deleted from Git, ArgoCD removes the resource from the cluster.
* **Self-Heal**: If someone manually modifies the cluster (drift), ArgoCD reverts it to the state declared in Git.

## Automatic Update Workflow (Git → ArgoCD → Kubernetes)

With **Automatic Sync** enabled, the deployment lifecycle follows a strictly declarative flow:

1. **Modify**: Manifests or ConfigMaps are updated in the local Git repo.
2. **Push**: `git commit` + `git push` sends changes to GitHub.
3. **Detect**: ArgoCD detects the commit and identifies the "OutOfSync" status.
4. **Reconcile**: ArgoCD pulls the new manifests and applies them.
5. **Rollout**: Kubernetes performs a **Zero-Downtime RollingUpdate**, respecting `maxSurge` and `maxUnavailable` parameters while checking Health Probes.
6. **Validate**: The ArgoCD dashboard confirms the status as `Synced` and `Healthy`.

