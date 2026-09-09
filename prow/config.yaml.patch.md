# `prow/config.yaml` changes for onboarding `hivecommons`

Apply these edits to `prow/config.yaml` in `kubestellar/infra` (the source of truth for
the `config` ConfigMap). Stanzas are modeled on the existing `kubestellar` entries.

## 1. `tide.merge_method` — add an org-wide default for hivecommons

```yaml
tide:
  merge_method:
    kubestellar: merge  # (existing)
    # ... existing kubestellar/<repo> overrides ...
    hivecommons: squash  # org-wide default for hivecommons
    hivecommons/hive: rebase  # override: see note below (hivecommons/hive#6312)
```

Hive Commons repos default to squash (matching the operator's existing squash-merge
practice for hive/console work); add further per-repo `hivecommons/<repo>: merge` or
`rebase` overrides later if a repo needs a different merge shape.

`hivecommons/hive` is overridden to `rebase` because Tide's squash rebuilds the merge
commit from the PR title/body and the PR author's *login*, not from the branch's signed
commits — the resulting squash commit's `Signed-off-by:` trailer is either dropped or no
longer matches the squash commit's author identity, so v4/v5 fail their post-merge DCO
check (see hivecommons/hive#6312 for two observed failure shapes). Tide's
`merge_commit_template` cannot fix this because it only has PR title/body/author login
available, not the author's DCO-signing email. `rebase` replays each already-signed
branch commit onto the base as-is, so every commit lands with its own valid
`Signed-off-by:` trailer. This requires PR branches to be free of merge conflicts and
`needs-rebase`-clean, which the existing `tide.queries` entry already enforces via
`missingLabels: [needs-rebase, ...]`.

## 2. `tide.queries` — add hivecommons to the org-wide query

The existing query is scoped to `orgs: [kubestellar]`. Since the label requirements are
identical (lgtm + approved, DCO enforced), simply add the new org to the same query:

```yaml
tide:
  queries:
    # Org-wide query with DCO requirement
    - orgs:
        - kubestellar
        - hivecommons        # <-- add
      labels:
        - lgtm
        - approved
      missingLabels:
        - "dco-signoff: no"
        - do-not-merge
        - do-not-merge/hold
        - do-not-merge/invalid-owners-file
        - do-not-merge/work-in-progress
        - needs-rebase
```

(If hivecommons ever needs different merge criteria, split it into its own query block
instead.)

## 3. `branch-protection` — add the hivecommons org

Modeled on the `kubestellar/infra` repo stanza (DCO required, admin-team restriction,
main + release branches). Start with the `infra` repo; add further repos as they are
created in the org:

```yaml
branch-protection:
  orgs:
    kubestellar:
      # ... existing ...
    hivecommons:             # <-- add
      repos:
        infra:
          protect: true
          required_status_checks:
            contexts:
              - dco
          restrictions:
            users: []
            teams:
              - hivecommons/hivecommons-admins
          include:
            - "^main$"
            - "^release-.+$"
```

> The `hivecommons/hivecommons-admins` team must exist before the `branchprotector`
> CronJob next runs, or the run will error for this org.

## 4. Deck URLs (optional, no change required)

`plank.report_templates` and `plank.job_url_prefix_config` both have `"*"` catch-all
entries, so hivecommons jobs report against the default Deck instance with no change.
To surface hivecommons jobs on the public Deck instead, add per-repo entries modeled on
the existing `kubestellar/<repo>` lines.
