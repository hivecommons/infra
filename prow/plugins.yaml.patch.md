# `prow/plugins.yaml` changes for onboarding `hivecommons`

Apply these edits to `prow/plugins.yaml` in `kubestellar/infra` (the source of truth for
the `plugins` ConfigMap). Stanzas are modeled on the existing `kubestellar` entries.

## 1. `triggers` — trust hivecommons org members

```yaml
triggers:
  - repos:
      - kubestellar   # (existing)
      - hivecommons   # <-- add: org-wide, applies to all repos in the hivecommons org
    only_org_members: true
```

## 2. `approve` — same approval semantics as kubestellar

```yaml
approve:
  - repos:
      - kubestellar   # (existing)
      - hivecommons   # <-- add
    ignore_review_state: true
    require_self_approval: true
```

## 3. `lgtm` — GitHub reviews act as lgtm, tree hash preserved

```yaml
lgtm:
  - repos:
      - kubestellar   # (existing)
      - hivecommons   # <-- add
    review_acts_as_lgtm: true
    store_tree_hash: true
```

## 4. `plugins` — org-wide default plugin set

Mirror of the `kubestellar` org-wide list:

```yaml
plugins:
  # ... existing kubestellar entries ...

  # Org-wide defaults - applies to ALL repos in the hivecommons org
  hivecommons:
    plugins:
      - approve
      - assign
      - branchcleaner
      - dco
      - hold
      - label
      - lgtm
      - lifecycle
      - milestoneapplier
      - override
      - owners-label
      - retitle
      - size
      - skip
      - trigger
      - verify-owners
      - wip
```

Do **not** add `config-updater` for any hivecommons repo — the Prow config's source of
truth stays in `kubestellar/infra` for now (only that repo may carry `config-updater`).

## 5. `external_plugins` — needs-rebase and cherrypicker

```yaml
external_plugins:
  # ... existing kubestellar entry ...
  hivecommons:            # <-- add
    - name: needs-rebase
      events:
        - pull_request
    - name: cherrypicker
      events:
        - issue_comment
        - pull_request
```

## Not included (deliberately)

- `require_matching_label` (needs-kind / needs-triage enforcement) is scoped to
  `kubestellar/kubestellar` only; adopt for hivecommons later if wanted.
- `cherry_pick_unapproved` is global (branch-regexp based) and needs no org entry.
- `owners.filenames` overrides only apply to downstream forks; not needed.
