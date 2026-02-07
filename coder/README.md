# Coder

[Coder](https://coder.com) — Cloud development environments (CDEs) for secure, on-demand workspaces on your infrastructure.

## Architecture

- **PostgreSQL**: CloudNativePG operator (same pattern as `high-command-postgres`)
- **Coder**: Helm chart from [helm.coder.com/v2](https://helm.coder.com/v2)

## Prerequisites

1. [CloudNativePG operator](https://cloudnative-pg.io/documentation/current/installation/) installed on the cluster
2. Create secrets (see [secrets/README.md](secrets/README.md))

## Deployment

```bash
# 1. Create secrets first (see secrets/README.md)
kubectl create namespace coder
# ... create coder-postgres-credentials and coder-db-url

# 2. Deploy
kubectl config use-context prd-apps
kubectl apply -k overlays/prd-apps/
```

## Access

After deployment, get the LoadBalancer IP:

```bash
kubectl get svc -n coder
```

Visit the IP in your browser to create the first admin account. For production, set `CODER_ACCESS_URL` to your domain and configure TLS.

## References

- [Install Coder on Kubernetes](https://coder.com/docs/install/kubernetes)
- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/current/)
- [TRUENAS_CSI_MIGRATION.md](TRUENAS_CSI_MIGRATION.md) — Migrate from Democratic CSI to TrueNAS CSI
