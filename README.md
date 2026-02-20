# GitOps Deployment with ArgoCD (Staging → Production)

## 1. Project Overview

This project demonstrates a beginner-friendly yet production-oriented GitOps workflow using ArgoCD to manage application deployments across `staging` and `production` Kubernetes namespaces. The core principle is to use Git as the single source of truth for declarative infrastructure and application configurations, enabling automated deployments, drift detection, and seamless rollbacks.

## 2. Why GitOps?

GitOps is an operational framework that takes DevOps best practices used for application development, like version control, collaboration, compliance, and CI/CD, and applies them to infrastructure automation. Key benefits include:

*   **Increased Productivity**: Automates deployments and updates, reducing manual effort.
*   **Enhanced Reliability**: Ensures consistent and reproducible environments.
*   **Faster Time to Market**: Accelerates software delivery through automated pipelines.
*   **Improved Security**: Git provides an audit trail for all changes, enhancing compliance and traceability.
*   **Easier Rollbacks**: Quickly revert to previous stable states by simply reverting Git commits.

## 3. Architecture Diagram (ASCII)

```
+-----------------+
|    GitHub Repo    |
| (Single Source  |
|    of Truth)    |
+--------+--------+
         |
         | (Git Push)
         v
+--------+--------+
|    ArgoCD       |
| (Reconciliation |
|    Engine)      |
+--------+--------+
         |          |
         | (Sync)   | (Sync)
         v          v
+-----------------+  +-----------------+
| Kubernetes      |  | Kubernetes      |
| (Staging        |  | (Production     |
|    Namespace)   |  |    Namespace)   |
+-----------------+  +-----------------+
```

## 4. Tools Used

*   **Kubernetes**: k3s (Lightweight, single-node cluster)
*   **Container Runtime**: Docker
*   **GitOps Tool**: ArgoCD
*   **Configuration Management**: Kustomize
*   **Version Control**: Git / GitHub
*   **CLI Tools**: `kubectl`, `argocd`, `gh`

## 5. Setup Instructions

This project assumes an AlmaLinux VM on Azure with SSH access and `gh` CLI configured. The following steps outline the environment setup and ArgoCD deployment.

1.  **SSH into Azure VM**:
    ```bash
    ssh -i raajkumar_key azureuser@<your-vm-ip>
    ```

2.  **Install Prerequisites (Docker, k3s, kubectl, ArgoCD CLI)**:
    ```bash
    # Install Docker
    sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sudo dnf install -y docker-ce docker-ce-cli containerd.io
    sudo systemctl enable --now docker

    # Install k3s
    curl -sfL https://get.k3s.io | sh -
    sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    mkdir -p ~/.kube
    sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
    sudo chown azureuser:azureuser ~/.kube/config

    # Install kubectl (if not already installed by k3s)
    sudo dnf install -y kubectl || (curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl)

    # Install ArgoCD CLI
    curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
    sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
    rm argocd-linux-amd64
    ```

3.  **Deploy ArgoCD to Kubernetes**:
    ```bash
    kubectl create namespace argocd || true
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
    ```

4.  **Access ArgoCD UI and CLI**:
    *   Retrieve the initial admin password:
        ```bash
        kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
        ```
    *   Port-forward the ArgoCD server to access the UI locally (run this in a separate terminal):
        ```bash
        kubectl port-forward service/argocd-server -n argocd 8080:443
        ```
    *   Log in via CLI (replace `<your-password>` with the retrieved password):
        ```bash
        argocd login localhost:8080 --username admin --password <your-password> --insecure
        ```

5.  **Clone this Repository and Configure ArgoCD Application**:
    ```bash
    git clone https://github.com/RajkumarR1206/-gitops-argocd-demo.git
    cd -gitops-argocd-demo

    # Apply the ArgoCD Application manifest
    kubectl apply -f argocd-app.yaml
    ```

## 6. Demo Walkthrough

This project includes three mandatory demonstrations of GitOps capabilities:

### DEMO 1: GitOps Auto Deployment

*   **Scenario**: Update the Nginx application image in `app/deployment.yaml` from `nginx:1.25` to `nginx:1.26`.
*   **Action**: Commit and push the change to the `main` branch.
*   **Observation**: ArgoCD automatically detects the change, syncs the application, and updates the deployment in the `staging` namespace. Verify the new image version in the Kubernetes cluster.

### DEMO 2: Drift Detection & Self-Heal

*   **Scenario**: Manually scale down the `staging-demo-app` deployment in the Kubernetes cluster to 0 replicas.
*   **Action**: `kubectl scale deployment staging-demo-app -n staging --replicas=0`
*   **Observation**: ArgoCD detects the manual change (drift) and, due to `selfHeal: true` in the `Application` manifest, automatically reverts the deployment back to 1 replica, restoring the desired state.

### DEMO 3: Rollback (MOST IMPORTANT)

*   **Scenario**: Introduce a bad Nginx image (e.g., `nginx:nonexistent`) in `app/deployment.yaml`.
*   **Action**: Commit and push the bad configuration. Observe the application failing (e.g., `ImagePullBackOff` status).
*   **Action**: Perform a `git revert` on the commit that introduced the bad image and push the revert commit.
*   **Observation**: ArgoCD automatically detects the revert and rolls back the application to the previous stable version, restoring functionality.

## 7. Key DevOps/SRE Concepts Demonstrated

*   **Declarative Configuration**: All infrastructure and application states are defined in Git.
*   **Single Source of Truth**: Git repository serves as the authoritative source for desired state.
*   **Immutability**: Deployments are treated as immutable artifacts; changes are made via Git, not direct cluster modifications.
*   **Automation**: ArgoCD automates the synchronization between Git and the cluster.
*   **Observability**: ArgoCD provides visibility into application health and sync status.
*   **Reliability & Recoverability**: Self-healing and easy rollback mechanisms ensure application stability.

## 8. Interview Talking Points

*   Discuss the benefits of GitOps over traditional CI/CD approaches for deployment and management.
*   Explain how ArgoCD acts as a reconciliation engine, continuously monitoring Git and the cluster.
*   Detail the role of Kustomize in managing environment-specific configurations (staging vs. production).
*   Describe the self-healing mechanism and its importance in maintaining desired state.
*   Highlight the rollback process and how Git-based rollbacks are more reliable and auditable.
*   Emphasize the security benefits of GitOps, including audit trails and reduced direct access to clusters.

## 9. Resume Bullet

*   **Engineered and implemented a robust GitOps workflow using ArgoCD and Kustomize for automated, self-healing deployments across staging and production Kubernetes environments, significantly enhancing reliability and accelerating delivery.**
