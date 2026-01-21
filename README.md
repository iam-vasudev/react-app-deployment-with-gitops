## CI/CD Pipeline Workflow

This project uses **Jenkins** for Continuous Integration (CI) and **ArgoCD** for Continuous Deployment (CD) (GitOps).

### 1. Continuous Integration (Jenkins)
Triggered automatically on code commit to the GitHub repository.

1.  **Checkout**: Jenkins checks out the source code from the repository.
2.  **Check Commit Message**: Skips the build if the commit message contains `[skip ci]`, preventing infinite loops when Jenkins commits back to the repo.
3.  **Build App**:
    *   Uses a `node:22-alpine` Docker container.
    *   Run `npm install` to install dependencies.
    *   Runs `npm run build` to generate the production build in `./dist`.
4.  **Docker Build & Push**:
    *   Creates a `Dockerfile` on the fly (Nginx alpine base).
    *   Copies the `./dist` folder to the Nginx html directory.
    *   Builds the Docker image: `iamvasu/react-app:build-<BUILD_NUMBER>`.
    *   Pushes the image to **Docker Hub**.
    *   Tags and pushes as `latest` as well.
5.  **Update Manifest**:
    *   Updates `k8s/deployment.yaml` with the new image tag (`build-<BUILD_NUMBER>`).
    *   Commits this change back to the `main` branch with `[skip ci]` in the message.

### 2. Continuous Deployment (ArgoCD)
ArgoCD monitors the `k8s/` folder in the Git repository.

1.  **Sync**: ArgoCD detects the change in `k8s/deployment.yaml` (updated by Jenkins).
2.  **Apply**: It automatically applies the new manifest to the Kubernetes cluster.
3.  **Deploy**: Kubernetes updates the `Deployment` to pull the new Docker image and restarts the pods.

## How to Run (GitOps & ArgoCD)

### Prerequisites
*   Minikube or a Kubernetes cluster
*   `kubectl` CLI

### Step-by-Step Instructions

1.  **Ensure Minikube/Cluster is Running**
    ```bash
    kubectl get nodes
    ```
    If not started: `minikube start`

2.  **Verify ArgoCD is Running**
    Ensure ArgoCD pods are running in the `argocd` namespace:
    ```bash
    kubectl get pods -n argocd
    ```

3.  **Apply the Application Manifest**
    Register the React app with ArgoCD:
    ```bash
    kubectl apply -f argocd/application.yaml
    ```

4.  **Access ArgoCD UI (Optional)**
    *   **Port Forward**:
        ```bash
        kubectl port-forward svc/argocd-server -n argocd 8080:443
        ```
    *   **Get Password**:
        ```bash
        kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
        ```
    *   **Login**: [https://localhost:8080](https://localhost:8080) (User: `admin`)

5.  **Access the React Application**
    Get the service URL using Minikube:
    ```bash
    minikube service react-app-service
    ```
