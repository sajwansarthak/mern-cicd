# MERN Stack App with CI/CD and GitOps on Kubernetes

A three-tier MERN application (React + Vite, Express, MongoDB) that is containerized with Docker, deployed to Kubernetes (Minikube), built and published by **GitHub Actions**, and delivered automatically by **Argo CD** using a GitOps workflow. A `git push` to `main` ends up running in the cluster with no manual steps.

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Run Locally with Docker](#run-locally-with-docker)
- [Run Locally with Docker Compose](#run-locally-with-docker-compose)
- [Deploy to Kubernetes (Minikube)](#deploy-to-kubernetes-minikube)
- [CI: GitHub Actions](#ci-github-actions)
- [CD: Argo CD](#cd-argo-cd)
- [Accessing the Application](#accessing-the-application)
- [Data Persistence](#data-persistence)
- [Troubleshooting Notes](#troubleshooting-notes)
- [Monitoring (Attempted, Not Completed)](#monitoring-attempted-not-completed)
- [Future Improvements](#future-improvements)

## Architecture

```
Developer
   |  git push (main)
   v
GitHub ──► GitHub Actions (CI)
              1. Build backend and frontend Docker images
              2. Push images to Docker Hub, tagged with the commit SHA
              3. Update the image tags in k8s/kustomization.yaml
              4. Commit the change back to the repo
                      |
                      v
              Git repo (single source of truth)
                      |
                      v
              Argo CD (CD) watches the repo's k8s/ folder
                      |  auto-sync, prune, self-heal
                      v
              Kubernetes (Minikube), namespace "mern"
                 ├── frontend  (Deployment + Service)
                 ├── backend   (Deployment + Service)
                 └── mongodb   (StatefulSet + PersistentVolumeClaim + headless Service)
```

## Tech Stack

| Layer | Technology |
| --- | --- |
| Frontend | React, Vite, Tailwind CSS |
| Backend | Node.js, Express |
| Database | MongoDB 7 |
| Containers | Docker, Docker Compose |
| Orchestration | Kubernetes (Minikube), Kustomize |
| CI | GitHub Actions, Docker Hub |
| CD | Argo CD (GitOps) |

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── ci.yaml              # CI pipeline
├── argocd/
│   └── application.yaml         # Argo CD Application for the app
├── k8s/
│   ├── backend.yaml             # Deployment + Service
│   ├── frontend.yaml            # Deployment + Service
│   ├── mongodb.yaml             # StatefulSet + headless Service + PVC template
│   └── kustomization.yaml       # Namespace + image tags (updated by CI)
├── mern/
│   ├── backend/                 # Express API (Dockerfile, db/, routes/, server.js)
│   └── frontend/                # React + Vite app (Dockerfile, src/, public/)
├── docker-compose.yaml
└── README.md
```

## Prerequisites

- Docker Desktop
- Minikube and kubectl
- A Docker Hub account (and an access token with **Read & Write** permission)
- A GitHub repository for this project

## Run Locally with Docker

Create a network so the containers can reach each other by name:

```sh
docker network create demo
```

Start MongoDB (the container must be named `mongodb`, because the backend connects to `mongodb://mongodb:27017`). A named volume keeps the data when the container is removed:

```sh
docker run --network=demo --name mongodb -d -p 27017:27017 -v mongo-data:/data/db mongo:7
```

Build and run the backend:

```sh
cd mern/backend
docker build -t mern-backend .
docker run --name=backend --network=demo -d -p 5050:5050 mern-backend
```

Build and run the frontend. Vite reads `VITE_API_URL` at build time, so pass it as a build argument:

```sh
cd mern/frontend
docker build -t mern-frontend --build-arg VITE_API_URL=http://localhost:5050 .
docker run --name=frontend --network=demo -d -p 5173:5173 mern-frontend
```

Open `http://localhost:5173`.

## Run Locally with Docker Compose

```sh
docker compose up -d --build
```

This starts the frontend, backend and MongoDB on a shared network, with the database stored in a named volume. Stop everything with `docker compose down` (add `-v` only if you also want to delete the database volume).

## Deploy to Kubernetes (Minikube)

Start a cluster. The Docker driver works well, and a few GB of memory is recommended:

```sh
minikube start --driver=docker --memory=4096 --cpus=2
```

Before relying on the pipeline, you can test the manifests by hand:

```sh
kubectl create namespace mern
kubectl apply -k k8s/
kubectl get pods -n mern -w
```

Before applying, replace `YOUR_DOCKERHUB_USERNAME` in `k8s/backend.yaml`, `k8s/frontend.yaml` and `k8s/kustomization.yaml` with your Docker Hub username, and make sure the images exist on Docker Hub (the CI pipeline pushes them automatically; for a first manual test, build and push them yourself). Delete the manual deployment with `kubectl delete -k k8s/` before handing control to Argo CD.

**What the manifests create**

- **Backend:** Deployment and ClusterIP Service on port 5050, with `MONGO_URI` set to `mongodb://mongodb:27017` and readiness/liveness probes.
- **Frontend:** Deployment and ClusterIP Service on port 5173.
- **MongoDB:** StatefulSet with a 1Gi PersistentVolumeClaim mounted at `/data/db`, exposed through a headless Service named `mongodb`.

## CI: GitHub Actions

The workflow in `.github/workflows/ci.yaml` runs on every push to `main`.

1. Log in to Docker Hub.
2. Build and push the backend and frontend images, tagged with the commit SHA.
3. Install Kustomize and update the image tags in `k8s/kustomization.yaml`.
4. Commit the change back to `main`.

Pushes that only change `k8s/**` or `argocd/**` are ignored through `paths-ignore`. This is important: it stops the pipeline's own manifest commit from triggering another run in an endless loop.

**One-time setup in GitHub**

- Add repository secrets under *Settings > Secrets and variables > Actions*:
  - `DOCKERHUB_USERNAME`: your Docker Hub username
  - `DOCKERHUB_TOKEN`: a Docker Hub access token with **Read & Write** permission (a read-only token fails with `insufficient scopes`)
- Under *Settings > Actions > General > Workflow permissions*, allow **Read and write permissions** so the workflow can commit the updated tags.

## CD: Argo CD

Argo CD runs inside the cluster, watches the `k8s/` folder in this repository, and syncs the cluster whenever the manifests change (about every 3 minutes by default, or instantly with a manual refresh).

Install Argo CD:

```sh
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

Update `repoURL` in `argocd/application.yaml` to point at your repository, then register the application:

```sh
kubectl apply -f argocd/application.yaml
```

The application uses automated sync with `prune` (remove resources deleted from Git) and `selfHeal` (revert manual changes made in the cluster), so Git is always the source of truth.

Open the Argo CD UI:

```sh
kubectl port-forward svc/argocd-server -n argocd 8081:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Go to `https://localhost:8081` and log in as `admin` with the password printed above.

**Try the full loop:** change something small in `mern/`, push to `main`, and watch GitHub Actions build the images, commit the new tags, and Argo CD roll out new pods.

## Accessing the Application

Services are internal to the cluster, so use port-forwarding. The browser calls the backend directly, so forward both:

```sh
kubectl port-forward svc/backend 5050:5050 -n mern
kubectl port-forward svc/frontend 5173:5173 -n mern
```

Then open `http://localhost:5173`.

## Data Persistence

- **Docker / Compose:** MongoDB data lives in a named volume (`mongo-data`). Removing the container does not remove the data; only deleting the volume does.
- **Kubernetes:** MongoDB data lives on a PersistentVolumeClaim created by the StatefulSet. Deleting the `mongodb-0` pod recreates it and the data is still there.

## Troubleshooting Notes

Problems hit while building this project, and their fixes.

| Symptom | Cause | Fix |
| --- | --- | --- |
| `kubectl apply -k k8s/` says the directory does not exist | Command run from inside `mern/`; `k8s/` is at the repo root | Run it from the repo root |
| Pods in `ImagePullBackOff`, "repository does not exist or may require authorization" | Images were never tagged and pushed under the Docker Hub name used in the manifests | Build with `docker build -t <user>/mern-backend:latest`, push, and make the repos public |
| CI fails with "no test specified" | The default `npm test` script exits with an error, and the workflow ran it | Replace it with a real test, a syntax check (`node --check server.js`), or remove the test job |
| CI push fails with `access token has insufficient scopes` | Docker Hub token was created read-only | Create a token with Read & Write permission and update the `DOCKERHUB_TOKEN` secret |
| Pushing Argo CD or manifest files does not trigger CI | `paths-ignore` skips runs that only change `k8s/**` or `argocd/**` | Expected behavior; change a file outside those folders or add `workflow_dispatch` for manual runs |
| Frontend cannot reach the backend | The browser cannot resolve in-cluster names such as `http://backend:5050`; Vite also ignores `REACT_APP_*` variables | Use `VITE_API_URL=http://localhost:5050` as a build argument and port-forward the backend |
| Pods crash with `exec format error` | Image CPU architecture (amd64 from GitHub runners) differs from the cluster (arm64 on Apple Silicon) | Build multi-platform images (`linux/amd64,linux/arm64`) with QEMU and Buildx |

## Monitoring (Attempted, Not Completed)

Monitoring was part of the original plan but is **not part of the final deployment**.

**What was attempted:** installing the `kube-prometheus-stack` Helm chart (Prometheus, Grafana and the standard Kubernetes dashboards) as a second Argo CD application in a separate `monitoring` namespace, so it would be managed through GitOps like the rest of the project.

**What happened:** on the local Minikube cluster the stack was too heavy alongside Argo CD and the application. The operator restarted repeatedly, Grafana and Prometheus were slow to start, and the cluster became overloaded. Removing the application through Argo CD also stalled and had to be cleaned up manually (removing the finalizer, deleting the namespace, and removing the leftover Prometheus CRDs). Monitoring was then removed entirely, so the project runs without it.

**How it could be completed:**

- Give Minikube more resources (at least 4 GB of memory, ideally 6 to 8 GB), or run the stack on a larger cluster.
- Install `kube-prometheus-stack` through Argo CD with trimmed settings (Alertmanager disabled, short retention, control-plane scrape targets disabled for Minikube) and `ServerSideApply=true` for the large CRDs.
- Use the built-in dashboards to view CPU and memory per pod in the `mern` namespace.
- Add application-level metrics by exposing a `/metrics` endpoint in the Express server (for example with `prom-client`) and a `ServiceMonitor` so Prometheus scrapes it, plus a MongoDB exporter for database metrics.

## Future Improvements

- Add automated tests (for example Jest and Supertest for the API) and re-enable a test stage in CI before the build.
- Serve the frontend as a production build behind nginx instead of the Vite dev server, and use slimmer base images (`node:20-alpine`) to shrink image sizes.
- Store credentials in Kubernetes Secrets, and add authentication to MongoDB.
- Add an Ingress so the app is reachable on one URL without port-forwarding.
- Complete the monitoring setup described above.