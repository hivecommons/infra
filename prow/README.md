# Prow Onboarding for the `hivecommons` Org

This directory documents how the existing Prow instance (the one that serves the
`kubestellar` org today) is laid out, and exactly what has to change to onboard the
`hivecommons` org. **Nothing in this directory is applied automatically** — the config
changes and the GitHub App installation are operator actions (see the checklist at the
bottom).

## Current Prow Topology

Prow runs on a dedicated cluster ("the prow cluster"). Build jobs execute in a separate
build cluster; with few exceptions no ProwJobs run on the prow cluster itself.

### Components (namespace `prow`)

| Component | Role |
|---|---|
| `hook` (x2) | Receives GitHub webhook events, dispatches plugins |
| `deck` / `deck-public` | Web UI (private and public instances) |
| `tide` | Merge automation (lgtm/approved label-driven merge queue) |
| `prow-controller-manager` | Runs ProwJobs (plank) |
| `crier` | Reports job status back to GitHub |
| `sinker` | Garbage-collects finished pods/ProwJobs (pods 2h, ProwJobs 7d) |
| `horologium` | Triggers periodic jobs |
| `branchprotector` | CronJob (every ~6h) applying `branch-protection` config to GitHub |
| `statusreconciler` | Reconciles required status contexts on config change |
| `ghproxy` | Caching GitHub API proxy (all API traffic goes through it first) |
| `needs-rebase`, `cherrypicker` | External plugins |
| `minio` + `gcsweb` / `gcsweb-public` | S3-compatible job-log storage (`s3://prow-logs`) and log browser |

Job pods run in the `test-pods` namespace (`pod_namespace: test-pods`,
`prowjob_namespace: prow`).

### Configuration sources

| ConfigMap (ns `prow`) | Contents | Source of truth |
|---|---|---|
| `config` | `config.yaml` (plank, tide, deck, branch-protection, presets, periodics) | `kubestellar/infra` repo, `prow/config.yaml` |
| `plugins` | `plugins.yaml` (plugin enablement, triggers, approve/lgtm, external plugins) | `kubestellar/infra` repo, `prow/plugins.yaml` |
| `label-config` | `labels.yaml` | `kubestellar/infra` repo, `prow/labels.yaml` |
| `job-config` | Central job definitions, keyed by full path | `kubestellar/infra` repo, `prow/jobs/**/*.yaml` |

The `config-updater` plugin is enabled **only on `kubestellar/infra`**: merging a PR that
touches `prow/*.yaml` there automatically syncs the corresponding ConfigMap. In addition,
`in_repo_config` is enabled for all repos (`"*"`), so individual repos can carry a
`.prow.yaml` with their own presubmits/postsubmits.

### GitHub integration

- Prow authenticates as a **GitHub App** (App ID `1227751`); the private key lives in the
  `github-token` secret on the prow cluster. Webhook events for installed orgs are
  delivered to `hook`.
- API endpoints are tried in order: `http://ghproxy`, then `https://api.github.com`.

### Orgs currently served

Only **`kubestellar`**. Every stanza in the live config (tide queries, merge methods,
trigger/approve/lgtm plugin scopes, branch-protection, external plugins) is scoped to the
`kubestellar` org. `hivecommons` is served by nothing until the changes below are applied.

## Onboarding `hivecommons`

The config diffs are prepared in this directory:

- [`config.yaml.patch.md`](config.yaml.patch.md) — tide merge method + query,
  branch-protection stanzas for `hivecommons`.
- [`plugins.yaml.patch.md`](plugins.yaml.patch.md) — trigger/approve/lgtm scopes,
  org-wide plugin list, external plugins for `hivecommons`.

Because plank's `report_templates` and `job_url_prefix_config` both have `"*"` catch-all
entries, hivecommons jobs will report against the default (private) Deck instance without
any change. Adding hivecommons-specific entries pointing at the public Deck is optional
and can be done later, modeled on the existing per-repo entries.

### Operator checklist (manual actions — do not automate)

1. **Install the Prow GitHub App on the `hivecommons` org.** Use the app's installation
   page (GitHub App ID `1227751` — from the app owner account: Settings → Developer
   settings → GitHub Apps → the Prow app → Install App → select `hivecommons`, all
   repositories). Because Prow authenticates as a GitHub App, installing the App also
   delivers webhook events for the org to `hook` — no separate org-level webhook is
   required. If an org webhook is preferred instead, it must point at the hook endpoint
   and reuse the existing HMAC secret (`hmac-token` on the prow cluster).
2. **Verify event delivery**: open a test issue in a `hivecommons` repo and confirm
   `hook` logs show the event (`kubectl -n prow logs deploy/hook`).
3. **Apply the config changes** by PR to `kubestellar/infra`:
   - Add the stanzas from `config.yaml.patch.md` to `prow/config.yaml`.
   - Add the stanzas from `plugins.yaml.patch.md` to `prow/plugins.yaml`.
   - On merge, the `config-updater` plugin syncs the `config` and `plugins` ConfigMaps
     automatically. (Manual fallback, from a checkout of `kubestellar/infra`:
     `kubectl -n prow create configmap config --from-file=config.yaml=prow/config.yaml
     --dry-run=client -o yaml | kubectl -n prow apply -f -`, and the same pattern for
     `plugins`.)
4. **Create the GitHub team** referenced by branch-protection
   (`hivecommons/hivecommons-admins`) before the branchprotector cron next runs, and add
   maintainers to the `hivecommons` org — the `trigger` plugin is configured with
   `only_org_members: true`, so only org members' PRs auto-trigger jobs.
5. **Sync labels**: ensure the labels Prow depends on exist in `hivecommons` repos
   (`lgtm`, `approved`, `do-not-merge/*`, `needs-rebase`, `dco-signoff: yes/no`,
   lifecycle and size labels). Reuse `prow/labels.yaml` from `kubestellar/infra` with a
   `label_sync` run scoped to the new org, or create them manually.
6. **Add `OWNERS` files** to each `hivecommons` repo (root at minimum: `approvers:` +
   `reviewers:`), following the upstream
   [OWNERS spec](https://www.kubernetes.dev/docs/guide/owners/). The `approve`,
   `blunderbuss`-style assignment, and `verify-owners` plugins all key off these.
7. **Add job definitions**: either a `.prow.yaml` in each `hivecommons` repo
   (`in_repo_config` is already enabled for `"*"`) or centrally under
   `prow/jobs/hivecommons/...` in `kubestellar/infra`.
8. **End-to-end verification**: open a test PR in a `hivecommons` repo, confirm
   `/lgtm` + `/approve` from a non-author org member adds labels, tide picks the PR up on
   its status page, and the merge lands with the configured merge method.
