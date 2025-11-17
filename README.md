# Jenkins CI / ArgoCD CD (GitOps) POC (Local-Only)

This Proof of Concept demonstrates a modern, hybrid CI/CD pipeline for Kubernetes (K8s) using Docker Desktop, running entirely locally without an external container registry (like Docker Hub).

- **Jenkins** handles Continuous Integration (CI): **building the Docker image locally** and updating the manifest file in the GitOps repository.
    
- **ArgoCD** handles Continuous Delivery (CD/GitOps): monitoring the repository and automatically deploying the new image from the local Docker Desktop cache to Kubernetes.
    

## Prerequisites

1. **Docker Desktop:** Installed, running, and **Kubernetes enabled** (Settings -> Kubernetes -> Enable Kubernetes). The local K8s cluster uses the host's local Docker image cache, which is essential for this POC.
    
2. **kubectl:** Installed and configured to interact with your Docker Desktop K8s cluster.
    
3. **Git:** Installed for setting up the simulated GitOps repository.
    

## Project Structure

Ensure your local directory matches this structure:

```
.
├── docker-compose.yml       # Defines the Jenkins service
├── app/
│   ├── Dockerfile           # App build file
│   └── Jenkinsfile          # The CI pipeline script
└── k8s-config/
    └── deployment-dev.yaml  # The K8s manifest (ArgoCD's Source of Truth)
```

## Step 1: Initialize the GitOps Repository

For this local POC, we simulate the GitOps repository using the local `k8s-config` directory.

1. Open your terminal and navigate to the root of your project.
    
2. Initialize the configuration folder as a Git repository:
    
    ```
    cd k8s-config
    git init
    git add .
    git commit -m "Initial K8s dev deployment manifest"
    cd ..
    ```
    

## Step 2: Install ArgoCD on Kubernetes

ArgoCD is installed directly into your local Kubernetes cluster.

1. **Create Namespace and Deploy ArgoCD:**
    
    ```
    kubectl create namespace argocd
    kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
    ```
    
2. **Access ArgoCD UI:**
    
    - Forward the service port to your local machine:
        
        ```
        kubectl port-forward svc/argocd-server -n argocd 8080:443
        ```
        
    - Access the UI at **`https://localhost:8080`** (you'll need to bypass the browser's warning about the self-signed certificate).
        
3. **Retrieve Initial Password:**
    
    - The username is `admin`. Get the auto-generated password:
        
        ```
        kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
        ```
        

## Step 3: Create the ArgoCD Application

This tells ArgoCD which repository to monitor (`k8s-config`) and where to apply the changes (`default` namespace).

1. Log into the ArgoCD UI (`https://localhost:8080`).
    
2. Click **`+ New App`**.
    
3. **General Settings:**
    
    - **Application Name:** `poc-ci-cd-dev`
        
    - **Project:** `default`
        
    - **Sync Policy:** `Automatic` (with **Prune** and **Self Heal** enabled)
        
4. **Source Settings (Crucial for POC):**
    
    - **Repository URL:** The local path, as exposed to the controller via Docker Desktop: `/k8s-config`
        
    - **Revision:** `HEAD`
        
    - **Path:** `.`
        
5. **Destination Settings:**
    
    - **Cluster URL:** `https://kubernetes.default.svc`
        
    - **Namespace:** `default`
        
6. Click **`CREATE`**.
    

ArgoCD will now deploy the app using the initial placeholder tag (`poc-web-app:v0.0.0-PLACEHOLDER`).

## Step 4: Configure and Run Jenkins (CI)

1. **Start Jenkins Container:**
    
    ```
    # (Inside your project root)
    docker compose up -d
    ```
    
2. **Access Jenkins:** Go to **`http://localhost:8080`**.
    
3. **Unlock and Setup:**
    
    - Find the initial admin password: `docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`
        
    - Log in and complete the setup (install suggested plugins).
        
4. **Create the Pipeline Job:**
    
    - Click **New Item** -> Name it `Simple-App-CI` -> Select **Pipeline**.
        
    - Under the Pipeline section, select **Pipeline script from SCM**.
        
    - **SCM:** `Git`
        
    - **Repository URL:** `/var/jenkins_home/workspace/simple-app` (The internal path within the Jenkins container).
        
    - **Script Path:** `Jenkinsfile`
        
    - Click **Save**.
        

## Step 5: Execute the Pipeline

1. Go to the `Simple-App-CI` job in Jenkins and click **Build Now**.
    
2. **Observe the Flow:**
    
    - **Jenkins (CI):** The job runs, **builds the Docker image locally** (e.g., tagged `build-1`), and then **modifies** the `k8s-config/deployment-dev.yaml` file on the host machine to use the new tag.
        
    - **ArgoCD (CD):** ArgoCD detects the change in the local GitOps directory, marks the application as **`OutOfSync`**, and automatically executes a synchronization (deployment) to update the K8s pod with the new image, pulling it from the shared Docker Desktop image cache.