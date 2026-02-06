# Coder Secrets

Coder requires two secrets before deployment. Create these **before** applying the Coder manifests.

## 1. coder-postgres-credentials

Used by CloudNativePG to bootstrap the PostgreSQL database. Must contain `username` and `password`.

```bash
kubectl create secret generic coder-postgres-credentials -n coder \
  --from-literal=username=coder \
  --from-literal=password='YOUR_SECURE_PASSWORD'
```

## 2. coder-db-url

Used by Coder to connect to PostgreSQL. Must contain the full connection URL. The password must match `coder-postgres-credentials`.

```bash
# Replace YOUR_SECURE_PASSWORD with the same password from step 1
kubectl create secret generic coder-db-url -n coder \
  --from-literal=url="postgres://coder:YOUR_SECURE_PASSWORD@coder-postgres-rw.coder:5432/coder?sslmode=disable"
```

## Order of Operations

1. Create the `coder` namespace (or let Kustomize create it)
2. Create `coder-postgres-credentials`
3. Create `coder-db-url`
4. Apply the Coder manifests (postgres cluster will start, then Coder after DB is ready)

## Production Notes

- Use a secrets manager (Sealed Secrets, External Secrets, Vault) for production
- Consider using `CODER_ACCESS_URL` in the Helm values for workspace connectivity
- For TLS, configure `coder.tls.secretNames` with your certificate secret
