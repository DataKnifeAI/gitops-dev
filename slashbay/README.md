# Slashbay

[Slashbay](https://github.com/DataKnifeAI/slashbay) — DataKnifeAI issue-webhook herald. Issues dock here, cheap-LLM triage decides if they are work, and a Coder workspace is berthed for a coding agent.

Image is built in GitLab CI (GitHub mirror) and pushed to Harbor. This repo only deploys it.

## Target

| Item | Value |
|------|--------|
| Cluster | `prd-apps` |
| Namespace | `slashbay` |
| Image | `harbor.dataknife.net/library/slashbay` |
| Fleet path | `slashbay/overlays/prd-apps` |
| Ingress | `slashbay.dataknife.net` (nginx; GitHub/GitLab webhooks) |
| Coder | `https://coder.dataknife.net` (`CODER_ACCESS_URL`) |

## Prerequisites

1. Harbor image exists (`library/slashbay`) and `harbor-registry-secret` in namespace `slashbay`
2. Application secret `slashbay-secrets` — see [secrets/README.md](secrets/README.md)
3. DNS for `slashbay.dataknife.net` if webhooks must reach the cluster from GitHub/GitLab.com

## Deployment

```bash
kubectl config use-context prd-apps
# secrets first — see secrets/README.md
kubectl apply -k overlays/prd-apps/
```

`overlays/prd-apps` is self-contained (no `../../base`) so Fleet can use that directory as the GitRepo path, same as Coder.

Leave `SLASHBAY_DRY_RUN=true` on the ConfigMap until secrets and Coder are ready. Pin the image tag after the first Harbor release (siblings pin `high-command-*:v0.N`).

App-repo copy of these manifests: [DataKnifeAI/slashbay](https://github.com/DataKnifeAI/slashbay) `deploy/`.
