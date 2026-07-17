# Rehearsal runbook: archive-branch migration dress rehearsal

Fully expanded version of the appendix in `ARCHIVE_BRANCH_MIGRATION.md`. Every command
is written out; follow top to bottom on a personal machine with the personal GitHub
account. Placeholder used throughout: `rakuzen25/ef-api-test` — rename
freely.

Expected total time: **30–45 min hands-on**, plus upload/download time for ~420 MB of
git data and a few minutes per workflow run.

Reminders of why this is safe and needs no setup:

- **No secrets go into the test repo.** Do *not* set `CONFIG_AUTH_YAML`. The `archive`
  command needs no auth; `ghMirrorUpload` (the only reader of `config_auth.yaml`) will
  fail on the empty file and is already `continue-on-error: true` — which guarantees
  the rehearsal cannot write to the real mirror repos.
- Everything happens in a **private** repo that is deleted at the end.

---

## 0. Prerequisites (home machine)

```bash
# git-filter-repo (any one of these)
pip install git-filter-repo        # or: brew install git-filter-repo / apt install git-filter-repo
git filter-repo --version          # verify
```

Auth note: pushes in this runbook go to a brand-new private repo under the personal
account, so an **SSH key** on that account is the simplest path — all test-repo URLs
below use `git@github.com:` for that reason. (The fine-grained PAT embedded in the
fork's remote is likely scoped to the fork only and cannot push to a newly created
repo.)

`bun` is only needed for the optional local spot-check in step 9.

## 1. Create the private test repo

Web UI: **New repository → name `ef-api-test` → Private → no README/.gitignore/
license** — it must be completely empty.

## 2. Seed it with the real history

```bash
git clone --bare https://github.com/daydreamer-json/ak-endfield-api-archive.git seed
cd seed

# Record the pre-rewrite tip SHA — used in step 8 to prove old links survive
git rev-parse main | tee ../PRE_REWRITE_SHA.txt

# Push main AND create the archive branch in one go (rehearses Phase 1)
git push git@github.com:rakuzen25/ef-api-test.git \
    main:refs/heads/main main:refs/heads/archive

cd .. && rm -rf seed
```

Then in the test repo's Settings, confirm the **default branch is `main`** (GitHub
usually picks it automatically when both branches arrive together).

## 3. Rewrite `main` to code-only (rehearses Phase 2)

```bash
git clone git@github.com:rakuzen25/ef-api-test.git rewrite-work
cd rewrite-work
git filter-repo --path output --invert-paths
```

Sanity checks before continuing:

```bash
git log --oneline | wc -l     # expect roughly 100–110 (was ~600)
git log --oneline -- output   # expect empty
du -sh .git                   # expect ~100 MB (was ~420 MB) — residual is pages-v2
                              # webfonts + old v1 site assets, not archive data
```

## 4. Apply the code changes (rehearses Phase 3)

Still inside `rewrite-work`:

1. Append `/output/` and `/data/` to `.gitignore` (root-anchored — see § Phase 3a).
2. Replace `.github/workflows/main.yml` with the workflow from
   **`ARCHIVE_BRANCH_MIGRATION.md` § Phase 3b**, verbatim (it contains no hardcoded
   repo names — `verified-bot-commit` defaults to the repo it runs in).
3. Prefix the two `ignorePatterns` in `.oxfmtrc.json` with `**/` (§ Phase 3c) —
   without this, oxfmt errors on the encrypted `index_main.json`/`index_initial.json`
   resource files once they live under `data/output` instead of `output`.
4. Skip the website/README edits (3d/3e) — or include them if rehearsing the full
   diff; they don't affect the workflow test.

```bash
git add -A
git commit -m "Migrate archive data to dedicated archive branch"
```

## 5. Force-push the rewritten main (rehearses Phase 4)

```bash
git remote add origin git@github.com:rakuzen25/ef-api-test.git   # filter-repo removed it
git push --force origin main
```

## 6. Trigger the workflow — run 1

