# Real-Time DevOps Project: Azure DevOps & GitOps

This project demonstrates a complete, end-to-end CI/CD (Continuous Integration / Continuous Delivery) pipeline for a multi-microservice application. It uses **Azure DevOps** for CI and **Argo CD (GitOps)** for CD, deploying to an **Azure Kubernetes Service (AKS)** cluster.

## 🚀 Project Overview

The goal is to create a fully automated system. When a developer pushes a code change for any microservice, the following pipeline is triggered:

1.  **CI (Azure Pipelines):** Automatically builds *only* the modified service's Docker image.
2.  **Push (ACR):** The new image is pushed to a private Azure Container Registry (ACR).
3.  **Update (GitOps):** The CI pipeline automatically updates the service's Kubernetes (K8s) manifest *in this Git repository* with the new image tag.
4.  **CD (Argo CD):** Argo CD detects the change in the K8s manifest and automatically deploys the new image to the AKS cluster, ensuring the cluster state matches the Git repo.

### The Application

We are deploying the "Example Voting App," which consists of:
* **`voting-app` (Python):** Frontend for casting votes.
* **`redis`:** In-memory queue for new votes.
* **`worker` (.NET):** Polls Redis and persists votes to Postgres.
* **`db` (Postgres):** Permanent database for vote data.
* **`results-app` (Node.js):** Frontend to display live results.

---

## 💻 Tech Stack

* **Cloud:** Microsoft Azure
* **CI/CD:** Azure DevOps (Azure Repos, Azure Pipelines)
* **Container Registry:** Azure Container Registry (ACR)
* **Container Orchestration:** Azure Kubernetes Service (AKS)
* **GitOps Tool:** Argo CD
* **Containerization:** Docker, Kubernetes YAML

---

## 🛠️ Step 1: Azure Setup (Prerequisites)

Before you begin, you must have the following Azure resources set up.

1.  **Azure DevOps Project:**
    * Create a new project in Azure DevOps.
    * Push this code to **Azure Repos** within that project.
2.  **Azure Container Registry (ACR):**
    * Create a private **ACR** instance.
    * Go to **Access keys** and **enable Admin user**. Note the `Username` and `Password` for later.
3.  **Azure Kubernetes Service (AKS):**
    * Create an **AKS cluster**.
    * Install the Azure CLI (`az`) and connect your local `kubectl` to the cluster:
        ```bash
        az login
        az aks get-credentials --resource-group <Your-RG-Name> --name <Your-AKS-Name>
        ```

---

## ⚙️ Step 2: Configure the CI Pipeline (Azure Pipelines)

This pipeline builds the Docker image, pushes it to ACR, and updates the K8s manifest in this repo.

1.  **Create a Self-Hosted Agent:**
    * Create a **Linux Virtual Machine** in Azure.
    * Install Docker on the VM.
    * In Azure DevOps, go to **Project Settings > Agent pools** > **Add pool**.
    * Follow the instructions to download and configure the agent on your VM. This agent will run our builds.
2.  **Create the CI Pipeline:**
    * In Azure Pipelines, create a **New pipeline** and select **Azure Repos Git**.
    * Choose "Existing Azure Pipelines YAML file" and point it to the `azure-pipelines.yml` file in this repository.
3.  **Configure Pipeline Variables:**
    * Before running, you must set up the pipeline's variables and connections:
    * **Service Connection:** Create a service connection to your Azure Container Registry.
    * **Agent Pool:** Edit the `azure-pipelines.yml` file and update the `pool` name to match the self-hosted agent pool you created.
        ```yaml
        pool:
          name: 'Your-Agent-Pool-Name' # <-- CHANGE THIS
        ```
    * **Variables:** Update the variables at the top of the YAML file.
        ```yaml
        variables:
          dockerRegistryServiceConnection: 'Your-ACR-Connection-Name' # <-- CHANGE THIS
          imageRepository: 'voting-app' # Can be any name you like
          containerRegistry: 'your-acr-name.azurecr.io' # <-- CHANGE THIS
          dockerfilePath: '$(Build.SourcesDirectory)/vote/Dockerfile'
          tag: '$(Build.BuildId)'
        ```
4.  **Update `update-k8s-manifest.sh`:**
    * This script commits the new image tag back to the repo. You must edit it:
    * Generate a **Personal Access Token (PAT)** in Azure DevOps with `Code (Read & write)` access.
    * Edit the `update-k8s-manifest.sh` file and replace the placeholder variables:
        ```bash
        AZURE_DEVOPS_PAT="<YOUR-PAT-TOKEN-HERE>"
        AZURE_DEVOPS_ORG_URL="[https://dev.azure.com/](https://dev.azure.com/)<YourOrgName>"
        PROJECT_NAME="<YourProjectName>"
        REPO_NAME="<YourRepoName>"
        ```

---

## ⛵ Step 3: Configure the CD Pipeline (Argo CD & GitOps)

This process pulls the configuration from Git and deploys it to AKS.

1.  **Install Argo CD on AKS:**
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
    ```
2.  **Expose Argo CD UI:**
    * Change the `argocd-server` service to `NodePort` (or set up an Ingress):
        ```bash
        kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
        ```
    * Get the NodePort:
        ```bash
        kubectl get svc -n argocd argocd-server
        ```
    * In the Azure Portal, go to your **AKS Node Pool's Network Security Group (NSG)** and add an **inbound rule** to allow traffic on that NodePort.
3.  **Get Argo CD Admin Password:**
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```
4.  **Create the Image Pull Secret:**
    * This allows AKS to pull private images from your ACR. Use the ACR Admin credentials from Step 1.
    ```bash
    kubectl create secret docker-registry acr-secret \
        --namespace=default \
        --docker-server=<your-acr-name>.azurecr.io \
        --docker-username=<ACR-Admin-Username> \
        --docker-password=<ACR-Admin-Password>
    ```
5.  **Connect Argo CD to Your Repo:**
    * Log in to the Argo CD UI (using the `admin` user and the password from step 3).
    * Go to **Settings > Repositories > Connect Repo**.
    * Enter your Azure Repo's **HTTPS URL**.
    * For `Username`, enter your Azure DevOps email.
    * For `Password`, use the **Personal Access Token (PAT)** you created earlier.
6.  **Create the Argo CD Application:**
    * In the UI, click **New App**.
    * **Application Name:** `voting-app`
    * **Project:** `default`
    * **Sync Policy:** `Automatic` & `Prune Resources`
    * **Source:**
        * **Repository URL:** Select your connected Azure Repo.
        * **Path:** `k8s-specifications`
    * **Destination:**
        * **Cluster URL:** `https://kubernetes.default.svc`
        * **Namespace:** `default`
    * Click **Create**. Argo CD will immediately sync the repo and deploy your application.

---

## ✅ How to Test

1.  Make a small code change to the `voting-app` (e.g., in `app.py`).
2.  Commit and push the change to your `main` branch in Azure Repos.
3.  **Observe:**
    * The **Azure Pipeline** will trigger, build, push the new image, and update the `vote-deployment.yaml` file in Git.
    * The **Argo CD UI** will show "OutOfSync" for a moment, then automatically sync the change and deploy the new pod.
4.  **Verify:**
    * Get the `NodePort` for the `vote-service`:
        ```bash
        kubectl get svc vote
        ```
    * Add a firewall rule for this port in your NSG (like you did for Argo CD).
    * Access `http://<Your-AKS-Node-IP>:<Vote-Service-NodePort>` in your browser to see your changes live.
