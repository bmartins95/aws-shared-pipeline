# aws-shared-pipeline — reusable GitHub Actions workflows

This repo contains reusable GitHub Actions workflows shared across all apps in the platform. It has no application code and no deployment of its own.

## Workflows

### `register-infra.yml`

Registers an app's infrastructure config with `aws-infra` and waits for the platform deployment to complete before the calling pipeline proceeds.

**What it does:**
1. Reads `infra/{stage}.yaml` from the caller's repo
2. Pushes it to `apps/{app_name}/{stage}.yaml` in the `aws-infra` repo via GitHub API
3. Fires a `repository_dispatch` event to trigger the aws-infra pipeline
4. Waits for that run to complete (success or failure)

**Inputs:**

| Input | Required | Description |
|---|---|---|
| `app_name` | yes | App name — matches the folder under `apps/` in aws-infra |
| `stage` | yes | Stage name (`prod`, `stg`, `dev`) — must match a file at `infra/{stage}.yaml` |
| `infra_repo` | no | `owner/repo` of aws-infra. Defaults to `{caller-owner}/aws-infra` |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `infra_github_token` | yes | PAT with `contents:write` + `actions:write` on the aws-infra repo |

**Usage:**

```yaml
register-infra:
  uses: bmartins95/aws-shared-pipeline/.github/workflows/register-infra.yml@master
  with:
    app_name: my-app
    stage: prod
  secrets:
    infra_github_token: ${{ secrets.INFRA_GITHUB_TOKEN }}
```

The `infra_repo` input can be omitted if your app and aws-infra share the same GitHub owner.

## Adding a new workflow

- Workflows here must use `on: workflow_call` — they are not triggered directly
- Keep workflows generic: no hardcoded app names, repo names, or stages
- All app-specific values must come through `inputs` or `secrets`
- Bump the README/AGENTS when adding or changing a workflow interface

## Versioning

Callers pin to `@main`. Breaking changes to workflow inputs/secrets must be backwards-compatible or coordinated across all caller repos before merging.