Web UI: **Actions tab → "Auto API update check" → Run workflow → branch `main`**, then
watch the run live from the same page. (If the workflow doesn't appear, check
Settings → Actions isn't disabled.)

Expected result:

- [ ] Run is **green** overall.
- [ ] "Run archive script" genuinely fetches live API data (takes a couple of minutes).
- [ ] "Run GitHub mirror upload" shows a **swallowed failure** (empty auth config) —
      expected and desired.
- [ ] One or two new `[Auto] API update` commits on **`archive`**; **zero** new
      commits on `main` — compare the two branches' commit lists in the web UI
      (`/commits/archive` and `/commits/main` on the repo).

Troubleshooting: if the commit step fails *and* its fallback fails, check the workflow
kept `permissions: contents: write` and that the `archive` checkout step wasn't given
`persist-credentials: false`.

## 7. Trigger the workflow — run 2

Dispatch again immediately. Expected: green run ending in "no changes to commit"
(`if-no-commit: info`) on both commit steps — *or* another small commit if the upstream
API genuinely changed or a response contains volatile fields. Either outcome passes;
the property under test is that commits only ever land on `archive`.

### If every run says "no changes": force a real incremental commit

A freshly seeded test repo already matches the live API (production captured
everything minutes ago), so runs legitimately have nothing to commit. "No changes" is
itself a pass — it proves the archiver sees the existing data through
`--output-dir data/output`; a mis-wired path would have re-archived everything into a
huge commit. To also watch a commit land end-to-end, rewind `archive` to an older
commit — a genuine archiver-produced past state — so the next run does a real
incremental catch-up, the same code path production runs daily:

```bash
# Blob-less bare clone: commit/tree metadata only, a few MB — not the 420 MB of data
git clone --bare --filter=blob:none git@github.com:rakuzen25/ef-api-test.git rewind-tmp
cd rewind-tmp
git push --force git@github.com:rakuzen25/ef-api-test.git archive~20:refs/heads/archive
cd .. && rm -rf rewind-tmp
```

Then dispatch the workflow again from the Actions tab.

Keep the rewind shallow (~10–20 commits). The verified-commit action uploads every
changed file individually through the GitHub API (that's what makes commits
"verified"), so the catch-up commit's duration scales with the rewind depth — a deep
rewind can take 15+ minutes at this step versus ~6 seconds for production's
5-minute increments. If the verified step times out or errors on a big diff, the
plain-git fallback step committing instead is a pass too — that recovery path fires
occasionally in production as well.

Expected: green run, one or two `[Auto] API update` catch-up commits on `archive`,
still zero on `main`. (Prefer this over hand-deleting files — manual removal creates
tree states the archiver never produces, so you'd be rehearsing an artificial
scenario.)

## 8. Verification checklist (rehearses the Phase 4 checks)

```bash
# Rewritten main is tiny and fast to clone
time git clone --single-branch -b main \
    https://github.com/rakuzen25/ef-api-test.git main-only
du -sh main-only/.git         # ~100 MB (vs ~420 MB before) — residual is web fonts

# Old commit SHAs still resolve (reachable via archive)
cat PRE_REWRITE_SHA.txt
# open: https://github.com/rakuzen25/ef-api-test/commit/<that SHA>
```

In the web UI:

- [ ] `main` commit list shows only code commits — no `[Auto] API update`.
- [ ] `main` Code tab has no `output/` folder.
- [ ] `archive` Code tab shows `output/` intact, including the new auto-commit data.
- [ ] The pre-rewrite SHA URL above renders (raw URLs can't be checked on a private
      repo without auth — the Code tab on `archive` is the equivalent check).

**Capture evidence for the owner while it exists:** links/screenshots of the green
runs, the clean `main` history, the auto-commit on `archive`, and the
before/after clone sizes. This turns the proposal from "should work" into "worked".

## 9. Optional extras

- **Rehearse the rollback** (worth the 2 minutes — it's the step you least want to
  improvise on the real repo):
  ```bash
  git push --force origin refs/heads/archive:refs/heads/main   # restore old state
  # verify main once again shows the full old history, then re-apply:
  cd rewrite-work && git push --force origin main
  ```
- **Rehearse the archive tidy-up** (§ Phase 5 subsection): run the `find … git rm`
  command against the test repo's `archive` branch and re-dispatch the workflow — a
  green run afterwards proves the workflow really doesn't depend on any code from the
  data branch.
- **Local archiver spot-check** (no repo needed):
  ```bash
  bun install
  bun run src/main.ts archive --output-dir /tmp/ef-test-output
  ls /tmp/ef-test-output      # akEndfield/, raw/, mirror lists
  ```
- **Website check:** run `pages-v2` locally with its base-URL constants temporarily
  pointed at `rakuzen25/ef-api-test` + branch `archive` (needs the repo
  public or an authenticated fetch — simpler to defer this to the real migration).

## 10. Clean up

Delete the test repo in the web UI: **Settings → Danger Zone → Delete this
repository**. Then locally:

```bash
rm -rf rewrite-work main-only PRE_REWRITE_SHA.txt
```
