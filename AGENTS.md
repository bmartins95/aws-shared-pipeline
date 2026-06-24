# aws-shared-pipeline — reusable GitHub Actions workflows

This repo contains reusable GitHub Actions workflows and composite actions shared across all apps in the platform. It has no application code and no deployment of its own.

## Structure

```
.github/
  workflows/
    register-infra.yml      # Register app config + wait for aws-infra deploy
    deploy-stage.yml        # Full deploy stage: tools → AWS → ci-deploy.sh → SSM → ci-build-web.sh → promote
  actions/
    promote-branch/         # Composite: force-push HEAD to a target branch
    read-platform-ssm/      # Composite: read Cognito + BackendApiUrl SSM outputs
templates/
  deploy.yml                # Copy-paste starting point for a new app's pipeline
```

## Workflows

### `register-infra.yml`

Registers an app's infrastructure config with `aws-infra` and waits for the platform deployment to complete.

**What it does:**
1. Reads `infra/{stage}.yaml` from the caller's repo
2. Pushes it to `apps/{app_name}/{stage}.yaml` in `aws-infra` via GitHub API
3. Fires a `repository_dispatch` to trigger the aws-infra pipeline
4. Waits for that run to complete (exits non-zero on failure)

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `app_name` | yes | — | App name — matches `apps/` folder in aws-infra and SSM namespace |
| `stage` | yes | — | Stage (`prod`, `stg`, `dev`) — must match `infra/{stage}.yaml` in caller |
| `infra_repo` | no | `{owner}/aws-infra` | Override the aws-infra repo |

**Secrets:** `INFRA_GITHUB_TOKEN` — PAT with `contents:write` + `actions:write` on aws-infra. Use `secrets: inherit` in callers.

---

### `deploy-stage.yml`

Full deploy job for one stage. Apps call this instead of writing their own deploy steps.

**Contract — the calling repo must provide:**
- `scripts/ci-deploy.sh <stage>` — deploys the backend (receives stage as `$1`)
- `scripts/ci-build-web.sh <stage>` — builds the frontend (receives stage as `$1`; Cognito + backend env vars are injected automatically)

**What it does (in order):**
1. Checkout (with `PROMOTE_TOKEN` if provided, else `GITHUB_TOKEN`)
2. Setup Python 3.12 + uv (if `stack: python-sst`)
3. Setup Go 1.22 (if `stack: go`)
4. Setup Node.js
5. Configure AWS credentials
6. `bash scripts/ci-deploy.sh $stage`
7. Read SSM outputs: `CognitoUserPoolId`, `CognitoWebClientId`, `CognitoDomain`, `BackendApiUrl`
8. `bash scripts/ci-build-web.sh $stage` (if `has_web: true`) — env vars injected
9. Force-push HEAD to `promote_branch` (if set)

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `app_name` | yes | — | SSM namespace prefix |
| `stage` | yes | — | `dev`, `stg`, or `prod` |
| `promote_branch` | no | `''` | Branch to force-push to after deploy (e.g. `staging`, `master`) |
| `stack` | no | `'node'` | Backend stack: `python-sst` (Python 3.12 + uv), `go` (Go 1.22), `node` (Node only) |
| `node_version` | no | `'20'` | Node.js version |
| `has_web` | no | `true` | Whether to run `ci-build-web.sh` |
| `aws_region` | no | `'us-east-1'` | AWS region |

**Secrets:** `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (required); `PROMOTE_TOKEN` (optional, falls back to `GITHUB_TOKEN`). Use `secrets: inherit` in callers.

**Usage:**
```yaml
deploy-dev:
  uses: bmartins95/aws-shared-pipeline/.github/workflows/deploy-stage.yml@master
  with:
    app_name: my-app
    stage: dev
    stack: python-sst
  secrets: inherit
```

## Composite Actions

### `promote-branch`

Force-pushes HEAD to a named branch. Used internally by `deploy-stage.yml` but also available as a step in any workflow.

```yaml
- uses: bmartins95/aws-shared-pipeline/.github/actions/promote-branch@master
  with:
    target_branch: staging  # or master
```

Requires the job's checkout to use a token with `contents:write` on the target branch.

### `read-platform-ssm`

Reads the standard platform SSM outputs for a stage and sets `GITHUB_OUTPUT`.

```yaml
- uses: bmartins95/aws-shared-pipeline/.github/actions/read-platform-ssm@master
  id: ssm
  with:
    app_name: my-app
    stage: dev
# outputs: pool_id, web_client_id, cognito_domain, backend_url
```

Requires AWS credentials to be configured before this step.

> **Note:** These composite actions work when called directly from an app workflow. They do NOT work when referenced from inside another reusable workflow (GitHub resolves the path against the calling repo's workspace, not this repo). `deploy-stage.yml` inlines their logic as `run:` steps for this reason.

## Adding a new workflow or action

- Workflows must use `on: workflow_call` — not triggered directly
- Keep everything generic: no hardcoded app names, stages, or repo names
- All app-specific values must come through `inputs` or `secrets`
- Inline logic in reusable workflows rather than calling composite actions from within them (cross-repo composite action resolution issue)
- Update this file when changing any public interface

## Starting a new app

Copy `templates/deploy.yml` to `your-app/.github/workflows/deploy.yml`, fill in the `app_name` and stack options, and implement `scripts/ci-deploy.sh` + `scripts/ci-build-web.sh`.
