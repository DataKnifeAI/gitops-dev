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

## Database backups

`coder-postgres` archives WAL continuously and takes a daily base backup (09:00 UTC) through the
Barman Cloud plugin (installed by [gitops-core `cnpg-barman-cloud`](https://github.com/DataKnifeAI/gitops-core/tree/main/cnpg-barman-cloud)).

- **Where**: `s3://rke2-backups/cnpg/prd-apps/coder-postgres/` on rustfs
  (`https://rustfs.dataknife.net:30292`), 14 day retention — see `overlays/prd-apps/objectstore.yaml`.
- **Credentials**: `cnpg-backup-rustfs` secret in `coder` (keys `ACCESS_KEY_ID`, `ACCESS_SECRET_KEY`),
  created by hand, not in git.
- **WAL cap**: `max_slot_wal_keep_size: 2GB` stops a broken replica's slot from filling the 10Gi volume.

Check:

```bash
kubectl cnpg status coder-postgres -n coder
kubectl -n coder get backups.postgresql.cnpg.io
kubectl -n coder get clusters.postgresql.cnpg.io coder-postgres \
  -o jsonpath='{.status.conditions[?(@.type=="ContinuousArchiving")].status}'
kubectl cnpg backup coder-postgres -n coder --method=plugin --plugin-name=barman-cloud.cloudnative-pg.io  # on demand
```

Restore: create a new Cluster with `bootstrap.recovery.source` pointing at an `externalClusters`
entry using plugin `barman-cloud.cloudnative-pg.io` with `barmanObjectName: coder-postgres-rustfs`
and `serverName: coder-postgres` (optionally `recoveryTarget.targetTime` for PITR). Full example in
the gitops-core README. Rebuild a single broken replica with `kubectl cnpg destroy coder-postgres <n> -n coder`.

## References

- [Install Coder on Kubernetes](https://coder.com/docs/install/kubernetes)
- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/current/)
- [docs/truenas-csi/migration.md](../docs/truenas-csi/migration.md) — Migrate from Democratic CSI to TrueNAS CSI
