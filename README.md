# mwt-renovate-config

Shared Renovate presets for TheWeatherCompany. Currently holds one preset,
`otel-collector`, which keeps OpenTelemetry Collector version pins in sync
across every repo that deploys or installs the collector. Also runs a
self-hosted Renovate workflow (`.github/workflows/renovate.yml`) for the
`mwt-ee-*` repos - see "Self-hosted Renovate for CodeArtifact repos" below.

## Self-hosted Renovate for CodeArtifact repos

The org-wide hosted Renovate GitHub App can't authenticate to our private
CodeArtifact npm registry (`engage-npm`) - it runs on Mend's infrastructure
with no AWS credentials, and CodeArtifact tokens are STS-based with a
~12h max TTL, so a static secret would go stale with no way for the
hosted app to refresh it. Every Renovate PR against `mwt-ee-echo`,
`mwt-ee-dispatcher`, `mwt-ee-helios`, `mwt-ee-sonar`, and
`mwt-ee-engage-shared` was resolving npm packages via `registry.npmjs.org`
instead, silently drifting the lockfile's `resolved:` URLs away from what
CI/CD actually installs from.

`.github/workflows/renovate.yml` self-hosts Renovate for just those five
repos instead: each run (every 6h, or on demand) assumes
`arn:aws:iam::372500853281:role/renovate-codeartifact-token` via OIDC (a
role scoped to only `codeartifact:GetAuthorizationToken`/
`GetRepositoryEndpoint`/`ReadFromRepository` on the `engage`/`engage-npm`
domain/repository - deliberately not reusing `mwt-ee-deploy`'s
`github-actions-deploy` role, which also carries Terraform state, ECR, and
IAM management permissions this doesn't need), fetches a fresh token via
`aws codeartifact login --tool npm`, and runs `renovatebot/github-action`
with that token already live in `~/.npmrc`. Renovate shells out to real
`npm`/`yarn` commands to regenerate lockfiles, so it picks the token up
the same way any local `npm install` would.

**Two prerequisites this workflow does not (and cannot) set up on its
own:**

1. `secrets.RENOVATE_GITHUB_TOKEN` - a PAT (or fine-grained token) with
   `contents:write` + `pull-requests:write` on all five repos above. The
   default `GITHUB_TOKEN` is scoped to only this repo.
2. The hosted Renovate GitHub App must be uninstalled/excluded for those
   five repos, or both it and this workflow will run and open duplicate
   PRs. That's an org-level GitHub App installation setting - needs org
   admin access to change.

## Background

An audit of every place TheWeatherCompany pins an OTel Collector version
found six distinct deployment mechanisms, only one of which was automated
end to end (`mwt-observability`'s own EC2 fleet, via `cmd/observability-ops`
and `.github/workflows/update-observability.yml`). Everywhere else, bumps
were 100% manual and had already drifted — four repos were stuck on a
stale `0.132.0` while the rest had moved to `0.150.1`+.

This preset extends the org's existing, already-active Renovate footprint
(30+ repos already run it) to the mechanisms that had no automation,
instead of building new fan-out infrastructure. See:

- [mwt-observability#47](https://github.com/TheWeatherCompany/mwt-observability/pull/47) —
  tags an `otel-collector-vX` GitHub release once a version's RPM is
  actually mirrored to S3, giving this preset's gated custom datasource a
  safe signal to key off.
- [mwt-ee-infra-live#13](https://github.com/TheWeatherCompany/mwt-ee-infra-live/pull/13) —
  removed a stale, superseded copy of the EKS otel-collector Terraform
  module so it isn't accidentally onboarded alongside the live one.

## Onboarding a repo

Add to (or create) the repo's `renovate.json`:

```json
{
  "extends": ["github>TheWeatherCompany/mwt-renovate-config:otel-collector"]
}
```

If the repo doesn't already extend `config:recommended` or similar, extend
this repo's `default` preset too:

```json
{
  "extends": [
    "github>TheWeatherCompany/mwt-renovate-config",
    "github>TheWeatherCompany/mwt-renovate-config:otel-collector"
  ]
}
```

## What this preset covers

| Mechanism | Repo | File(s) | Datasource | Gated? |
|---|---|---|---|---|
| Helm chart version | `mwt-ee-deploy` | `terraform/compute/services/otel-collector/environments/{dev,stg,prod}.tfvars` (`helm_chart_version`) | `helm` (public) | No |
| Collector image tag | `mwt-ee-deploy` | same tfvars (`collector_image_tag`) | `docker` (public) | No |
| Per-app EC2 agent config | `mwt-config-api`, `mwt-forecasteditor-api`, `mwt-forecasteditor-ui`, `mwt-widgets`, `mwt-headlines`, `mwt-headlines-api`, `wsi-media-push`, `wsi-media-subscriptions-api`, `wsi-media-subscriptions-reader`, `wsi-media-notification-tracking-reader`, `mwt-push-precipgrid` | `**/otel-*.json` (`collector_version`) | `github-releases` on `TheWeatherCompany/mwt-observability` | **Yes** — only proposes a version once its RPM is mirrored to S3 |
| Perforce EC2 agent | `mwt-perforce-deploy` | `modules/perforce-server/variables.tf` (`otel_collector_version` default) | same gated datasource | **Yes** |

Each of the repos in row 3/4 needs its own onboarding PR adding the
`extends` block above — not done yet, this repo only holds the shared
preset. Ping the observability channel before onboarding widely; the first
run should be watched.

## Prerequisite

The `github-releases` manager reads from `TheWeatherCompany/mwt-observability`,
which is a **private** repo. Whatever Renovate identity runs across the org
(GitHub App install or PAT) needs read access to it, or the gated managers
will silently find no releases and never propose an update.

## What's intentionally NOT covered here

- `mwt-ee-infra-live`'s old otel-collector Terraform copies — deleted
  (superseded/stale), not onboarded. Onboarding a dead module would just
  resurrect and mask the drift it was removed to fix.
- `wsi-media-web-deploy`'s otel scripts — never committed to `origin/main`;
  nothing to automate.
- `mwt-perforce-deploy`'s second `modules/monitoring` `otel_collector_version`
  variable (separate from the one this preset targets) — looked dead (last
  touched months before the live path), needs a human decision to delete or
  reconcile before any automation touches it.
- Individual EE app repos (dispatcher/echo/helios/sonar/engage-shared) — they
  carry no pinned OTel Collector or SDK version, only Prometheus scrape
  annotations consumed by the cluster-wide collector. Nothing to bump.
- Post-bump health verification — this preset only proposes/patches version
  strings via PR; nothing here confirms the new collector came up healthy
  after a deploy.
