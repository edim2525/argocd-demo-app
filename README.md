# argocd-demo-app

A simple Node.js Hello World application used to demonstrate a full GitOps CI/CD pipeline with **Jenkins**, **Gitea**, and **ArgoCD** on a Raspberry Pi 5 k3s cluster.

**GitOps manifests repo:** [argocd-demo-gitops](http://10.100.102.51:30300/edim2525/argocd-demo-gitops)  
**Jenkins:** [http://10.100.102.51:30114](http://10.100.102.51:30114)  
**ArgoCD:** [http://10.100.102.51:30800](http://10.100.102.51:30800)  
**Gitea:** [http://10.100.102.51:30300](http://10.100.102.51:30300)

---

## Pipeline Overview

```
 ┌─────────────┐    ┌─────────────┐    ┌──────────────────┐    ┌─────────────────────┐
 │    Test     │───▶│ Build Image │───▶│ Push to Gitea    │───▶│  Update Manifest    │
 │  npm test   │    │   kaniko    │    │ Container Registry│    │ argocd-demo-gitops  │
 └─────────────┘    └─────────────┘    └──────────────────┘    └──────────┬──────────┘
                                                                           │
                                                                    ArgoCD detects change
                                                                           │
                                                                           ▼
                                                                ┌─────────────────────┐
                                                                │  Auto-sync to k3s   │
                                                                │   argocd-demo ns    │
                                                                └─────────────────────┘
```

Every push to `main` triggers Jenkins to:
1. Run Jest unit tests
2. Build a Docker image using **Kaniko** (no Docker daemon needed on k3s)
3. Push the image to the Gitea container registry tagged with the Jenkins build number
4. Clone [argocd-demo-gitops](http://10.100.102.51:30300/edim2525/argocd-demo-gitops), update the image tag in `manifests/deployment.yaml`, and push the change
5. ArgoCD detects the manifest change and auto-syncs the new version to the cluster

---

## Repository Structure

```
app-source/
├── src/
│   ├── index.js          # Express app — serves Hello World on / and health on /health
│   └── index.test.js     # Jest tests for both endpoints
├── package.json          # Dependencies: express. Dev: jest, supertest
├── Dockerfile            # Multi-stage build: node:18-alpine builder + slim runtime
├── Jenkinsfile           # Declarative pipeline (4 stages, Kubernetes agent with Kaniko)
└── README.md
```

---

## Application

A minimal **Express.js** server with two endpoints:

| Endpoint | Response |
|---|---|
| `GET /` | `{ "message": "Hello World", "version": "1.0.0" }` |
| `GET /health` | `{ "status": "ok" }` |

Listens on port `3000` (configurable via `PORT` env var).

---

## Files

### `src/index.js`
The Express application. Exports the server instance so Jest can cleanly start/stop it during tests.

### `src/index.test.js`
Jest + Supertest unit tests. Tests both the `/` and `/health` endpoints. Runs with `npm test`.

### `package.json`
| Script | Command |
|---|---|
| `npm start` | Starts the app on port 3000 |
| `npm test` | Runs Jest tests with `--forceExit` |

### `Dockerfile`
Multi-stage build to keep the image small:
- **Stage 1 (builder):** `node:18-alpine` — installs production dependencies only
- **Stage 2 (runtime):** `node:18-alpine` — copies only `node_modules` + `src/`, runs as non-root `node` user

The resulting image is compatible with `linux/arm64` (Raspberry Pi 5).

### `Jenkinsfile`
Declarative pipeline using a **Kubernetes pod agent** with three containers:

| Container | Image | Purpose |
|---|---|---|
| `node` | `node:18-alpine` | Run `npm ci` and `npm test` |
| `kaniko` | `gcr.io/kaniko-project/executor:debug` | Build and push Docker image without Docker daemon |
| `git-updater` | `alpine/git` | Clone gitops repo, update image tag, push commit |

Pipeline stages:
1. **Test** — `npm ci && npm test` in the `node` container
2. **Build & Push** — Kaniko builds the image and pushes two tags to Gitea registry: `:latest` and `:<BUILD_NUMBER>`
3. **Update Manifest** — Clones [argocd-demo-gitops](http://10.100.102.51:30300/edim2525/argocd-demo-gitops), updates the image tag in `manifests/deployment.yaml` using `sed`, commits and pushes

Jenkins credentials used:
- `gitea-registry-creds` — for Kaniko to authenticate with the Gitea container registry
- `gitea-git-creds` — for git push to the gitops repo

---

## Infrastructure

| Component | Location | URL |
|---|---|---|
| k3s cluster | Raspberry Pi 5 (master: ediraspberrypi1, worker: ediraspberrypi2) | — |
| Gitea | `gitea` namespace | [http://10.100.102.51:30300](http://10.100.102.51:30300) |
| Jenkins | `jenkins` namespace (USB storage on master) | [http://10.100.102.51:30114](http://10.100.102.51:30114) |
| ArgoCD | `argocd` namespace | [http://10.100.102.51:30800](http://10.100.102.51:30800) |
| App | `argocd-demo` namespace | NodePort (assigned after first sync) |

---

## Local Development

```bash
# Install dependencies
npm install

# Run tests
npm test

# Run the app
npm start
# → http://localhost:3000
```

---

## First-Time Setup

Before the pipeline can deploy the app, ensure these exist on the cluster:

```bash
# 1. Create namespace and imagePullSecret
kubectl create namespace argocd-demo
kubectl create secret docker-registry gitea-registry-secret \
  --docker-server=10.100.102.51:30300 \
  --docker-username=edim2525 \
  --docker-password=<gitea-token> \
  -n argocd-demo

# 2. Apply the ArgoCD Application
kubectl apply -f ../argocd/application.yaml
```

Then trigger the Jenkins pipeline — it will build, push, and deploy the app automatically.
