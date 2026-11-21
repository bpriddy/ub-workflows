# ub-workflows

Shared GitHub Actions workflows for UB services.

| Workflow | What it does | Docs |
|---|---|---|
| `.github/workflows/deploy-service.yml` | Builds an image once, deploys it to Cloud Run, promotes the same digest dev to stage to prod | [docs/deploy-service.md](docs/deploy-service.md) |

Call a workflow pinned to a tag:

```yaml
jobs:
  stage:
    uses: bpriddy/ub-workflows/.github/workflows/deploy-service.yml@v1
    secrets: inherit
    with:
      environment: stage
      # ...
```

Don't call `@main`. `v1` moves only for fixes that keep the interface; a
breaking change gets `v2`.

This repo is public so that both public and private repos can call it. It
must never hold secrets, project ids, project numbers, service account
emails or WIF provider paths. Callers pass those in, or set them on their own
GitHub Environments. A workflow here runs with the caller's credentials only.
