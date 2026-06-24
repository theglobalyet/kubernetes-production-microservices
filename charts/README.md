# Microservices — Kubernetes on GKE with Helm

Deploy Google's Online Boutique microservices application to Google Kubernetes Engine (GKE) using a reusable Helm chart architecture, following production and security best practices.

---

## Live Demo

> **![alt text](online-boutique-homepage.png)Online Boutique homepage running in browser at http://35.246.77.82**

---

## Technologies Used

| Tool | Purpose |
|------|---------|
| Google Kubernetes Engine (GKE) | Managed Kubernetes cluster |
| Helm | Kubernetes package manager / templating |
| Redis | In-memory cart session storage |
| kubectl | Cluster interaction CLI |
| Google Cloud CLI (gcloud) | GCP resource management |
| Docker | Container runtime |

---

## Project Structure

```
helm-chart-microservices/
├── charts/
│   ├── microservice/             # Reusable chart for all microservices
│   │   ├── templates/
│   │   │   ├── deployment.yaml   # Templated Deployment manifest
│   │   │   └── service.yaml      # Templated Service manifest
│   │   ├── Chart.yaml
│   │   └── values.yaml           # Default values (overridden per service)
│   └── redis/                    # Redis sub-chart
│       ├── templates/
│       │   ├── deployment.yaml
│       │   └── service.yaml
│       ├── Chart.yaml
│       └── values.yaml
├── values/                       # Per-service value overrides
│   ├── ad-service-values.yaml
│   ├── cart-service-values.yaml
│   ├── checkout-service-values.yaml
│   ├── currency-service-values.yaml
│   ├── email-service-values.yaml
│   ├── frontend-values.yaml
│   ├── payment-service-values.yaml
│   ├── productcatalog-service-values.yaml
│   ├── recommendation-service-values.yaml
│   ├── redis-values.yaml
│   └── shipping-service-values.yaml
├── config.yaml                   # Raw K8s manifests (reference)
├── helmfile.yaml                 # Helmfile to deploy all services at once
├── install.sh                    # One-command install script
└── uninstall.sh                  # One-command uninstall script
```

---

## Architecture Overview

The application is Google's **Online Boutique** — a cloud-native microservices e-commerce demo app consisting of 11 services.

```
                        ┌─────────────────┐
                        │    frontend     │ ← LoadBalancer (port 80)
                        └────────┬────────┘
           ┌────────────┬────────┼──────────────┬──────────────┐
           ▼            ▼        ▼              ▼              ▼
    cartservice  recommendationservice  checkoutservice   adservice
           │                           │
           ▼                    ┌──────┼────────────────┐
       redis-cart         emailservice  paymentservice  shippingservice
                                        currencyservice
                                        productcatalogservice
```

All internal services use `ClusterIP`. Only `frontend` is exposed via `LoadBalancer`.

---

## Prerequisites

- [ ] [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) installed and authenticated
- [ ] [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [ ] [Helm 3](https://helm.sh/docs/intro/install/) installed
- [ ] GCP account with a project created and billing enabled
- [ ] Kubernetes Engine API enabled on your GCP project

---

## Step-by-Step Deployment Guide

### Step 1 — Authenticate and Configure GCP

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
gcloud services enable container.googleapis.com
```

### Step 2 — Install the GKE Auth Plugin

```bash
gcloud components install gke-gcloud-auth-plugin
```

### Step 3 — Create the GKE Cluster

```bash
gcloud container clusters create myapp-cluster \
  --zone europe-west2-a \
  --num-nodes 2 \
  --machine-type e2-medium
```

> This takes approximately 5 minutes.

### Step 4 — Get Cluster Credentials

```bash
gcloud container clusters get-credentials myapp-cluster \
  --zone europe-west2-a

# Verify nodes are ready
kubectl get nodes
```

### Step 5 — Deploy All Services

```bash
chmod +x install.sh
./install.sh
```

> **![alt text](helm-list.png) helm list showing all 11 services STATUS: deployed**

### Step 6 — Verify Pods and Services

```bash
kubectl get pods
```

> **![alt text](online-microservices-pods.png)kubectl get pods showing all pods Running**

```bash
kubectl get svc
```

>**![alt text](online-microservices-svc.png)kubectl get svc showing frontend LoadBalancer with EXTERNAL-IP**

### Step 7 — Access the Application

```bash
kubectl get svc frontend
```

Copy the `EXTERNAL-IP` and open it in your browser:

```
http://<EXTERNAL-IP>
```

---

## Helm Chart Values Reference

Each service has its own `values/*.yaml` file overriding the shared chart template:

```yaml
# example: values/email-service-values.yaml
appName: emailservice
appImage: gcr.io/google-samples/microservices-demo/emailservice
appVersion: v0.8.0
appReplicas: 1
containerPort: 8080
servicePort: 5000
serviceType: ClusterIP
containerEnvVars:
  - name: PORT
    value: "8080"
```


---

## Teardown

```bash
# Uninstall all Helm releases
chmod +x uninstall.sh
./uninstall.sh

# Delete the GKE cluster
gcloud container clusters delete myapp-cluster --zone europe-west2-a
```

---

## Troubleshooting

| Symptom | Command | Notes |
|---------|---------|-------|
| Pod stuck in `Pending` | `kubectl describe pod <name>` | Check node capacity / resource limits |
| Pod in `CrashLoopBackOff` | `kubectl logs <pod-name>` | Check env vars and image pull errors |
| Service not reachable | `kubectl get endpoints <svc>` | Verify selector labels match pod labels |
| LoadBalancer IP pending | `kubectl describe svc frontend` | GKE takes 1–2 min to provision |
| Helm install fails | `helm lint charts/microservice` | Check values file YAML indentation |

---

## Services Summary

| Service | Image | Internal Port | Replicas |
|---------|-------|--------------|----------|
| emailservice | microservices-demo/emailservice:v0.8.0 | 5000 | 1 |
| recommendationservice | microservices-demo/recommendationservice:v0.8.0 | 8080 | 2 |
| productcatalogservice | microservices-demo/productcatalogservice:v0.8.0 | 3550 | 2 |
| paymentservice | microservices-demo/paymentservice:v0.8.0 | 50051 | 2 |
| currencyservice | microservices-demo/currencyservice:v0.8.0 | 7000 | 2 |
| shippingservice | microservices-demo/shippingservice:v0.8.0 | 50051 | 2 |
| adservice | microservices-demo/adservice:v0.8.0 | 9555 | 2 |
| cartservice | microservices-demo/cartservice:v0.8.0 | 7070 | 2 |
| redis-cart | redis:alpine | 6379 | 2 |
| checkoutservice | microservices-demo/checkoutservice:v0.8.0 | 5050 | 2 |
| frontend | microservices-demo/frontend:v0.8.0 | 80 → 8080 | 2 |

---

## Author

Yetunde — DevOps / Cloud Engineering
