# Coder: TrueNAS CSI Migration

This document describes how to migrate Coder and its dependencies from Democratic CSI (`truenas-nfs`) to TrueNAS CSI (`truenas-csi-nfs`). See the rancher-deploy repository `docs/TRUENAS_CSI_MIGRATION.md` for the overall migration process.

## What Needs Migration

| Component | PVCs | Size | Complexity |
|-----------|------|------|------------|
| **coder-postgres** | coder-postgres-1, 2, 3 | 10Gi each (30Gi total) | CloudNativePG / StatefulSet |
| **Coder workspaces** | coder-*-home (per workspace) | 50Gi total | Dynamic PVCs via templates |

## Manifests Already Updated

The gitops manifests in this repo use `truenas-csi-nfs` for new deployments:

- `coder/base/postgres-cluster.yaml` — storageClass: truenas-csi-nfs
- `coder/overlays/prd-apps/postgres-cluster.yaml` — storageClass: truenas-csi-nfs

**Fresh installs** will use the new storage class. For **existing deployments**, follow the steps below.

---

## 1. Migrate coder-postgres (CloudNativePG)

CloudNativePG uses volumeClaimTemplates; existing PVCs cannot change storage class. **Backup → rebuild → restore** keeps the same cluster name and connection string.

### 1.1 Backup the database

```bash
kubectl config use-context prd-apps
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
# Or let Fleet/GitOps sync — manifests already use truenas-csi-nfs
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

**NFS ownership fix:** TrueNAS CSI NFS mounts as root by default. The postgres cluster uses `pvcTemplate.mountOptions` (`all_squash`, `anonuid=26`, `anongid=26`) so the postgres user (UID 26) can own the data directory. Without this, initdb fails with "data directory has wrong ownership".

---

## 2. Migrate Coder Workspace PVCs

Workspace PVCs (`coder-*-home`) are created by Coder templates. Each workspace has its own PVC.

### 2.1 Update workspace templates (new workspaces)

In the Coder UI: **Templates → Edit template → Workspace resources**  
Set the storage class to `truenas-csi-nfs` for new workspace PVCs.

New workspaces will use the new storage class automatically.

### 2.2 Migrate existing workspaces

For each existing workspace that has a `truenas-nfs` PVC:

1. **Option A — User self-service**: Ask users to export their workspace data, stop the workspace, and start a new workspace (which will get a `truenas-csi-nfs` PVC). They can re-import their data.

2. **Option B — Migration pod** (per workspace):

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pvc-migration-<workspace-id>
     namespace: coder-workspaces
   spec:
     containers:
       - name: migrate
         image: busybox:1.36
         command: ["sleep", "3600"]
         volumeMounts:
           - name: old
             mountPath: /old
           - name: new
             mountPath: /new
     volumes:
       - name: old
         persistentVolumeClaim:
           claimName: <OLD_WORKSPACE_PVC>
       - name: new
         persistentVolumeClaim:
           claimName: <NEW_PVC_NAME>
   ```

   Create new PVC with `truenas-csi-nfs`, run migration pod, copy data, then update the Coder template/resource to use the new PVC. This is template-specific.

3. **Option C — Gradual**: Don’t migrate existing workspaces. New workspaces use `truenas-csi-nfs`. Old workspaces remain on Democratic CSI until users naturally cycle them.

---

## 3. Verification

```bash
# No PVCs should use truenas-nfs in coder namespaces
kubectl get pvc -n coder -o custom-columns='NAME:.metadata.name,SC:.spec.storageClassName'
kubectl get pvc -n coder-workspaces -o custom-columns='NAME:.metadata.name,SC:.spec.storageClassName'
```

---

## References

- rancher-deploy `docs/TRUENAS_CSI_MIGRATION.md` — General migration process
- rancher-deploy `docs/TRUENAS_CSI_MIGRATION_PRIORITY.md` — Coder is priority #9 (postgres) and #11 (workspaces)
