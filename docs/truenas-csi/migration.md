# Coder: TrueNAS CSI Migration

Migrate Coder and its dependencies from Democratic CSI (`truenas-nfs`) to TrueNAS CSI NFS (`truenas-csi-nfs`).

## Current setup (post-migration)

Postgres uses the **existing cluster-level** StorageClass `truenas-csi-nfs` with omit mapall + dataset permissions. The cluster config must have these parameters:

```yaml
parameters:
  nfs.mapAllUser: ""
  nfs.mapAllGroup: ""
  nfs.datasetPermissionsMode: "0777"
  nfs.datasetPermissionsUser: "0"
  nfs.datasetPermissionsGroup: "0"
```

`coder/overlays/prd-apps/postgres-cluster.yaml` uses `storageClass: truenas-csi-nfs` and `podSecurityContext` (no fsGroup) for NFS compatibility.

**Requires:** TrueNAS CSI driver with mapall and dataset permissions support.

---

## StorageClass (cluster-level)

The existing `truenas-csi-nfs` StorageClass is updated at cluster level. Parameters are immutable — to change them, delete and recreate (ensure no PVCs depend on it first).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: truenas-csi-nfs
provisioner: csi.truenas.io
parameters:
  protocol: "nfs"
  pool: "SAS"
  compression: "LZ4"
  nfs.mapAllUser: ""
  nfs.mapAllGroup: ""
  nfs.datasetPermissionsMode: "0777"
  nfs.datasetPermissionsUser: "0"
  nfs.datasetPermissionsGroup: "0"
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

Ensure cluster-level config applies before the coder overlay.

---

## What needs migration

| Component | PVCs | Size | Complexity |
|-----------|------|------|------------|
| **coder-postgres** | coder-postgres-1, 2, 3 | 10Gi each (30Gi total) | CloudNativePG |
| **Coder workspaces** | coder-*-home (per workspace) | 50Gi total | Dynamic PVCs via templates |

---

## 1. Migrate coder-postgres (CloudNativePG)

CloudNativePG uses volumeClaimTemplates; existing PVCs cannot change storage class. **Backup → rebuild → restore** keeps the same cluster name and connection string.

### 1.1 Backup the database

```bash
kubectl exec -n coder coder-postgres-1 -- pg_dump -U coder -h coder-postgres-rw -d coder > coder-dump.sql
```

### 1.2 Delete the old cluster

```bash
kubectl delete cluster coder-postgres -n coder
```

This removes the old PVCs (truenas-nfs). Coder will be unable to reach the DB until restore.

### 1.3 Rebuild the cluster (GitOps applies updated manifests)

```bash
kubectl apply -k coder/overlays/prd-apps/
# Or let Fleet/GitOps sync
```

Wait for the new cluster to be ready:

```bash
kubectl get cluster -n coder -w
# Phase: Cluster in healthy state
```

### 1.4 Restore the database

```bash
kubectl exec -i -n coder coder-postgres-1 -- psql -U coder -h coder-postgres-rw -d coder < coder-dump.sql
```

Restart Coder so it reconnects cleanly:

```bash
kubectl rollout restart deployment -n coder -l app.kubernetes.io/name=coder
```

No changes to `coder-db-url` — the cluster name and service stay the same.

---

## 2. Migrate Coder Workspace PVCs

Workspace PVCs (`coder-*-home`) are created by Coder templates. Each workspace has its own PVC.

### 2.1 Update workspace templates (new workspaces)

In the Coder UI: **Templates → Edit template → Workspace resources**  
Set the storage class to `truenas-csi-nfs` for new workspace PVCs.

### 2.2 Migrate existing workspaces

For each existing workspace that has a `truenas-nfs` PVC:

1. **Option A — User self-service**: Ask users to export their workspace data, stop the workspace, and start a new workspace (which will get the new PVC). They can re-import their data.

2. **Option B — Migration pod** (per workspace): Create new PVC, run a migration pod with both volumes mounted, copy data, then update the Coder template/resource to use the new PVC.

3. **Option C — Gradual**: Don't migrate existing workspaces. New workspaces use the new storage class. Old workspaces remain on Democratic CSI until users naturally cycle them.

---

## 3. Verification

```bash
# No PVCs should use truenas-nfs in coder namespaces
kubectl get pvc -n coder -o custom-columns='NAME:.metadata.name,SC:.spec.storageClassName'
kubectl get pvc -n coder-workspaces -o custom-columns='NAME:.metadata.name,SC:.spec.storageClassName'
```
