# Scaling Applications with Kubernetes: Load Balancing and Autohealing on GKE
**By Mwiine Daniel and Batamye Umar Isabirye**

## Overview
This repository contains materials for the hands-on demo **"Scaling Applications with Kubernetes: Load Balancing and Autohealing on GKE"**.  
It includes architecture diagrams and a complete, copy‑pasteable step-by-step guide you can run during a live presentation or workshop.

> **Important:** Replace placeholder values (like `PROJECT_ID` and `BILLING_ID`) with your actual project and billing IDs before running the commands.

---

## Prerequisites
- `gcloud` CLI installed and authenticated.
- `kubectl` installed (the `gcloud` SDK can install it for you).
- A Google Cloud billing account (you will link it to the demo project).
- Sufficient quota to create a small GKE cluster.

---

## Files in this repo
- `deployment.yaml` — demo 1 application Deployment (autohealing).
- `service.yaml` — demo 1 Service (LoadBalancer).
- `hpa-deployment.yaml` — demo 2 application (HPA demo).
- `gke_load_balancer_architecture.png` — GKE Load Balancer architecture diagram.
- `hpa_scaling_diagram.png` — HPA scaling diagram.

> Note: The YAML files are provided in the presentation materials; you can also create them from the commands shown below.

---

## Full Demo Guide (copy & paste during presentation)

### Phase 1 — Project & Billing Setup
**Goal:** Create a project, link billing, and enable the Kubernetes API.

1. **Log in**
```bash
gcloud auth login
```

2. **Create a Project**
```bash
export PROJECT_ID="your-unique-project-id-here"
export PROJECT_NAME="DevFest Mbarara Demo"

gcloud projects create ${PROJECT_ID} --name="${PROJECT_NAME}"
```

3. **Set Project & Link Billing**
```bash
gcloud config set project ${PROJECT_ID}

gcloud beta billing accounts list

export BILLING_ID="012345-6789AB-CDEF01"

gcloud beta billing projects link ${PROJECT_ID} --billing-account=${BILLING_ID}
```

4. **Enable Kubernetes API**
```bash

gcloud services enable container.googleapis.com
```

---

### Phase 2 — Create GKE Cluster
**Goal:** Provision a 3-node cluster.

```bash
export CLUSTER_NAME="demo-cluster"
export CLUSTER_REGION="us-central1"

# What to say:
# "I'll create a 3-node GKE cluster. This normally takes 5–10 minutes."
gcloud container clusters create ${CLUSTER_NAME}     --region ${CLUSTER_REGION}     --num-nodes=3     --machine-type=e2-standard-2

# Configure kubectl
gcloud container clusters get-credentials ${CLUSTER_NAME} --region ${CLUSTER_REGION}

# Verify nodes
kubectl get nodes

# What to say:
# "You should see three Ready nodes. We're ready for Demo 1."
```

---

## Phase 3 — Demo 1: Autohealing
**Goal:** Deploy an app with liveness/readiness probes, kill it, and show Kubernetes self-heal.

### 1) Create `deployment.yaml`
```bash

cat > deployment.yaml << 'EOL'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
EOL
```

### 2) Create `service.yaml`
```bash

cat > service.yaml << 'EOL'
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  type: LoadBalancer
  selector:
    app: hello
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
EOL
```

### 3) Deploy and watch
```bash

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

kubectl get pods -w

# Terminal 2 (watch service external IP)
kubectl get service hello-service --watch
# Once IP appears, stop watch with Ctrl+C

# Test the service (replace below with actual IP)
export SERVICE_IP=$(kubectl get service hello-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://${SERVICE_IP}
# Expect: "Hello, world!"
```

### 4) Break the app (simulate crash)
```bash

export POD_NAME=$(kubectl get pods -l app=hello -o jsonpath='{.items[0].metadata.name}')
echo "--- Breaking pod: ${POD_NAME} ---"

kubectl exec -it ${POD_NAME} -- /bin/sh
# (inside pod) run:
# ps aux
# kill 1
# exit
```

### 5) Watch healing
- Point attendees to the terminal showing pods: you'll see readiness/liveness fail, then a restart or new pod creation.
- Explain that readinessProbe removes pod from service endpoints; livenessProbe triggers restarts.

---

## Phase 4 — Demo 2: Autoscaling (HPA)
**Goal:** Deploy a CPU-consuming app, create HPA, generate load, and watch scaling.

### 1) Clean up Demo 1
```bash

kubectl delete deployment hello-app
kubectl delete service hello-service
```

### 2) Create `hpa-deployment.yaml`
```bash

cat > hpa-deployment.yaml << 'EOL'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
EOL
```

### 3) Deploy, expose, and create HPA
```bash
kubectl apply -f hpa-deployment.yaml

# Expose via LoadBalancer
kubectl create service loadbalancer php-apache --tcp=80:80

# Create HPA: target 50% CPU, min 1, max 5 replicas
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5
```

### 4) Watch the HPA
```bash
# Terminal 1
kubectl get hpa -w
```

### 5) Generate load
```bash
# Terminal 2: wait for LB IP
export PHP_IP=$(kubectl get service php-apache -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
while [ -z "${PHP_IP}" ]; do
  sleep 2
  export PHP_IP=$(kubectl get service php-apache -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
done
echo "IP: ${PHP_IP}"

# Run a temporary load-generator pod (interactive)
kubectl run -it --rm load-generator --image=busybox -- /bin/sh

# Inside the load-generator shell:
# while true; do wget -q -O - http://${PHP_IP}; done
# (Ctrl+C when done; exit to leave pod)
```

### 6) Watch scaling
- Observe `kubectl get hpa -w` and `kubectl get pods -w` in different terminals.
- Explain scaling behavior: HPA reads metrics, increases replicas when CPU % > target, and respects min/max.

### 7) Stop load and observe scale-down
- Stop the load-generator (Ctrl+C, then `exit`).
- HPA observes lower CPU and will scale down after cooldown (~5 minutes by default).

---

## Phase 5 — Cleanup (VERY IMPORTANT)
**Goal:** Delete resources to avoid charges.

```bash
# Delete HPA
kubectl delete hpa php-apache

# Delete deployments and services
kubectl delete deployment php-apache
kubectl delete service php-apache

# Delete cluster
echo "--- Deleting cluster; may take 5-10 minutes ---"
gcloud container clusters delete ${CLUSTER_NAME} --region ${CLUSTER_REGION} --quiet

# Optional: delete the entire project
echo "--- Deleting project in 30 seconds... Press Ctrl+C to cancel ---"
sleep 30
gcloud projects delete ${PROJECT_ID} --quiet
```

---

## YAML Explanation (Quick Reference)
- **Deployment**: declares desired Pods, replica count, container image, and probes. `livenessProbe` triggers restarts; `readinessProbe` controls LB traffic routing.
- **Service (LoadBalancer)**: provisions an external IP and routes traffic to Pods via their labels.
- **HPA (Horizontal Pod Autoscaler)**: watches metrics (CPU by default) and adjusts `replicas` between a min and max. **You must specify `resources.requests.cpu`** for HPA to compute percentages.

---

## Tips for Presenters
- Use **three terminal tabs**: 1) `kubectl get pods -w`, 2) service/exec/load generator, 3) `kubectl get hpa -w` and other diagnostics.
- Tell attendees to **replace placeholders** (`PROJECT_ID`, `BILLING_ID`) before running commands.
- Mention billing and cleanup at the start so participants remember to delete resources.

---

## Authors
- **Mwiine Daniel**
- **Batamye Umar Isabirye**

---

## Licensing
Use this content freely for workshops and teaching. Attribution appreciated.
