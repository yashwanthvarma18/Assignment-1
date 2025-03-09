# Minikube, ArgoCD & Argo Rollouts Setup Guide

This guide explains step-by-step how to set up a Minikube cluster, install ArgoCD and Argo Rollouts, deploy an example Nginx rollout using GitOps, and expose the application for testing.
## Prerequisites

- Docker installed
- kubectl installed
- Git installed
- GitHub account


## 1. Start a Minikube Cluster

```bash
minikube start --driver=docker --addons=none
```

Verify the cluster is running:

```bash
minikube status
kubectl cluster-info
```


## 2. Install ArgoCD

### Create ArgoCD Namespace and Deploy ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


### Expose the ArgoCD Server

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```


### Get the Initial Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode
```

The username is `admin` and the password is the output from the above command.

## 3. Install Argo Rollouts

### Create Namespace and Install Components

```bash
kubectl create namespace argo-rollouts
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/manifests/crds/rollout-crd.yaml
kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/manifests/install.yaml
```

Verify the installation:

```bash
kubectl get pods -n argo-rollouts
```


### Install the Argo Rollouts CLI Plugin

```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
sudo install -m 555 kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
kubectl-argo-rollouts version
```


### Install the Argo Rollouts Dashboard

```bash
kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/manifests/dashboard-install.yaml
```


## 4. Set Up GitOps Repository

### Create a Local Git Repository

```bash
mkdir -p my-gitops-repo
cd my-gitops-repo
```


### Create the Nginx Rollout Manifest

Create a file named `nginx-rollout.yaml` with the following content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: nginx-rollout
  namespace: default
spec:
  replicas: 3
  strategy:
    canary:
      steps:
        - setWeight: 20      # Update 20% of pods initially.
        - pause: {duration: 10s}  # Pause to monitor the new version.
        - setWeight: 50      # Update 50% of pods next.
        - pause: {duration: 20s}  # Pause again for observation.
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx       # This label matches pods with our Service.
    spec:
      containers:
        - name: nginx
          image: nginx:latest  # Using the latest nginx image.
          ports:
            - containerPort: 80
```


### Create the Nginx Service Manifest

Create a file named `nginx-service.yaml` with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: nginx      # This matches the pods labeled "app: nginx"
  ports:
    - protocol: TCP
      port: 80      # Exposes port 80 externally
      targetPort: 80  # Routes traffic to container's port 80
```


### Initialize Git and Push to GitHub

```bash
git init
git add .
git commit -m "Initial commit: Add nginx manifests"
git remote add origin https://github.com/yourusername/your-repo-name.git
git push -u origin main
```


## 5. Configure ArgoCD

Access the ArgoCD UI at https://localhost:8080 and log in using the credentials obtained earlier.

Create a new application with the following details:

- Application Name: nginx-rollout-app
- Project: default
- Repository URL: https://github.com/yourusername/your-repo-name.git
- Path: .
- Destination Cluster: https://kubernetes.default.svc
- Destination Namespace: default
- Sync Policy: Enable Automated Sync


## 6. Monitor and Test the Deployment

### Verify Rollout Status

```bash
kubectl get rollouts -n default
kubectl describe rollout nginx-rollout -n default
```


### Monitor Rollout Progress

```bash
kubectl argo rollouts get rollout nginx-rollout -n default --watch
```


### Test the Service

```bash
kubectl get svc -n default
minikube service nginx-service --url -n default
```

Visit the URL in your browser to view the Nginx welcome page.

## 7. Access the Argo Rollouts Dashboard

In one terminal:

```bash
kubectl port-forward -n argo-rollouts svc/argo-rollouts-dashboard 3100:3100
```

In another terminal:

```bash
kubectl port-forward -n argo-rollouts deployment/argo-rollouts 8090:8090
export ARGO_ROLLOUTS_API=http://localhost:8090
```

Access the dashboard at http://localhost:3100/rollouts

## 8. Update the Application

Edit the `nginx-rollout.yaml` file to change the image version:

```yaml
image: nginx:1.19
```

Commit and push the changes:

```bash
git add nginx-rollout.yaml
git commit -m "Update nginx image to 1.19"
git push origin main
```

ArgoCD will automatically detect the changes and trigger a new rollout with the canary strategy.

## 9. Stopping and Restarting the Cluster

To stop the cluster:

```bash
minikube stop
```

To restart the cluster:

```bash
minikube start --driver=docker --addons=none
```

After restarting, re-establish the port forwarding:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
kubectl port-forward -n argo-rollouts svc/argo-rollouts-dashboard 3100:3100
kubectl port-forward -n argo-rollouts deployment/argo-rollouts 8090:8090
```

Verify all resources are running:

```bash
kubectl get pods -A
```
Name: Yashwanth Varma 
