# Argo CD Hub and Spoke Demo on EKS

This project demonstrates a multi-cluster GitOps workflow using a Hub and Spoke architecture on Amazon EKS. Argo CD, installed on a central **hub cluster**, is used to manage and deploy applications to multiple **spoke clusters**.

The sample application deployed is a simple "Guestbook" web application.

## Architecture

The architecture consists of three Amazon EKS clusters:

*   **Hub Cluster (`hub-cluster`):** This cluster hosts the Argo CD installation. It acts as the central control plane for deploying applications.
*   **Spoke Cluster 1 (`spoke-cluster-1`):** A target cluster where applications are deployed.
*   **Spoke Cluster 2 (`spoke-cluster-2`):** Another target cluster for application deployment.

Argo CD on the hub cluster is configured to manage the spoke clusters, allowing for centralized deployment and management of applications across a multi-cluster environment.

## Prerequisites

Before you begin, ensure you have the following tools installed and configured:

*   [AWS CLI](https://aws.amazon.com/cli/)
*   [eksctl](https://eksctl.io/introduction/#installation)
*   [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
*   [argocd CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

## Setup Instructions

### 1. Create EKS Clusters

Create the three EKS clusters (one hub and two spokes) using `eksctl`.

```bash
# Create the Hub Cluster
eksctl create cluster --name hub-cluster --region us-west-1

# Create the first Spoke Cluster
eksctl create cluster --name spoke-cluster-1 --region us-west-1

# Create the second Spoke Cluster
eksctl create cluster --name spoke-cluster-2 --region us-west-1
```

### 2. Set kubectl Context to Hub Cluster

Ensure your `kubectl` context is pointing to the `hub-cluster`. All subsequent commands for setting up Argo CD will be run against this cluster.

```bash
# Update kubeconfig for the hub cluster
aws eks update-kubeconfig --name hub-cluster --region us-west-1

# Verify the current context
kubectl config current-context
```

### 3. Install Argo CD on the Hub Cluster

Install Argo CD into its own namespace on the hub cluster.

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Apply the Argo CD installation manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. Expose Argo CD Server

By default, the Argo CD API server is not exposed externally. For this demo, we will change the `argocd-server` service type to `NodePort` to make it accessible.

```bash
# Edit the argocd-server service
kubectl edit svc argocd-server -n argocd

# In the editor, change the 'spec.type' from 'ClusterIP' to 'NodePort' and save the file.
#
# spec:
#   ports:
#   - name: http
#     port: 80
#     protocol: TCP
#     targetPort: 8080
#   - name: https
#     port: 443
#     protocol: TCP
#     targetPort: 8080
#   selector:
#     app.kubernetes.io/name: argocd-server
#   sessionAffinity: None
#   type: NodePort # <-- Change this line
```

Once saved, get the NodePort URL for the Argo CD server.

```bash
# Get the NodePort (look for the port mapped to 80/TCP)
kubectl get svc -n argocd argocd-server

# Get the external IP of one of your hub cluster nodes
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
NODE_PORT=$(kubectl get svc -n argocd argocd-server -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')

echo "Argo CD UI available at: http://$NODE_IP:$NODE_PORT"
```

### 5. Log in to Argo CD

First, retrieve the initial admin password for Argo CD.

```bash
# Get the initial admin password
argocd admin initial-password -n argocd
```

Now, log in using the Argo CD CLI. Use the NodePort URL you obtained in the previous step.

```bash
# Replace <NODE_IP> and <NODE_PORT> with your actual values
argocd login <NODE_IP>:<NODE_PORT> --username admin --password <YOUR_PASSWORD> --insecure
```

### 6. Add Spoke Clusters to Argo CD

Add the two spoke clusters to Argo CD so it can deploy applications to them. You will need to switch your `kubectl` context to each spoke cluster to generate the correct cluster configuration for Argo CD.

```bash
# Get the context names for your clusters
kubectl config get-contexts

# Add spoke-cluster-1
argocd cluster add <CONTEXT_NAME_FOR_SPOKE_1>

# Add spoke-cluster-2
argocd cluster add <CONTEXT_NAME_FOR_SPOKE_2>
```

After adding them, you can list the clusters in Argo CD to verify.

```bash
argocd cluster list
```

## Deploying the Guestbook Application

With the setup complete, you can now deploy the Guestbook application to the spoke clusters using an Argo CD `Application` resource.

### Application Manifests

The Kubernetes manifests for the Guestbook application are located in the `manifests/guest-book` directory:
*   `deployment.yml`: Deploys the Guestbook UI.
*   `service.yml`: Exposes the Guestbook UI via a `LoadBalancer` service.
*   `configmap.yml`: Provides a custom greeting message to the application.

### Create the Argo CD Application

Create a file named `guestbook-app.yaml` with the following content. This manifest tells Argo CD to deploy the Guestbook application to **both** spoke clusters.

**Note:** You must replace `<YOUR_GITHUB_REPO_URL>` with the URL of your forked or cloned repository.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <YOUR_GITHUB_REPO_URL> # e.g., https://github.com/amayjaiswal/argocd-hub-spoke-demo.git
    targetRevision: HEAD
    path: manifests/guest-book
  destination:
    # Deploy to both spoke clusters by name
    name: in-cluster # This is a placeholder, see ApplicationSet for multi-cluster
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

A better approach for deploying to multiple clusters is to use an `ApplicationSet`. Create a file named `guestbook-appset.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: spoke-cluster-1
        url: <URL_OF_SPOKE_1> # Get from `argocd cluster list`
      - cluster: spoke-cluster-2
        url: <URL_OF_SPOKE_2> # Get from `argocd cluster list`
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: <YOUR_GITHUB_REPO_URL> # e.g., https://github.com/amayjaiswal/argocd-hub-spoke-demo.git
        targetRevision: HEAD
        path: manifests/guest-book
      destination:
        server: '{{url}}'
        namespace: guestbook
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

Apply this manifest to your **hub cluster**:

```bash
# Set kubectl context back to the hub cluster
kubectl config use-context <CONTEXT_NAME_FOR_HUB>

# Apply the ApplicationSet manifest
kubectl apply -f guestbook-appset.yaml -n argocd
```

### Verify the Deployment

Once you apply the manifest, you can check the status of the application in the Argo CD UI or via the CLI. Argo CD will automatically sync the resources defined in the Git repository and deploy them to the specified spoke clusters.

To find the Guestbook UI, get the `LoadBalancer` address for the `guestbook-ui` service on each spoke cluster.

```bash
# Check on spoke-cluster-1
kubectl get svc -n guestbook --context <CONTEXT_NAME_FOR_SPOKE_1>

# Check on spoke-cluster-2
kubectl get svc -n guestbook --context <CONTEXT_NAME_FOR_SPOKE_2>
```
