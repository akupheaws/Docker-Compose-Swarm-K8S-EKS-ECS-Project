# Project Overview

This repository is a **hands‑on, portfolio‑ready lab** that demonstrates container orchestration across four levels of maturity: **Docker Compose** (single‑host dev), **Docker Swarm** (clustered but simple), **Kubernetes** (production‑grade orchestration), and two managed AWS flavors: **Amazon EKS** (managed Kubernetes) and **Amazon ECS** (native AWS scheduler).  
You can run the same sample application on each platform and compare **build → deploy → scale → observe** workflows side by side. The goal is to help engineers understand **trade‑offs, operational models, and deployment mechanics** across popular orchestrators while keeping **one codebase** and **repeatable automation**.

---

# Docker‑Compose • Swarm • K8s • EKS • ECS — Project

![Containers](https://img.shields.io/badge/Containers-Docker-orange)
![Kubernetes](https://img.shields.io/badge/Orchestrator-Kubernetes-blue)
![AWS](https://img.shields.io/badge/AWS-EKS%20%7C%20ECS-ff9900)
![IaC](https://img.shields.io/badge/IaC-Terraform-blueviolet)
![License](https://img.shields.io/badge/license-MIT-informational)

---

## ✨ What this project demonstrates

- **Same app, multiple orchestrators** — run locally with Compose, scale with Swarm, productionize with K8s, then deploy to **EKS** and **ECS**.
- **Infrastructure as Code** — optional Terraform modules for EKS/ECS/VPC; manifests and task definitions are versioned.
- **CI/CD friendly** — pipelines can build/push images, apply manifests, and run smoke tests on each target.
- **Observability hooks** — examples for logs/metrics (CloudWatch on ECS/EKS; `kubectl top`/`metrics-server` on K8s; `docker stats` locally).
- **Cost awareness** — local runs are free; cloud runs document cleanup commands and guardrails.

---

## 📦 Example repository structure

> Adjust the folders to match your repo if they differ. The layout below keeps each orchestrator self‑contained but shares the same app image(s).

```
.
├─ app/                         # Sample app (e.g., Node/Express API + simple web UI)
│  ├─ src/
│  └─ Dockerfile
├─ compose/                     # Docker Compose dev stack
│  └─ docker-compose.yml
├─ swarm/                       # Docker Swarm stack
│  └─ stack.yml
├─ k8s/                         # Kubernetes manifests (base)
│  ├─ deployment.yaml
│  ├─ service.yaml
│  └─ ingress.yaml
├─ eks/                         # EKS-specific overlays/IaC
│  ├─ kustomization.yaml
│  └─ (terraform/ or eksctl/)
├─ ecs/                         # ECS task/service definitions + IaC
│  ├─ task-definition.json
│  └─ service.json
├─ terraform/                   # Optional: VPC/EKS/ECR/ECS IaC modules
├─ scripts/                     # Helper scripts (build, push, smoke-tests)
├─ .github/workflows/           # (Optional) CI/CD pipelines
└─ README.md
```

---

## 🧭 Architecture (high level)

```
                          ┌─────────────── Build & Push ───────────────┐
                          │ docker buildx bake / docker build & push   │
                          └─────────────────────────┬───────────────────┘
                                                    │
                                        Container registry (ECR/DockerHub)
                                                    │
         ┌───────────────────────────────┬──────────┴─────────┬───────────────────────────────┐
         ▼                               ▼                    ▼                               ▼
   Docker Compose                    Docker Swarm        Kubernetes (local)            AWS (EKS / ECS)
single host dev env            swarm services/stack     deployments/services/ingress   managed control plane
`docker compose up`            `docker stack deploy`    `kubectl apply -k k8s/`        IaC + `kubectl` or `aws ecs`
```

---

## 🚀 Quickstart

### 1) Build the app image (shared for all targets)
```bash
# from repo root
docker build -t your-dockerhub-username/app:latest ./app
```

> In CI, prefer `docker buildx build --platform linux/amd64 -t <registry>/app:SHA --push` to produce multi‑arch images and immutable tags.

### 2) Run with Docker Compose (local)
```bash
cd compose
docker compose up -d
# open http://localhost:8080  (or whichever port your compose file exposes)
```

### 3) Run on Docker Swarm (local or remote manager)
```bash
# init swarm on your host (if not already)
docker swarm init

# deploy the stack
docker stack deploy -c swarm/stack.yml appstack

# scale an app service
docker service ls
docker service scale appstack_web=3
```

### 4) Run on Kubernetes locally (kind or minikube)
```bash
# create a local cluster (choose one)
kind create cluster --name orch-lab
# or: minikube start

# deploy base manifests
kubectl apply -f k8s/

# verify
kubectl get deploy,svc,ingress -A
```

### 5) Deploy to Amazon EKS
```bash
# (IaC) create cluster using Terraform or eksctl
# terraform -chdir=terraform/eks init && terraform apply -auto-approve
# eksctl create cluster -f eks/cluster.yaml

# update kubeconfig and deploy
aws eks update-kubeconfig --name <cluster-name> --region <region>
kubectl apply -k eks/
```

### 6) Deploy to Amazon ECS (Fargate)
```bash
# (IaC) create VPC/ECR/Cluster/Service via Terraform (or AWS Copilot)
# terraform -chdir=terraform/ecs init && terraform apply -auto-approve

# register task definition and create service
aws ecs register-task-definition --cli-input-json file://ecs/task-definition.json
aws ecs create-service --cli-input-json file://ecs/service.json
```

---

## 🔧 Configuration & variables

- **Images:** `your-registry/app:<tag>` — set once and reference across compose/swarm/k8s/ecs.
- **Ports:** `8080` (web) by default; change in manifests/compose/stack files.
- **Environment:** common `.env` can be consumed by Compose and scripts. Use **Secrets Manager/SSM** for cloud secrets.
- **Ingress/ALB:** K8s uses Ingress (Nginx/ALB); ECS uses ALB target groups.

---

## 🤖 (Optional) CI/CD pipeline outline

1. **Lint & Test**: run unit tests (`npm test`, etc.).  
2. **Build & Push**: build image once, tag with commit SHA, push to ECR/DockerHub.  
3. **Deploy (matrix)**: job matrix runs **compose (smoke)**, **swarm**, **k8s**, **eks**, **ecs** targets.  
4. **Post‑deploy checks**: curl /health, check service/task/replica counts.  
5. **Cleanup**: ephemeral envs destroyed on branch deletion.

> Add badges once your workflow filenames are final, e.g. `orchestration.yml`.

---

## 🧪 Smoke tests

- **Compose/Swarm:** `curl http://localhost:8080/health`  
- **K8s/EKS:** `kubectl -n <ns> port-forward svc/web 8080:80 && curl http://localhost:8080/health`  
- **ECS:** Find the ALB DNS, then `curl http://<alb-dns>/health`.

---

## 🔒 Security & cost best practices

- Use **OIDC** for CI to assume AWS roles (no long‑lived keys).  
- Restrict **ECR/EKS/ECS** permissions to least privilege.  
- Enable **deletion protection**/guardrails only where appropriate; provide **destroy** targets for labs.  
- Set **resource limits/requests** on K8s; set **desired count** and **autoscaling** on ECS.  

---

## 🛣️ Roadmap

- [ ] Add Helm chart with values for env parity.  
- [ ] Add HPA (K8s) and ECS Service Auto Scaling examples.  
- [ ] Add tracing (X‑Ray, OpenTelemetry) and dashboards (Grafana).  
- [ ] Blue/Green canary on EKS with Argo Rollouts; ECS weighted target groups.  
- [ ] GitHub Actions matrix deploy across all targets.

---

## 🆘 Troubleshooting

- **Image not pulling** → check registry auth/permissions and image tag.  
- **Ingress 404 on K8s** → correct host/path rules and controller installation.  
- **ECS tasks draining** → verify target group health checks, security groups, subnets.  
- **EKS `CrashLoopBackOff`** → inspect logs (`kubectl logs`), verify env vars and config maps.  

---

## 📜 License

MIT © Akuphe
