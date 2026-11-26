# UplincIT/CIPP: fork maintenance

This repo is Uplinc's fork of `KelvinTegelaar/CIPP` (branch `main`). It carries a
few fork-only changes that **an upstream sync must never remove**.

## Fork-only state

| What | Why |
| --- | --- |
| `.github/workflows/azure-static-web-apps-brave-tree-02a14ea0f.yml` | Deploys `main` to Uplinc's Azure Static Web App **brave-tree-02a14ea0f**. Upstream's `cipp_frontend_build.yml` skips forks (`if: github.event.repository.fork == false`), so without this file nothing deploys. Uses secret `AZURE_STATIC_WEB_APPS_API_TOKEN_BRAVE_TREE_02A14EA0F`. |
| `.github/workflows/azure-static-web-apps-red-stone-0d8c5ed03.yml` **deleted** | Upstream's own SWA, not Uplinc's. Keep it deleted. |
| `.github/UPLINC-FORK.md` | This file. |
| GitHub setting: workflow **PR Branch Check** (`pr_check.yml`) disabled | Upstream's check auto-closes every PR from a fork into `main`, which includes our own PRs. Leave it disabled (`gh workflow list -R UplincIT/CIPP --all`). |

History: on 2026-11-07 `main` was force-updated to the upstream head (GitHub
"Sync fork → Discard commits"), which deleted the deploy workflow and stopped
deploys until it was restored on 2026-11-25.

## Syncing upstream safely

Never use **Sync fork → Discard commits**, `gh repo sync --force`, or any
`git push --force` to `main`. Those reset `main` to upstream and drop the files above.

1. See how far behind/ahead the fork is:
   ```sh
   gh api repos/UplincIT/CIPP/compare/KelvinTegelaar:CIPP:main...UplincIT:CIPP:main \
     --jq '{ahead:.ahead_by, behind:.behind_by}'
   ```
   `behind: 0` means nothing to do. `ahead` is normally ≥ 1 (our fork-only commits); that is expected.
2. Merge upstream with GitHub's merge API (a normal merge commit, never a reset):
   ```sh
   gh api -X POST repos/UplincIT/CIPP/merge-upstream -f branch=main
   ```
   - `200` = merged (or fast-forwarded). Continue at step 3.
   - `409` = conflict. Do **not** discard. Merge locally on a branch and open a PR:
     ```sh
     git fetch upstream && git switch -c sync/upstream-$(date +%F) origin/main
     git merge upstream/main     # resolve; keep the fork-only state above
     git push -u origin HEAD && gh pr create -R UplincIT/CIPP -B main
     ```
     A likely conflict is upstream editing the red-stone workflow we deleted:
     resolve by keeping it deleted (`git rm` it).
3. Verify the fork-only state survived:
   ```sh
   gh api repos/UplincIT/CIPP/contents/.github/workflows/azure-static-web-apps-brave-tree-02a14ea0f.yml --jq .path
   gh api repos/UplincIT/CIPP/contents/.github/workflows/azure-static-web-apps-red-stone-0d8c5ed03.yml 2>&1 | grep -q 'Not Found' && echo "red-stone still deleted"
   ```
   If the brave-tree file is missing, restore it from history
   (`git log --all -- .github/workflows/azure-static-web-apps-brave-tree-02a14ea0f.yml`)
   through a PR, and report what removed it.

## Checking a deploy ran

Every push to `main` triggers **Azure Static Web Apps CI/CD** from the brave-tree file.

```sh
HEAD=$(gh api repos/UplincIT/CIPP/commits/main --jq .sha)
gh run list -R UplincIT/CIPP --workflow azure-static-web-apps-brave-tree-02a14ea0f.yml \
  --branch main --event push -L 5 --json headSha,status,conclusion,url
```

The newest run's `headSha` must equal `$HEAD` with `conclusion: success`. If it
failed, open the run (`gh run view <id> --log-failed`); the Oryx build on the
SWA side is the usual culprit (e.g. an upstream bump of `engines.node` that the
SWA build image doesn't have yet). A failed run can be retried with
`gh run rerun <id>` or `gh workflow run azure-static-web-apps-brave-tree-02a14ea0f.yml -R UplincIT/CIPP --ref main`.

Never print or change the deploy secret; only check it exists by name:
`gh secret list -R UplincIT/CIPP`.
