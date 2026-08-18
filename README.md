# CloudForm GitOps

The GitOps repository for the **CloudForm** project. FluxCD watches this repository and continuously reconciles the Kubernetes manifests here onto the EKS cluster — this is the single source of truth for what's actually running in the cluster.

## 📌 Purpose

Instead of anyone running `kubectl apply` by hand, deployments happen by **committing manifest changes to this repo**. Flux (bootstrapped onto the cluster by [`cloudform-infra`](https://github.com/rajeshdangi409/cloudform-infra)) polls this repository every minute and applies whatever it finds — a classic pull-based GitOps model.

The one exception is the container image tag, which is updated automatically by the CI pipeline in [`cloudform-app`](https://github.com/rajeshdangi409/cloudform-app) on every push to `main` — that pipeline commits directly into this repo, and Flux picks the change up from there.

## 🏗️ What's Deployed

| Manifest | Kind | Purpose |
|---|---|---|
| `deployment.yaml` | Deployment | Runs 2 replicas of the Flask app container |
| `configmap.yaml` | ConfigMap | Non-sensitive DB configuration (`DB_HOST`, `DB_USER`, `DB_NAME`, `DB_PORT`) |
| `service.yaml` | Service (ClusterIP) | Exposes the app internally on port 80 → container port 5000 |
| `ingress.yaml` | Ingress | Routes external traffic (via the `nginx` ingress controller) to the service |

The database password is **not** stored in this repo — the Deployment references it via `secretKeyRef` from a Kubernetes Secret (`flask-db-secret`) that's created directly in the cluster, outside of Git.

## 📁 Repository Structure

```
cloudform-gitops/
│
├── apps/
│   └── cloudform/
│       ├── deployment.yaml       # Flask app Deployment
│       ├── configmap.yaml        # non-sensitive DB config
│       ├── service.yaml          # ClusterIP Service
│       ├── ingress.yaml          # nginx Ingress
│       └── kustomization.yaml    # Kustomize resource list
│
└── clusters/
    └── production/
        └── cloudform-kustomization.yaml   # Flux Kustomization CR
```

## ⚙️ How Flux Wires It Together

`clusters/production/cloudform-kustomization.yaml` is a Flux `Kustomization` custom resource, **not** a plain Kustomize file:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
spec:
  interval: 1m
  path: ./apps/cloudform
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

- **`interval: 1m`** — Flux re-checks this repo for changes every minute
- **`path: ./apps/cloudform`** — tells Flux exactly which folder to reconcile
- **`prune: true`** — if a resource is removed from `apps/cloudform`, Flux deletes it from the cluster too (keeps the cluster in sync with Git, not just additive)
- **`wait: true`** — Flux waits for the applied resources to become healthy before marking the reconciliation successful
- **`sourceRef`** — points at the `GitRepository` object (created during `flux bootstrap`) that tracks this repo

`apps/cloudform/kustomization.yaml` is the actual Kustomize file that lists which manifests to build and apply from that path.

## 🔄 How a Deployment Happens

```
Developer pushes code to cloudform-app (main branch)
        │
        ▼
CI builds Docker image → pushes to ECR
        │
        ▼
CI clones this repo, updates the image tag in
apps/cloudform/deployment.yaml, commits & pushes
        │
        ▼
Flux (running in-cluster) detects the new commit
within ~1 minute
        │
        ▼
Flux applies the updated Deployment → EKS performs
a rolling update to the new image
```

No one needs cluster access or `kubectl` credentials to ship a new version — pushing to `cloudform-app` is enough.

## 🔍 Verifying Flux Is Working

```bash
# Check the Kustomization status
kubectl get kustomization -n flux-system

# Check Flux's own health
flux check

# Watch reconciliation logs
flux logs --follow

# Confirm the app is actually running
kubectl get pods -l app=flask-app
kubectl get deployment flask-app -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## 🔐 Secrets Handling

- `flask-db-secret` (containing `DB_PASSWORD`) is **not committed to this repository** — it's created directly in the cluster, either manually via `kubectl create secret` or through a secrets manager integration.
- Everything else needed to run the app — hostnames, usernames, ports — lives in `configmap.yaml`, which is safe to commit since it holds no credentials.
- For a more advanced setup, this repo could be extended with **Sealed Secrets** or the **External Secrets Operator** (pulling from AWS Secrets Manager / SSM Parameter Store) so that even encrypted secret references can live safely in Git.

## 🔗 Usage with Other CloudForm Repositories

```
cloudform-bootstrap  →  cloudform-infra
                              │  provisions EKS, RDS, ECR
                              │  bootstraps Flux, pointing it here
                              ▼
                    cloudform-gitops   ← this repository
                    Flux continuously reconciles
                    these manifests onto EKS
                              ▲
                              │  commits new image tag
                    cloudform-app
                    builds & pushes image on every push
```

## 👨‍💻 Project

**CloudForm** is a DevOps project demonstrating Infrastructure as Code, containerization, Kubernetes, CI/CD, and GitOps using AWS and modern DevOps tools.

Related repositories:
- [`cloudform-bootstrap`](https://github.com/rajeshdangi409/cloudform-bootstrap) — remote state backend
- [`cloudform-infra`](https://github.com/rajeshdangi409/cloudform-infra) — VPC, EKS, RDS, ECR + Flux bootstrap
- [`cloudform-app`](https://github.com/rajeshdangi409/cloudform-app) — Flask application + CI pipeline

## 📄 License

This project is licensed under the Apache License 2.0.