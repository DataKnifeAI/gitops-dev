# gitops-dev

Development tools deployed to the **prd-apps** Kubernetes cluster via GitOps.

## Overview

This repository contains GitOps configurations for developer-focused tools that run on the production applications cluster. Each tool is deployed using declarative manifests (Helm charts, Kustomize) and follows the same patterns used in [gitops-tools](https://github.com/DataKnifeAI/gitops-tools).

## Target Cluster

- **Cluster**: `prd-apps`
- **Context**: `kubectl config use-context prd-apps`

## Tools

### [Coder](https://coder.com) — Cloud Development Environments

Coder provides secure, on-demand development environments (workspaces) that run on your infrastructure. It integrates with Cursor, VS Code, JetBrains, and other IDEs.

- **Docs**: [Install Coder on Kubernetes](https://coder.com/docs/install/kubernetes)
- **Path**: `coder/`
- **PostgreSQL**: Deployed via [CloudNativePG](https://cloudnative-pg.io/) operator (same pattern as `high-command-postgres` in the cluster)
- **Storage**: TrueNAS CSI NFS — see [docs/](docs/README.md)

## Prerequisites

- [CloudNativePG operator](https://cloudnative-pg.io/documentation/current/installation/) installed on the cluster
- [Helm](https://helm.sh/) 3.5+
- [Fleet](https://fleet.rancher.io/) or `kubectl` for deployment

## Structure

```
gitops-dev/
├── coder/
│   ├── base/                    # Base Coder + PostgreSQL manifests
│   │   ├── postgres-cluster.yaml # CloudNativePG Cluster for Coder DB
│   │   ├── coder-helmchart.yaml  # Coder Helm chart config
│   │   └── ...
│   └── overlays/
│       └── prd-apps/            # prd-apps cluster overlay
│           ├── fleet.yaml
│           └── kustomization.yaml
├── docs/                        # Deployment and operations documentation
│   └── truenas-csi/             # TrueNAS CSI driver patches and migration
└── README.md
```

## PostgreSQL (CloudNativePG)

Coder requires PostgreSQL. This repo uses the **CloudNativePG operator** (same as `high-command-postgres` in the cluster) to provision a managed PostgreSQL cluster.

Connection format for Coder:
```
postgres://{user}:{password}@{cluster}-rw.{namespace}:5432/{database}?sslmode=disable
```

## Deployment

### Manual (kubectl)

```bash
kubectl config use-context prd-apps
kubectl apply -k coder/overlays/prd-apps/
```

### Fleet (GitOps)

Configure a Fleet GitRepo to monitor this repository and target the `prd-apps` cluster.

## Adding New Tools

1. Create a new directory (e.g., `tool-name/`)
2. Add `base/` and `overlays/prd-apps/` structure
3. Update this README with the new tool
4. Follow existing patterns from `coder/` and [gitops-tools](https://github.com/DataKnifeAI/gitops-tools)
