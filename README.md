# Dental Office Management System

Monorepo for a full-stack dental clinic app with Git submodules:
- **Backend**: Node.js + TypeScript (Express, JWT, Sequelize/ORM)  
- **Frontend**: Next.js (App Router)  
- **Infra**: Docker, AWS ECR, Kubernetes on AWS EKS, AWS RDS

---

## Repository Layout

```
├── apps
│   ├── dental-office-backend/    # API, models, controllers (submodule)
│   └── dental-office-frontend/   # Next.js UI (submodule)
├── backend-deployment.yaml       # K8s Deployment (backend)
├── backend-service.yaml          # K8s Service (backend, LB)
├── frontend-deployment.yaml      # K8s Deployment (frontend)
├── frontend-service.yaml         # K8s Service (frontend, LB)
└── README.md
```

### Submodules

```bash
# clone with submodules
git clone --recurse-submodules <repo-url>

# OR if already cloned
git submodule update --init --recursive
```

---

## Prerequisites

- **AWS**: an account with permissions for ECR, EKS, IAM, RDS
- **CLI tools**: `aws-cli`, `eksctl`, `kubectl`, `docker`
- **Cluster & DB**: An EKS cluster and an RDS instance reachable from the cluster

> EKS and RDS were created using **AWS Console** and AWS tooling. ECR hosts both images.

---

## Configuration

Create two env files at the repo root:

**`.env.ConfigMap`** (frontend, safe values)
```
# example
NEXT_PUBLIC_BACKEND_API_URL=http://<backend-lb-dns-or-ip>
```

**`.env.Secret`** (backend, sensitive values)
```
DB_HOST=<rds-endpoint>
DB_PORT=5432
DB_NAME=<db-name>
DB_USERNAME=<db-user>
DB_PASSWORD=<db-password>
JWT_SECRET=<random-strong-secret>
PORT=8000
```

> The backend Pod reads secrets via `backend-secret`; the frontend reads a ConfigMap named `frontend-config`.

---

## Build & Push Images (AWS ECR)

```bash
# Backend
docker build -t dental-office-backend apps/dental-office-backend
aws ecr get-login-password --region ap-southeast-2 \
  | docker login --username AWS --password-stdin 248189921368.dkr.ecr.ap-southeast-2.amazonaws.com
docker tag dental-office-backend:latest 248189921368.dkr.ecr.ap-southeast-2.amazonaws.com/dental-office-backend:latest
docker push 248189921368.dkr.ecr.ap-southeast-2.amazonaws.com/dental-office-backend:latest

# Frontend
docker build -t dental-office-frontend apps/dental-office-frontend
aws ecr get-login-password --region ap-southeast-2 \
  | docker login --username AWS --password-stdin 248189921368.dkr.ecr.ap-southeast-2.amazonaws.com
docker tag dental-office-frontend:latest 248189921368.dkr.ecr.ap-southeast-2.amazonaws.com/dental-office-frontend:latest
docker push 248189921368.dkr.ecr.ap-southeast-2.amazonaws.com/dental-office-frontend:latest
```

---

## Kubernetes Deployment (EKS)

### 1) Create ConfigMap & Secret

```bash
kubectl create configmap frontend-config --from-env-file=.env.ConfigMap
kubectl create secret generic backend-secret --from-env-file=.env.Secret

# (optional) verify contents (base64 for secrets)
kubectl get configmap frontend-config -o yaml
kubectl get secret backend-secret -o yaml
```

### 2) Apply Manifests

```bash
# Backend
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

# Frontend
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

### 3) Check Rollout & Pods

```bash
kubectl get pods -l app=backend
kubectl get pods -l app=frontend

# Trigger a safe restart after pushing new images (preferred):
kubectl rollout restart deployment/backend-deployment
kubectl rollout restart deployment/frontend-deployment

# (or force a new Pod if needed)
kubectl delete pod -l app=backend
kubectl delete pod -l app=frontend
```

### 4) Get Service Endpoints

Both services are `LoadBalancer` type:

```bash
kubectl get svc backend-service
kubectl get svc frontend-service
# Use EXTERNAL-IP (or hostname) once provisioned by AWS
```

---

## Manifests (Highlights)

- **Backend**: image `.../dental-office-backend:latest`, `containerPort: 8000`, env via `backend-secret`
- **Frontend**: image `.../dental-office-frontend:latest`, `containerPort: 3000`, env via `frontend-config` ConfigMap
- **Services**: expose each Deployment on port **80** (targets 8000/3000 respectively) through an AWS ELB

---

## Common Ops

```bash
# Tail logs
kubectl logs -l app=backend -f
kubectl logs -l app=frontend -f

# Describe resources
kubectl describe deploy backend-deployment
kubectl describe svc frontend-service
```

---

## Notes on AWS Setup

- **EKS**: created/managed with `eksctl`
- **RDS**: PostgreSQL for the backend
- **ECR**: hosts `dental-office-backend` and `dental-office-frontend` images
- Ensure the cluster nodes have IAM permissions for pulling from ECR and network access to RDS (VPC/subnets/SGs)
---

## Local Development (Optional)

You can develop each submodule locally following their own READMEs in:
- `apps/dental-office-backend/`
- `apps/dental-office-frontend/`

---