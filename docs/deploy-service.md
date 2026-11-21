# Shared deploy workflow

`.github/workflows/deploy-service.yml` is the one workflow every UB repo's
deploy calls. It builds an image once, deploys it to Cloud Run, and promotes
that same image by digest to the next environment.

Call it pinned to a tag, never `@main`:

```yaml
uses: bpriddy/ub-workflows/.github/workflows/deploy-service.yml@v1
```

`v1` moves only for fixes that keep the interface. A change to inputs,
outputs or rules gets a new tag (`v2`).

This repo is public, so any repo can call the workflow. That gives a caller
nothing: the workflow runs with the caller's own variables, secrets and OIDC
token. What it can deploy is decided by the caller's Workload Identity
Federation provider and deploy service account. No project ids, project
numbers, service accounts or provider paths live here; keep it that way.

## The rules it enforces

- **One build.** A call without `image_digest` builds the caller repo's image,
  pushes it to the shared Artifact Registry and deploys it. It outputs
  `image_digest`.
- **Promote by digest.** A call with `image_digest` deploys that exact image.
  It never rebuilds. `prod` refuses a call without a digest.
- **Prod needs approval.** The job runs in the GitHub Environment named by
  `environment`, so `prod`'s required reviewer gates it. GitHub does not allow
  `environment:` on a job that calls a reusable workflow, so the shared
  workflow sets it from the input.
- **Main only.** Every call refuses a ref other than `refs/heads/main`.
- **Never creates.** The service and every job it touches must already exist
  (Terraform provisions them).
- **Right project or nothing.** `GCP_PROJECT_ID`, `GCP_WORKLOAD_IDENTITY_PROVIDER`
  and `GCP_SERVICE_ACCOUNT_EMAIL` are read from the Environment's variables.
  The `targets` input maps `dev`, `stage` and `prod` to three different
  projects. The workflow checks that all three variables belong to the target
  environment's project, so a stage Environment with a missing variable cannot
  fall back to the repo-level prod value.
- **Rollback in dev and stage.** A service's new revision is created with no
  traffic; if it never becomes ready, nothing changed. Once traffic moves, a
  failed health check or any later failure moves 100% of traffic back to the
  revision that served before. Each job touched (the main job, the migrate
  job, `extra_jobs`) goes back to its previous image. In `prod` the job only
  fails and writes the rollback commands to the run summary.

## Interface

| Input | Default | Meaning |
|---|---|---|
| `environment` | required | `dev`, `stage` or `prod` |
| `service` | required | Cloud Run service or job name in that environment |
| `targets` | required | JSON `{"dev":{"project_id","project_number"},"stage":{...},"prod":{...}}`, three different projects |
| `registry` | required | `<location>-docker.pkg.dev/<project>`, shared by all environments |
| `kind` | `service` | `service` or `job` |
| `region` | `vars.GCP_REGION`, then `us-central1` | |
| `image_digest` | empty (build) | `sha256:...` to promote |
| `repository`, `image_name` | caller repo name | image is `<registry>/<repository>/<image_name>` |
| `dockerfile`, `context`, `build_args` | `Dockerfile`, `.`, none | build only |
| `flags` | none | deploy flags, one argument per line |
| `migrate_job`, `migrate_flags` | none | job updated to the image and executed with `--wait` before the deploy |
| `extra_jobs` | `[]` | JSON `[{"name","flags":[...]}]`, updated after the deploy, never executed |
| `health_path`, `health_auth`, `health_timeout_seconds` | none, `id-token`, 90 | services: path that must answer 200 |
| `timeout_minutes` | 30 | |

Environment variables read (set them on each of the caller's `dev`, `stage`
and `prod` Environments): `GCP_PROJECT_ID`, `GCP_WORKLOAD_IDENTITY_PROVIDER`,
`GCP_SERVICE_ACCOUNT_EMAIL`, optional `GCP_REGION`.

Pass `targets` from the calling workflow: a literal, or a repo-level or
org-level variable such as `${{ vars.UB_DEPLOY_TARGETS }}`. A `with:` value
is evaluated in the calling job, which has no Environment, so an Environment
variable cannot change it.

Secret: `GH_PACKAGES_TOKEN` (optional), given to the build as the BuildKit
secret `gh_packages_token`.

Outputs: `image_digest`, `image` (full `...@sha256:` reference), `revision`
and `url` (services).

