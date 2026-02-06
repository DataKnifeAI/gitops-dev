# Contributing to gitops-dev

Thank you for your interest in contributing! This document provides guidelines for contributing development tools to the prd-apps cluster.

## Code of Conduct

This project adheres to a [Code of Conduct](CODE_OF_CONDUCT.md) that all contributors are expected to follow.

## Getting Started

1. Fork the repository
2. Clone your fork: `git clone https://github.com/DataKnifeAI/gitops-dev.git`
3. Create a branch: `git checkout -b feat/your-feature-name`

## Development Guidelines

### Adding a New Tool

1. Create a directory: `{tool-name}/`
2. Add `base/` with manifests (namespace, helm chart, etc.)
3. Add `overlays/prd-apps/` with cluster-specific config
4. Update the root [README.md](README.md) with the new tool
5. Document secrets in `{tool-name}/secrets/README.md`

### Kubernetes Standards

- Use Kustomize for configuration management
- Keep base configurations generic; use overlays for prd-apps specifics
- Specify resource requests and limits
- Label resources: `managed-by: gitops`, `cluster: prd-apps`

### GitOps Principles

- All changes should be declarative and idempotent
- Never commit secrets; use Kubernetes Secrets with examples
- Test with `kubectl kustomize` before applying

### Commit Messages

Use conventional commits:

```
feat(coder): add GitHub OAuth for DataKnifeAI
fix(coder): reduce resource requests
docs: update secrets README
```

## Pull Request Process

1. Ensure changes follow the guidelines above
2. Update documentation as needed
3. Create a pull request with a clear description

### Checklist

- [ ] Manifests validated (`kubectl kustomize`)
- [ ] No secrets or sensitive data committed
- [ ] README updated if adding a new tool

## Security

- **Never commit secrets** — use `*.example` files and document in README
- Report security issues privately
