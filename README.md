# Jenkins CI / ArgoCD CD (GitOps) POC (Local-Only) - Multi-Environment

This Proof of Concept demonstrates a modern, hybrid CI/CD pipeline for Kubernetes (K8s) using Docker Desktop, running entirely locally without an external container registry (like Docker Hub), with **multi-environment support** (dev, qa, prod).

- **Jenkins** handles Continuous Integration (CI): **building the Docker image locally** and updating the manifest file in the GitOps repository.
    
- **ArgoCD** handles Continuous Delivery (CD/GitOps): monitoring the repository and automatically deploying the new image from the local Docker Desktop cache to Kubernetes.

## Multi-Environment Deployment Strategy

This POC supports three deployment scenarios:

- **DEV Environment** → Deployed via **Merge Request (MR)** to `dev` namespace
- **QA Environment** → Deployed via **Git Tag** to `qa` namespace  
- **PROD Environment** → Deployed via **Manual Promotion** to `prod` namespace
    

## Prerequisites

1. **Docker Desktop:** Installed, running, and **Kubernetes enabled** (Settings -> Kubernetes -> Enable Kubernetes). The local K8s cluster uses the host's local Docker image cache, which is essential for this POC.
    
2. **kubectl:** Installed and configured to interact with your Docker Desktop K8s cluster.
    
3. **Git:** Installed for setting up the simulated GitOps repository.
    

## Project Structure

Ensure your local directory matches this structure:

```
.
├── compose.yml              # Defines the Jenkins service
├── app/
│   ├── Dockerfile           # App build file
│   └── Jenkinsfile          # The CI pipeline script (multi-environment support)
└── k8s-config/
    ├── deployment-dev.yaml  # Dev environment K8s manifest
    ├── deployment-qa.yaml   # QA environment K8s manifest
    └── deployment-prod.yaml # Prod environment K8s manifest
```

## Step 1: Create Kubernetes Namespaces

Create the namespaces for each environment:

```bash
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace prod
```

Verify namespaces were created:

```bash
kubectl get namespaces | grep -E "dev|qa|prod"
```

## Step 2: Initialize the GitOps Repository

For this local POC, we simulate the GitOps repository using the local `k8s-config` directory.

1. Open your terminal and navigate to the root of your project.
    
2. Initialize the configuration folder as a Git repository:
    
    ```bash
    cd k8s-config
    git init
    git add .
    git commit -m "Initial K8s multi-environment deployment manifests"
    cd ..
    ```
    

## Step 3: Install ArgoCD on Kubernetes

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
        

## Step 4: Create ArgoCD Applications for Each Environment

Create separate ArgoCD applications to monitor and deploy to each environment namespace.

### 4.1: Create DEV Application

1. Log into the ArgoCD UI (`https://localhost:8080`).
    
2. Click **`+ New App`**.
    
3. **General Settings:**
    
    - **Application Name:** `poc-ci-cd-dev`
        
    - **Project:** `default`
        
    - **Sync Policy:** `Automatic` (with **Prune** and **Self Heal** enabled)
        
4. **Source Settings:**
    
    - **Repository URL:** The local path: `/k8s-config`
        
    - **Revision:** `HEAD`
        
    - **Path:** `deployment-dev.yaml`
        
5. **Destination Settings:**
    
    - **Cluster URL:** `https://kubernetes.default.svc`
        
    - **Namespace:** `dev`
        
6. Click **`CREATE`**.

### 4.2: Create QA Application

Repeat the process with these settings:

- **Application Name:** `poc-ci-cd-qa`
- **Path:** `deployment-qa.yaml`
- **Namespace:** `qa`
- **Sync Policy:** `Automatic` (with **Prune** and **Self Heal** enabled)

### 4.3: Create PROD Application

Repeat the process with these settings:

- **Application Name:** `poc-ci-cd-prod`
- **Path:** `deployment-prod.yaml`
- **Namespace:** `prod`
- **Sync Policy:** `Manual` (recommended for production - requires manual sync approval)

ArgoCD will now deploy the apps using the initial placeholder tag (`poc-web-app:v0.0.0-PLACEHOLDER`) to their respective namespaces.

## Step 5: Configure and Run Jenkins (CI)

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
        

## Step 6: Multi-Environment Deployment Scenarios

The Jenkins pipeline supports three deployment scenarios based on how it's triggered:

### Scenario 1: Deploy to DEV via Merge Request (MR)

**Simulating a Merge Request deployment:**

1. In Jenkins, go to `Simple-App-CI` job.
2. Click **"Build with Parameters"**.
3. Select **DEPLOY_ENVIRONMENT:** `dev`
4. Leave **IMAGE_TAG_OVERRIDE** empty (or specify a custom tag).
5. Click **"Build"**.

**What happens:**
- Jenkins builds the Docker image with tag `build-{BUILD_NUMBER}`
- Updates `k8s-config/deployment-dev.yaml` with the new image tag
- ArgoCD detects the change and automatically syncs to the `dev` namespace

**Verify deployment:**
```bash
kubectl get pods -n dev
kubectl get svc -n dev
```

### Scenario 2: Deploy to QA via Git Tag

**Simulating a Tag-based deployment:**

1. In Jenkins, configure the job to trigger on tags (or manually trigger with tag).
2. For manual simulation:
   - Click **"Build with Parameters"**
   - Select **DEPLOY_ENVIRONMENT:** `qa`
   - In **IMAGE_TAG_OVERRIDE**, enter a tag like `v1.0.0` or `qa-release-1`
   - Click **"Build"**

**What happens:**
- Jenkins builds the Docker image with the specified tag
- Updates `k8s-config/deployment-qa.yaml` with the new image tag
- ArgoCD detects the change and automatically syncs to the `qa` namespace

**Verify deployment:**
```bash
kubectl get pods -n qa
kubectl get svc -n qa
```

### Scenario 3: Deploy to PROD via Manual Promotion

**Manual Production Promotion:**

1. In Jenkins, go to `Simple-App-CI` job.
2. Click **"Build with Parameters"**.
3. Select **DEPLOY_ENVIRONMENT:** `prod`
4. In **IMAGE_TAG_OVERRIDE**, enter the image tag you want to promote (e.g., `build-5` or `v1.0.0`)
5. Click **"Build"**.

**What happens:**
- Jenkins builds/promotes the Docker image
- Updates `k8s-config/deployment-prod.yaml` with the new image tag
- ArgoCD detects the change but **requires manual sync** (if configured with Manual sync policy)
- In ArgoCD UI, go to `poc-ci-cd-prod` application and click **"Sync"** to deploy

**Verify deployment:**
```bash
kubectl get pods -n prod
kubectl get svc -n prod
```

## Step 7: Monitoring Deployments

### Check All Environments

```bash
# View all pods across environments
kubectl get pods --all-namespaces | grep poc-app

# View services
kubectl get svc --all-namespaces | grep poc-app

# Check specific environment
kubectl get all -n dev
kubectl get all -n qa
kubectl get all -n prod
```

### ArgoCD UI

Access ArgoCD UI at `https://localhost:8080` to:
- View application status for each environment
- See sync history
- Manually sync production deployments
- Monitor deployment health

## Summary

This multi-environment setup demonstrates:

- ✅ **DEV**: Automatic deployment on merge requests/builds
- ✅ **QA**: Deployment triggered by tags/releases
- ✅ **PROD**: Manual promotion with approval gates
- ✅ **Namespace isolation**: Each environment in its own namespace
- ✅ **GitOps**: All deployments managed via GitOps manifests
- ✅ **ArgoCD**: Automatic sync for dev/qa, manual sync for prod