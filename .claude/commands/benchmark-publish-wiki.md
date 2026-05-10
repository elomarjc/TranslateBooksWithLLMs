# Republish the v2 wiki from current submissions

Aggregate every submission in `benchmark/data/submissions/`, regenerate the
wiki locally, and push it to the wiki repo. Use this after adding new
submissions on `main` when the auto `publish-wiki.yml` workflow hasn't fired
or is failing (typically when `WIKI_PUSH_TOKEN` isn't configured).

**No args.**

The skill is **idempotent**: re-running with no new submissions just confirms
"no changes" and exits cleanly.

---

## Step 1 — Sanity check the working tree

Run via Bash from the repo root:

```
git branch --show-current && git status -s | head -20
```

If on a feature branch, ask the user whether to switch to `main` first
(`AskUserQuestion`). The wiki should reflect what's on `main`, not on a
feature branch.

If there are uncommitted submission files in `benchmark/data/submissions/`,
list them and ask the user whether they want to commit them now or proceed
with what's already on remote.

---

## Step 2 — Validate all submissions

```
python scripts/validate_submission.py benchmark/data/submissions/*.json
```

If any submission fails the schema, **stop**. Surface the failing file and
the schema error to the user. Don't republish a wiki built from an invalid
submission.

---

## Step 3 — Aggregate submissions

```
python -m benchmark.cli aggregate-submissions --run-id aggregated --output benchmark_results/aggregated.json --allow-empty
```

Capture the printed stats:
- `Submissions: N`
- `Raw results: M`
- `Aggregated results: K`
- `Conflicts (>=2 obs): C` — number of triples with multiple observations
- `Models: P`, `Languages: Q`

If `Submissions: 0`, stop and tell the user there's nothing to publish.

---

## Step 4 — Generate the wiki locally (clean dir)

```
rm -rf wiki/ && python -m benchmark.cli wiki aggregated
```

The fresh `rm -rf` is important — leftover files from earlier runs (e.g.
languages tested but no longer in submissions) would otherwise survive and
publish stale pages.

Confirm the output:

```
ls wiki/
```

Expected: `Home.md`, `All-Languages.md`, `All-Models.md`, plus one
`Language-<name>.md` per target language and one `Model-<id-slug>.md` per
benchmarked model.

---

## Step 5 — Clone wiki + sync content

Derive the wiki URL from the current repo's `origin`:

```
WIKI_URL=$(git remote get-url origin | sed 's/\.git$//').wiki.git
rm -rf .wiki_repo_archive && git clone "$WIKI_URL" .wiki_repo_archive
```

(On Windows, use the PowerShell equivalent or run the two commands manually.)

If the clone fails ("repository not found"), inform the user that the wiki
needs at least one page created via the GitHub UI before automated tools can
clone it.

Once cloned, sync v2 content while preserving archive pages:

```
cd .wiki_repo_archive && \
  find . -maxdepth 1 -name 'Language-*.md' ! -name 'Archive-*' -delete && \
  find . -maxdepth 1 -name 'Model-*.md' ! -name 'Archive-*' -delete && \
  rm -f Home.md All-Languages.md All-Models.md && \
  cp ../wiki/*.md .
```

The `! -name 'Archive-*'` exclusion preserves the v1 archive that lives at
`Archive-Home.md`, `Archive-Language-*.md`, etc.

---

## Step 6 — Commit and push to the wiki repo

```
cd .wiki_repo_archive && git add -A && git status --porcelain
```

If the porcelain output is empty, the wiki is already up to date. Tell the
user "no changes" and skip the push.

Otherwise commit and push:

```
git commit -m "Publish v2 benchmark wiki: <N> models, <Q> languages" && git push
```

Replace `<N>` and `<Q>` with the actual values from Step 3's stats.

If the push fails:
- **403/permission denied** → the user needs to authenticate. Tell them to
  ensure `git push` works from this clone interactively, or to set up a
  credential helper.
- **non-fast-forward** → someone else pushed to the wiki between Step 5 and
  now. Re-run from Step 5.

---

## Step 7 — Cleanup local artefacts

From the repo root:

```
rm -rf wiki/ benchmark_results/aggregated.json
```

Don't `rm -rf .wiki_repo_archive` — Windows sometimes refuses while the
shell still references it. Leave it; it's gitignored and will be
overwritten next time.

---

## Step 8 — Report

Tell the user:

- The number of models and languages now live on the wiki.
- Whether `Conflicts >= 2 obs` was non-zero (means median aggregation
  kicked in for some triples).
- The wiki URL (derive from origin: `<origin-url-without-.git>/wiki`).
- A reminder, if `WIKI_PUSH_TOKEN` isn't set in the repo secrets, that
  configuring it would automate this step in CI.

---

## Important guardrails

- **Never delete `Archive-*` files.** The v1 archive is kept indefinitely.
  All cleanup globs in this skill are written to exclude `Archive-*`.
- **Never push to the wiki without `git status --porcelain`** confirming
  there are changes. Empty commits pollute the wiki history.
- **Never run from a feature branch** unless explicitly authorized — the
  wiki should always reflect `main`.
- **Don't hardcode model counts in commit messages** — read them from the
  aggregator output.
