# learning-cluster

A single-node k3s Kubernetes cluster managed with Flux GitOps. This repo is the source of truth for everything running on the cluster — no manual `kubectl apply`, no configuration drift.

## What this is

This is my homelab Kubernetes environment, built to learn and demonstrate production-grade infrastructure patterns: GitOps-driven deployments, encrypted secret management, and publicly exposed services without opening inbound ports.

The cluster runs on a k3s node (`nogrod`), managed remotely from `gundabad` via kubectl.

## Architecture

```
GitHub Repo (source of truth)
        │
        ▼
    Flux (GitOps operator)
        │
        ├── Kustomize overlays (base → staging)
        │
        ├── SOPS-encrypted secrets (age encryption)
        │       └── Decrypted at deploy time by Flux
        │
        └── Deployed workloads
                ├── Linkding (self-hosted bookmark manager)
                └── Cloudflare Tunnel (zero-trust ingress, no open ports)
```

## Repo Structure

```
apps/
├── base/
│   └── linkding/           # Base manifests (deployment, service, storage, namespace)
└── staging/
    └── linkding/           # Staging overlay (Cloudflare tunnel, encrypted secrets)

clusters/
└── learning/
    ├── apps.yaml           # Flux Kustomization pointing at apps/staging
    └── flux-system/        # Flux bootstrap components (gotk)

monitoring/                 # In progress
├── configs/
└── controllers/
    ├── base/               # Namespace + HelmRepository for kube-prometheus-stack
    └── staging/
```

## What's Deployed

### Linkding
Self-hosted bookmark manager running as a containerized workload with persistent storage. Exposed externally via Cloudflare Tunnel — no inbound firewall rules required.

### Secret Management
Secrets are encrypted in-repo using [SOPS](https://github.com/getsops/sops) with an age key. Flux decrypts them at deploy time using a key stored as a cluster secret. This means the full application state — including credentials — lives safely in version control.

Two secrets managed this way:
- `tunnel-credentials` — Cloudflare tunnel authentication
- `linkding-secret` — Linkding superuser credentials

### Cloudflare Tunnel
Provides zero-trust ingress to the cluster without exposing any ports to the internet. The tunnel runs as a sidecar deployment in the `linkding` staging overlay.

## Key Design Decisions

**Kustomize base/staging overlay pattern** — Base manifests define the workload; staging overlays add environment-specific configuration (secrets, tunnel, external access). Scales cleanly as additional environments or apps are added.

**SOPS over Sealed Secrets** — SOPS with age keys was chosen for simplicity and portability. Encrypted files are human-readable YAML, diff cleanly in Git, and don't require a controller running in-cluster to manage key rotation.

**Cloudflare Tunnel over port forwarding** — Eliminates the attack surface of open inbound ports entirely. Public access is brokered through Cloudflare's network with zero-trust controls at the edge.

**No manual applies** — All cluster state is driven through Flux reconciliation. If it isn't in this repo, it doesn't run on the cluster.

## In Progress

- **Monitoring** — kube-prometheus-stack (Prometheus + Grafana) via Helm, managed through Flux HelmRelease
- **Ingress** — Traefik ingress controller with TLS termination
- **Automated image updates** — Flux Image Automation Controller to detect and commit new image tags automatically

## Environment

| Component | Detail |
|-----------|--------|
| Cluster | k3s single-node (`nogrod`) |
| Control machine | `gundabad` |
| GitOps | Flux v2 |
| Secret encryption | SOPS + age |
| Overlay tool | Kustomize |
| External access | Cloudflare Tunnel |
