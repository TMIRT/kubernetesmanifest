<div align="center">

# 🚀 kubernetesmanifest

### CD / GitOps Manifest Repository

![GitOps](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=for-the-badge&logo=flask&logoColor=white)

</div>

---

This repository is the **CD half** of the TMIRT CI/CD pipeline. It holds the Kubernetes manifests for the `flaskdemo` security-testing application and is watched by ArgoCD, which automatically reconciles the cluster state whenever this repo changes.

The companion CI repo is [kubernetescode](../kubernetescode/), which builds the Docker image and triggers the manifest update job that writes to this repo.

---

## 🗂️ Repository Contents

| File | Purpose |
|---|---|
| 📄 `deployment.yaml` | Kubernetes `Deployment` (3 replicas) + `LoadBalancer` Service |
| 🔧 `Jenkinsfile` | Jenkins pipeline that patches the image tag and pushes to GitHub |
| 📖 `README.md` | This file |

---

## 🔄 Pipeline Overview

```
╔══════════════════════════════════╗
║     kubernetescode  (CI)         ║
╚══════════════════════════════════╝
  │
  ├─ 📥  Stage: Clone repository
  ├─ 🏗️  Stage: Build image         →  tmirtdockerhub/test:<BUILD_NUMBER>
  ├─ 🧪  Stage: Test image
  ├─ 📤  Stage: Push image           →  Docker Hub
  └─ ⚡  Stage: Trigger ManifestUpdate
              │
              ▼
╔══════════════════════════════════╗
║   kubernetesmanifest  (CD) ←here ║
╚══════════════════════════════════╝
  │
  ├─ 🖊️  Jenkinsfile patches deployment.yaml with new image tag
  ├─ 💾  Commits & pushes to GitHub (main branch)
  └─ 🔁  ArgoCD detects the change → syncs cluster
```

### Step-by-step

1. 👨‍💻 A developer pushes code to `kubernetescode`.
2. 🏗️ Jenkins builds and pushes a new Docker image tagged with the Jenkins build number.
3. ⚡ Jenkins triggers the `updatemanifest` job, passing `DOCKERTAG=<BUILD_NUMBER>`.
4. 🖊️ The `updatemanifest` job (Jenkinsfile in this repo) patches `deployment.yaml` and commits the change.
5. 🔁 ArgoCD detects the new commit and rolls out the updated deployment to the cluster.

---

## 🧱 Kubernetes Manifest Details

`deployment.yaml` defines two resources:

### 📦 Deployment — `flaskdemo`
| Field | Value |
|---|---|
| Image | `tmirtdockerhub/test:<tag>` *(auto-updated by Jenkins)* |
| Replicas | `3` |
| Image Pull Policy | `Always` |

### 🌐 Service — `lb-service`
| Field | Value |
|---|---|
| Type | `LoadBalancer` |
| External Port | `80` |
| Container Port | `5000` |

---

## 🔐 Application — flaskdemo

> ⚠️ **Security Notice** — This is a **deliberately vulnerable Flask web app** built for security training and penetration-testing exercises. Do not expose it to untrusted networks.

The app accepts user input and executes it as a shell command. It ships with:

| Feature | Details |
|---|---|
| 🛠️ Networking tools | `curl`, `ping`, `netcat`, `net-tools` |
| ☁️ Mock AWS credentials | Pre-baked as environment variables |
| 👤 Non-privileged user | `flaskuser` inside the container |

---

## ✅ Prerequisites

Before using this repo, ensure you have the following in place:

### ☸️ Kubernetes Cluster
- A running cluster with **ArgoCD** installed
- ArgoCD configured to watch this repository (`main` branch)

### 🔧 Jenkins Setup
Required plugins:
- 🐳 Docker Plugin & Docker Pipeline
- 🐙 GitHub Integration Plugin
- ⚙️ Parameterized Trigger Plugin

Required credentials:
- `github` — username + password/token for pushing to this repo
- `dockerhub` — credentials for `tmirtdockerhub` on Docker Hub

---

## 📜 Jenkinsfile — What It Does

The `Jenkinsfile` in this repo runs as the downstream `updatemanifest` job:

1. **📥 Clone repository** — checks out this manifest repo via SCM.
2. **🖊️ Update GIT** — uses `sed` to replace the image tag in `deployment.yaml`:
   ```
   tmirtdockerhub/test:<old-tag>  →  tmirtdockerhub/test:<DOCKERTAG>
   ```
   Then commits and pushes to `main` with a message referencing the Jenkins build number.

---

## 🔁 ArgoCD Setup

Install ArgoCD in your cluster and create an Application pointing to this repository:

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Configure the ArgoCD Application to:

| Setting | Value |
|---|---|
| 🗂️ Source | This GitHub repo, `main` branch, path `/` |
| 🎯 Destination | Your target cluster namespace |
| 🔄 Sync Policy | Automated (auto-deploy on every commit) |

---

<div align="center">

*Built and maintained by the TMIRT team · Security Training & Research*

</div>