Flags lines are passed to gcloud verbatim, never through a shell. Lines
starting with `#` are skipped; one pair of surrounding double quotes is
dropped, so lines copied from a bash `FLAGS=( ... )` array work. These
placeholders are replaced: `${PROJECT_ID}` `${PROJECT_NUMBER}` `${REGION}`
`${ENVIRONMENT}` `${SERVICE}` `${IMAGE}`. Bare `$PROJECT_ID` is not. A line
`--image=${IMAGE}` (a multi-container service, after its `--container=app`)
means the workflow adds no top-level `--image`. Values from GitHub secrets
cannot be used in `with:`; non-secret values move to Environment variables,
secret ones belong in Secret Manager (`--set-secrets`).

## Example caller

In the calling repo's `.github/workflows/deploy.yml`. On push to `main` it
builds and deploys to stage; then `promote` waits for the `prod` reviewer and
deploys the same digest. A run nobody approves expires after 30 days and
deploys nothing.

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npx tsc --noEmit

  stage:
    needs: check
    uses: bpriddy/ub-workflows/.github/workflows/deploy-service.yml@v1
    secrets: inherit
    with:
      environment: stage
      service: my-api-stage
      targets: ${{ vars.UB_DEPLOY_TARGETS }}
      registry: ${{ vars.UB_DEPLOY_REGISTRY }}
      flags: |
        --no-allow-unauthenticated
        --max-instances=2
        --port=3000
        --service-account=sa-${SERVICE}@${PROJECT_ID}.iam.gserviceaccount.com
        --set-secrets=DATABASE_URL=my-api-db-url:latest

  promote:
    needs: stage
    uses: bpriddy/ub-workflows/.github/workflows/deploy-service.yml@v1
    secrets: inherit
    with:
      environment: prod
      service: my-api
      targets: ${{ vars.UB_DEPLOY_TARGETS }}
      registry: ${{ vars.UB_DEPLOY_REGISTRY }}
      image_digest: ${{ needs.stage.outputs.image_digest }}
      flags: |
        --no-allow-unauthenticated
        --max-instances=2
        --port=3000
        --service-account=sa-${SERVICE}@${PROJECT_ID}.iam.gserviceaccount.com
        --set-secrets=DATABASE_URL=my-api-db-url:latest
```

To promote an older image, add a `workflow_dispatch` input `image_digest` and
pass it to a `prod` call. With a dev environment, the push job targets `dev`
and a `stage` job takes `needs.dev.outputs.image_digest`.

A service with a migration and side jobs:

```yaml
      migrate_job: ${SERVICE}-migrate
      migrate_flags: |
        --command=npx
        --args=prisma,migrate,deploy
        --set-secrets=DATABASE_URL=my-api-db-url:latest
        --service-account=sa-${SERVICE}@${PROJECT_ID}.iam.gserviceaccount.com
      extra_jobs: >-
        [{"name": "${SERVICE}-cleanup", "flags": ["--command=node",
          "--args=dist/scripts/cleanup.js"]}]
      health_path: /health
      health_auth: none
```

## IAM it needs

All environments pull from the one registry, so a digest built for stage is
the digest prod runs.

- **Cloud Run service agent of each environment's project** pulls the image:
  `roles/artifactregistry.reader` on the registry's repositories.
- **The deploy SA of the environment that builds** pushes:
  `roles/artifactregistry.writer` on those repositories. Deploy SAs of
  environments that only promote need `roles/artifactregistry.reader` (the
  workflow checks the digest exists).
- **Each deploy SA in its own project**: `roles/run.admin` (or
  `run.developer`), and `roles/iam.serviceAccountUser` on the runtime SAs it
  deploys as. The WIF provider and deploy SA live in the environment's own
  project (the workflow checks this).
- **Health checks with `health_auth: id-token`**: `roles/run.invoker` on the
  service for the deploy SA. If minting the ID token is refused, the WIF
  principal also needs `roles/iam.serviceAccountOpenIdTokenCreator` on the
  deploy SA.
- **WIF provider**: its attribute condition must limit which GitHub repos or
  owners can use it. That condition, not this repo, is what stops a stranger's
  repo from deploying.

## Not tested end to end

Nothing has run it against a real project yet. CI checks it with actionlint
and shellcheck (`.github/workflows/lint.yml`).
