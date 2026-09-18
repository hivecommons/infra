# Hive Commons Infra

Shared CI workflows, Prow configuration, and org automation for the **Hive Commons**
(`hivecommons`) GitHub org — the umbrella org for the hive project. The hive codebase
itself lives in [`hivecommons/hive`](https://github.com/hivecommons/hive); this repo
bootstraps the org-level infrastructure that `hivecommons` repos share.

The contents are seeded from [`kubestellar/infra`](https://github.com/kubestellar/infra)
(Apache-2.0) and adapted for this org. Each workflow file carries an attribution header
noting its origin and any changes made.

## Layout

| Path | Contents |
|---|---|
| `.github/workflows/` | Reusable (`workflow_call`) workflows for repos in this org |
| `prow/` | Prow topology documentation and the prepared config patches for onboarding `hivecommons` onto the existing Prow instance |
| `LICENSE` | Apache-2.0 (matches kubestellar) |

## Reusable workflows

Call them from any repo in the org, e.g.:

```yaml
jobs:
  stale:
    uses: hivecommons/infra/.github/workflows/reusable-stale.yml@main
```

| Workflow | What it does | Adapted? |
|---|---|---|
| `reusable-copilot-dco.yml` | Sets the `dco` commit status to success and swaps `dco-signoff` labels on Copilot-authored PRs (Prow's `dco` plugin cannot trust apps) | Yes — DCO link now points at the calling repo's `CONTRIBUTING.md` |
| `reusable-copilot-automation.yml` | Full Copilot PR processing: DCO override, `copilot` label, removes blocking labels | No (verbatim) |
| `reusable-ai-fix.yml` | Assigns the Copilot coding agent to issues labeled `ai-fix-requested` + `triage/accepted`; links resulting PRs back to the issue | No (verbatim) |
| `reusable-stale.yml` | Marks issues stale after N days (default 90) and closes after N more; PRs exempt | No (verbatim) |
| `reusable-scorecard.yml` | OpenSSF Scorecard analysis, SARIF upload to code scanning | No (verbatim) |
| `reusable-spellcheck.yml` | Markdown spellcheck (`rojopolis/spellcheck-github-actions`), configurable config path | No (verbatim) |
| `reusable-image-scanning.yml` | Builds the repo's Dockerfile and scans it with Trivy, SARIF upload | No (verbatim) |
| `reusable-add-help-wanted.yml` | Adds `help wanted` to unassigned issues that lack help labels | No (verbatim) |
| `reusable-label-helper.yml` | Validates `/kind` and `/area` comment commands; handles `/help-wanted`, `/good-first-issue` | Yes — default `valid_areas` list replaced with Hive-oriented areas |
| `reusable-assignment-helper.yml` | Replies to natural-language "assign me" comments with the `/assign` slash-command hint | Yes — KubeStellar resource links removed |
| `reusable-greetings.yml` | Welcomes first-time issue/PR authors (message overridable via inputs) | Yes — default messages rewritten for Hive Commons |
| `reusable-feedback.yml` | Thank-you comment on merged PRs with a feedback link | Yes — defaults to the org discussions page |

## Not yet migrated

Still living in their kubestellar counterparts; migrate when needed:

- **Workflow distribution** — `kubestellar/infra` has a `sync-workflows.yml` +
  `caller-workflows/` mechanism that pushes thin caller stubs to every repo in the org
  (requires a `WORKFLOW_SYNC_TOKEN` org secret). Not migrated yet; hivecommons repos
  should call the reusable workflows here directly until the org is big enough to warrant
  it.
- **Org workflow templates** — `kubestellar/.github` `workflow-templates/` (starter
  CI/CD, release automation, security scan templates). A `hivecommons/.github` repo can
  adopt these later.
- **Org community health files** — issue templates, PR template, code of conduct,
  security policy (`kubestellar/.github`).
- **Prow cluster manifests and images** — `kubestellar/infra` `clusters/`, `images/`,
  `iac/`. The Prow instance itself remains operated from `kubestellar/infra`; this repo
  only documents onboarding (see [`prow/`](prow/)).
- **Prow config source of truth** — remains `kubestellar/infra` `prow/`
  (`config-updater` is bound to that repo). The patches in [`prow/`](prow/) are applied
  *there*, not here.

## Not applicable (kubestellar-specific, intentionally left out)

- `docs-link-checker.yml`, `broken-links-crawler` — kubestellar website/docs link checks.
- `docs-gen-and-push.yml`, `check-docs-pr-preview.yml` — kubestellar docs pipeline.
- `goreleaser.yml`, `golangci-lint.yml`, e2e/integration test workflows in
  `kubestellar/kubestellar` — repo-local, not reusable (`workflow_call`) workflows.
- `create-our-meeting-bi-weekly.yml`, `ocp-self-runner.yml`,
  `test-demo-env-creation-script.yml` — kubestellar community/infra specifics.

## License

Apache-2.0. Portions copyright the KubeStellar Authors.

<!-- prow e2e check 2026-09-01 -->
