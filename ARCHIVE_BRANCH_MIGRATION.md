# Proposal: Split archive data onto a dedicated `archive` branch

Follow-up to the discussion in issue #3 ("maybe the generated artifacts themselves
could live off another branch that's not main").

This document is a complete runbook for the split. Estimated hands-on time: **1–2 hours**,
plus coordination (pausing the cron trigger, notifying fork owners).

## Why, and why now

Current state of the repo:

- ~580 commits on `main`, of which ~486 (84%) are `[Auto] API update` — actual code
  changes are buried in the commit list.
- Repo size is ~423 MB and growing at roughly ~1 GB/year. The working tree carries
  ~853 MB of `output/` (817 MB of it `output/raw/`).
- The planned bulletin archiving (issue #3) adds banner images and more JSON, which
  accelerates growth.

The cost of this migration scales with adoption: today there are **2 forks, 0 open PRs,
and no known external consumers of the raw data** (the only consumer is this repo's own
`pages-v2` website, which we control). It will never be cheaper to do than right now.

## End state

| Branch | Contents | History |
|---|---|---|
| `main` (default) | Code only: `src/`, `pages-v2/`, workflows, configs. `output/` is gitignored. | Rewritten: ~90–100 code commits, a few MB total. |
| `archive` | The data: `output/` (same layout as today). | The **complete pre-rewrite history of `main`**, unchanged — every existing commit SHA remains reachable. New `[Auto] API update` commits land here. |

Three design choices keep this cheap:

1. **`archive` keeps the exact current layout** (`output/` at the repo root). Raw URLs
   only change their branch segment: `refs/heads/main/output/...` →
   `refs/heads/archive/output/...`. The website fix is a one-string find-and-replace.
2. **`archive` is simply a ref to today's `main` tip** — no rewrite on the data side.
   This means (a) it doubles as a full backup of the pre-rewrite state, and (b) old
   commit SHAs linked in issues/PRs keep resolving, because those commits stay
   reachable via `archive`.
3. **No archiver code changes.** The archive command already accepts `--output-dir`
   (default `./output`), and the `verified-bot-commit` action already supports
   committing to another branch (`ref:`) from a subdirectory checkout (`workspace:`).

---

## Phase 0 — Preparation

1. **Pause the external cron.** The workflow is fired via `repository_dispatch` from an
   external scheduler (cron-job.org). Disable that job first — an auto-commit racing the
   force-push would be destroyed or cause conflicts.
2. **Confirm no open PRs** (there are none at time of writing). Any in-flight local
   branches will need a trivial rebase afterwards (see Phase 5).
3. **Take a belt-and-braces backup** (the `archive` branch itself preserves everything,
   but this is free insurance):
   ```bash
   git clone --mirror https://github.com/daydreamer-json/ak-endfield-api-archive.git ef-api-backup.git
   ```
4. **Check branch protection on `main`.** If force pushes are blocked, temporarily allow
   them (Settings → Branches), or plan to delete/recreate the rule.
5. **Install git-filter-repo** (the tool the git project recommends over
   `filter-branch`/BFG):
   ```bash
   pip install git-filter-repo   # or: brew install git-filter-repo
   ```

## Phase 1 — Create the `archive` branch (no rewrite, instant)

Point a new branch at the current tip of `main` on the server:

```bash
git fetch origin
git push origin origin/main:refs/heads/archive
```

From this moment, all history and data are safe on `archive` regardless of what happens
to `main`.

## Phase 2 — Rewrite `main` to code-only

filter-repo insists on a fresh clone (deliberately, as a safety measure):

```bash
git clone https://github.com/daydreamer-json/ak-endfield-api-archive.git rewrite-work
cd rewrite-work
git filter-repo --path output --invert-paths
```

This removes `output/` from every commit in history. The ~486 auto-commits become empty
and are pruned automatically. Sanity checks before pushing anything:

```bash
git log --oneline | wc -l        # expect roughly 90–100 (was ~580)
git log --oneline -- output      # expect no results
du -sh .git                      # expect a few MB (was ~420 MB)
```

Do **not** push yet — Phase 3 changes go into the same push so the workflow and website
never point at a layout that doesn't exist.

## Phase 3 — Update code on the rewritten `main`

All of the following as one commit (or a few) on the rewritten `main` in `rewrite-work`:

### 3a. Gitignore the generated data

```gitignore
# .gitignore
output/
```

(Local runs of the archiver still write to `./output` by default; it just no longer
belongs on `main`.)

### 3b. Retarget the workflow

Replace `.github/workflows/main.yml` with the version below. The changes are marked
`# CHANGED`; the structure (retry, two commit phases, verified-commit with plain-git
fallback) is untouched.

```yaml
name: Auto API update check
permissions:
  actions: write
  checks: write
  contents: write
on:
  workflow_dispatch:
  repository_dispatch:
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository (code, main)
        uses: actions/checkout@v4
      - name: Checkout archive branch (data)          # CHANGED: second checkout
        uses: actions/checkout@v4
        with:
          ref: archive
          path: data
      - name: Setup Bun
        uses: oven-sh/setup-bun@v2
      - name: Install dependencies
        run: bun install
      - name: Configure git
        working-directory: data                       # CHANGED: config the data repo
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
      - name: Create config auth
        env:
          CONFIG_AUTH_YAML_CTX: ${{ secrets.CONFIG_AUTH_YAML }}
        run: |
          mkdir -p config
          echo "$CONFIG_AUTH_YAML_CTX" > config/config_auth.yaml
      - name: Run archive script
        uses: nick-fields/retry@v3
        with:
          max_attempts: 5
          timeout_minutes: 10
          command: bun run src/main.ts archive --output-dir data/output   # CHANGED
      - name: Format output folder
        run: bun x oxfmt data/output                  # CHANGED
      - name: Git commit and push
        id: commit_1
        continue-on-error: true
        uses: iarekylew00t/verified-bot-commit@v2
        with:
          ref: refs/heads/archive                     # CHANGED: commit to archive
          workspace: ${{ github.workspace }}/data     # CHANGED: from the data checkout
          message: "[Auto] API update"
          files: |
            output/akEndfield/**
            output/raw/**
            output/mirror_file_list_pending.json
            output/mirror_file_res_list_pending.json
            output/mirror_file_res_patch_list_pending.json
          if-no-commit: info
      - name: Git commit and push (Fallback)
        if: steps.commit_1.outcome == 'failure'
        working-directory: data                       # CHANGED
        run: |
          git add output/akEndfield/** output/raw/** output/mirror_file_list_pending.json output/mirror_file_res_list_pending.json output/mirror_file_res_patch_list_pending.json
          git commit -m "[Auto] API update" || echo "No changes to commit"
          git push
      - name: Run GitHub mirror upload
        run: bun run src/main.ts ghMirrorUpload --output-dir data/output   # CHANGED
        continue-on-error: true
      - name: Format output folder
        run: bun x oxfmt data/output                  # CHANGED
      - name: Git pull
        working-directory: data                       # CHANGED
        run: git pull
      - name: Git commit and push
        id: commit_2
        continue-on-error: true
        uses: iarekylew00t/verified-bot-commit@v2
        with:
          ref: refs/heads/archive                     # CHANGED
          workspace: ${{ github.workspace }}/data     # CHANGED
          message: "[Auto] API update"
          files: |
            output/mirror_file_list.json
            output/mirror_file_list_pending.json
            output/mirror_file_res_list.json.zst
            output/mirror_file_res_list_pending.json
            output/mirror_file_res_patch_list.json.zst
            output/mirror_file_res_patch_list_pending.json
          if-no-commit: info
      - name: Git commit and push (Fallback)
        if: steps.commit_2.outcome == 'failure'
        working-directory: data                       # CHANGED
        run: |
          git add output/mirror_file_list.json output/mirror_file_list_pending.json output/mirror_file_res_list.json.zst output/mirror_file_res_list_pending.json output/mirror_file_res_patch_list.json.zst output/mirror_file_res_patch_list_pending.json
          git commit -m "[Auto] API update" || echo "No changes to commit"
          git push
```

Notes:
- The `archive` checkout defaults to `fetch-depth: 1` — the job only needs the tip, so
  CI stays fast no matter how large the archive history grows.
- `verified-bot-commit` file globs are resolved relative to `workspace:`, so the
  existing `output/...` patterns work unchanged.

### 3c. Repoint the website

`pages-v2` fetches data at runtime from raw URLs hardcoded to `main`. Swap the branch
segment everywhere:

```bash
grep -rl 'refs/heads/main/output' pages-v2/src | xargs sed -i 's#refs/heads/main/output#refs/heads/archive/output#g'
```

(Roughly 16 occurrences: `pages-v2/src/utils/constants.ts`,
`pages-v2/src/legacy/utils/constants.ts`, and a duplicated URL helper in the
components. Optional follow-up: centralize the duplicated helpers onto the existing
constant so the next move is one line.)

### 3d. Update the README

Add a short section: data lives on the [`archive`](../../tree/archive) branch;
contributors should clone with `git clone --single-branch -b main <url>` (or
`--filter=blob:none`) to skip downloading the archive history.

## Phase 4 — Push and verify

```bash
cd rewrite-work
git remote add origin https://github.com/daydreamer-json/ak-endfield-api-archive.git   # filter-repo removes origin
git push --force origin main
```

Then verify end-to-end **before** re-enabling the cron:

- [ ] `main` on GitHub shows only code commits; repo "Code" tab has no `output/`.
- [ ] `archive` branch shows the full old history with `output/` intact.
- [ ] Manually trigger the workflow (`workflow_dispatch`) → run is green → a new
      `[Auto] API update` commit appears on `archive`, none on `main`.
- [ ] Website redeployed and loading data from the `archive` raw URLs.
- [ ] A fresh `git clone --single-branch -b main` is a few MB and fast.
- [ ] Old commit-SHA links from issues still resolve (they're reachable via `archive`).

## Phase 5 — Cleanup and re-enable

1. **Re-enable the external cron.** (Any data window missed during the migration is
   picked up by the next run — the archiver diffs against what's already stored.)
2. **Restore branch protection** on `main` if it was relaxed.
3. **Notify fork owners** (2 forks) to re-fork or hard-reset: existing forks share no
   history with the rewritten `main`.
4. **Rebase in-flight branches** onto the new `main`. Since all SHAs changed, the
   cleanest way is patch-based:
   ```bash
   git diff old-main...my-feature > feature.patch   # from the old clone
   git checkout -b my-feature origin/main           # in a fresh clone
   git apply --3way feature.patch
   ```
5. Optional tidy-up: one commit on `archive` deleting the now-stale code directories
   (`src/`, `pages-v2/`, configs) and adding a stub README ("this is the data branch;
   code lives on `main`"). Purely cosmetic — the workflow never reads code from
   `archive`. Doing it later (or never) is fine.

## Rollback

- **Before Phase 4:** nothing has changed on the server except the new `archive` ref;
  delete it or leave it, no harm.
- **After Phase 4:** restore the old state with one command — `archive` *is* the old
  `main`:
  ```bash
  git push --force origin refs/heads/archive:refs/heads/main
  ```
  (If the optional Phase 5.5 cleanup commit was already made, push the SHA just before
  it instead, or restore from the Phase 0 mirror backup.)

## Expected questions

**Will the repo get smaller on GitHub?** No — the data history remains, reachable via
`archive`, so total server-side size is unchanged and keeps growing with the data. What
changes: `main`'s history becomes tiny, single-branch clones become instant, and the
commit list becomes readable. If total repo size ever becomes a problem (GitHub warns
around 5 GB), moving `archive` to a dedicated data repo is a follow-up this split makes
trivial — but that's years away at current growth.

**Does a plain `git clone` get faster?** Only with `--single-branch` (or
`--filter=blob:none`), because a default clone fetches all branches including
`archive`. Hence the README note in 3d. GitHub's ZIP download and `actions/checkout`
are per-branch and benefit automatically.

**Why not avoid the rewrite entirely and just branch going forward?** That stops future
pollution but permanently bakes the existing ~420 MB into `main`'s history. Since the
rewrite's cost (re-forks, re-clones) is at its minimum right now, doing it properly once
is worth the extra hour.

**Ordering vs the bulletin feature (issue #3)?** Land this first. Bulletins add binary
banner assets; they should hit the `archive` branch from day one rather than deepening
`main`'s history before a later rewrite.

---

## Appendix: rehearsing the migration in a private mirror

The whole runbook can be dress-rehearsed end-to-end in an isolated private repo before
touching the real one. Two facts make the rehearsal high-fidelity with zero setup:

- **No secrets are required.** `config/config_auth.yaml` is only read by
  `ghMirrorUpload` (`src/cmds/ghMirrorUpload.ts`), which the workflow already runs with
  `continue-on-error: true`. The `archive` command itself hits public endpoints. So in
  a repo with no secrets configured, the workflow genuinely fetches live API data and
  commits it to the `archive` branch, while the mirror-upload step fails harmlessly —
  which is desirable, since a rehearsal should not push to the real mirror repos.
- **A private repo beats a fork for this.** Forks of public repos cannot be made
  private, and rehearsing a history rewrite on a fork risks disturbing real in-flight
  branches. A throwaway private mirror is invisible, isolated, and deletable.

### Procedure

1. **Create a private repo** (web UI), e.g. `<you>/ak-endfield-migration-test`, empty —
   no README/license. If your PAT is fine-grained and scoped per-repo, grant it access
   to the new repo (or use SSH / `gh auth`).

2. **Seed it with the real history:**
   ```bash
   git clone --bare https://github.com/daydreamer-json/ak-endfield-api-archive.git seed
   cd seed
   git push https://github.com/<you>/ak-endfield-migration-test.git main:main
   cd .. && rm -rf seed
   ```

3. **Run the runbook against the test repo**, substituting its URL everywhere:
   Phase 1 (create `archive`), Phase 2 (filter-repo rewrite), Phase 3 (workflow +
   `.gitignore`; the website/README edits can be included or skipped), Phase 4
   (force-push). Skip Phase 0 — there is no cron, no forks, and nothing to protect.

4. **Trigger the workflow** from the Actions tab (`workflow_dispatch`; Actions are
   enabled by default on repos you create — private-repo runners draw from the free
   minutes quota, a few minutes per run). Expected result:
   - Run is green; the "Run GitHub mirror upload" step shows a swallowed failure
     (missing auth) — expected.
   - A new `[Auto] API update` commit appears on `archive`, none on `main`.
   - Run it twice: the second run should end with "no changes" (`if-no-commit: info`)
     unless the upstream API actually changed between runs.

5. **Verify the rest of the Phase 4 checklist** against the test repo: rewritten
   `main` commit count and size, `git clone --single-branch -b main` speed, and that
   `https://raw.githubusercontent.com/<you>/ak-endfield-migration-test/refs/heads/archive/output/...`
   serves files (private repos require auth for raw URLs — checking via the repo's
   Code tab on the `archive` branch is equivalent).

6. **Optional website check:** run `pages-v2` locally with its base-URL constants
   temporarily pointed at the test repo's owner/repo/branch to confirm the site renders
   from `archive`-branch URLs.

7. **Clean up:** delete the test repo.

### Local-only spot checks (no repo needed)

- **Rewrite dry run:** clone into a scratch dir, run
  `git filter-repo --path output --invert-paths`, and inspect the resulting history
  (commit count, `du -sh .git`) without pushing anywhere.
- **`--output-dir` flag:** `bun run src/main.ts archive --output-dir /tmp/ef-test-output`
  confirms the archiver writes to a relocated data directory, which is exactly how the
  updated workflow invokes it.
