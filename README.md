Org-wide Claude Code agent. Auto-reviews PRs, auto-fixes issues, and
auto-merges clean PRs across all superhumn repos. The workflow at
`.github/workflows/claude-agent.yml` is applied to every repo via an
organization repository ruleset, so no per-repo enablement is needed.

## What it does

- **On PR open / sync**: reviews the diff, pushes fixup commits for any
  problems it finds, and squash-merges (via `gh pr merge --auto`) once the
  review is clean and required checks pass.
- **On PR approval by a human**: re-runs and squash-merges the PR if it's
  clean and CI is green.
- **On issue open**: implements the fix on a `claude/issue-<n>` branch,
  opens a PR linking the issue, and enables auto-merge so the PR squash-
  merges itself once required checks pass.
- **On @claude mention, inline review comment, or "changes requested"
  review**: reads the feedback, pushes a fixup commit if it's actionable,
  or replies asking for clarification.
- **On CI failure on a `claude/*` branch**: inspects the failing check
  runs, pushes a fix (or re-runs the failed jobs if it's flaky). Capped at
  3 retries per PR via `claude-ci-retry-N` labels.
- **On push to default branch and daily at 06:00 UTC**: walks open
  `claude/*` PRs, calls `gh pr update-branch` on ones that are simply
  behind, and invokes Claude to rebase/resolve any with conflicts.
- **On every PR Claude opens**: `github-actions[bot]` auto-approves it,
  so the single-approval branch-protection rule is satisfied without a
  human. (For repos that also require CODEOWNERS approval, add the
  Claude App to the branch-protection "allowed to bypass" list.)
- **On PRs from Dependabot, Renovate, or Mend**: reviewed and auto-
  merged through the same path as human PRs.
- **Daily issue sweep**: picks up to `CLAUDE_ISSUE_SWEEP_MAX` (default 3)
  oldest unassigned issues without a `claude/issue-<n>` branch and
  without the `claude-no-fix` label and tries to fix them. Issues Claude
  can't confidently handle get a comment plus a `claude-no-fix` label so
  they aren't re-picked.
- **Daily janitor**: closes stale `claude/*` PRs — anything untouched
  for `CLAUDE_PR_STALE_DAYS` days (default 14), or with the
  `claude-ci-retry-3` label and no progress for
  `CLAUDE_PR_EXHAUSTED_DAYS` days (default 3).

Auto-merge always goes through GitHub's `--auto` flag, so branch protection
and required status checks remain authoritative. Claude never bypasses
branch protection. Each repo runs through `CLAUDE.md` and `AGENTS.md` (if
present) for repo-specific conventions before Claude makes changes.

## Configuration

Org variables (all optional, all visibility: all repos):

- `CLAUDE_APP_ID` — numeric ID of the GitHub App used for Claude's pushes.
  Without it the workflow falls back to `GITHUB_TOKEN` and disables
  auto-merge (because GITHUB_TOKEN pushes can't trigger downstream CI, so
  `--auto` would wait forever).
- `CLAUDE_DAILY_RUN_CAP` — max Claude Agent workflow runs per repo per
  UTC day. Defaults to `50`. When reached, the preflight job logs a
  warning and downstream jobs skip.
- `CLAUDE_PR_LOC_CAP` — max LOC changed before the PR review is skipped.
  Defaults to `5000`. Skipped PRs get a one-time comment explaining why.
- `CLAUDE_ISSUE_SWEEP_MAX` — max number of stale issues the daily sweep
  tries to fix per repo per run. Defaults to `3`.
- `CLAUDE_PR_STALE_DAYS` — days of inactivity before the janitor closes
  a `claude/*` PR. Defaults to `14`.
- `CLAUDE_PR_EXHAUSTED_DAYS` — days of inactivity before the janitor
  closes a `claude/*` PR whose CI has hit `claude-ci-retry-3`. Defaults
  to `3`.

## Activation checklist

To turn this on org-wide, an org admin must:

1. **Org secret — `ANTHROPIC_API_KEY`** (already set, visibility: all repos).
2. **GitHub App for Claude commits** *(strongly recommended — without it,
   Claude's commits and PRs use the default `GITHUB_TOKEN`, which cannot
   trigger downstream CI workflows)*:
   - Create a GitHub App in the org with these repo permissions:
     `contents: write`, `pull-requests: write`, `issues: write`,
     `metadata: read`.
   - Install it on all org repos.
   - Add an org variable `CLAUDE_APP_ID` (visibility: all repos) with the
     app's numeric ID.
   - Add an org secret `CLAUDE_APP_PRIVATE_KEY` (visibility: all repos) with
     the app's PEM private key.
   - The workflow auto-detects `CLAUDE_APP_ID`; if unset it falls back to
     `GITHUB_TOKEN` and disables auto-merge.
3. **Org repository ruleset** that enforces this workflow as required across
   all repos:
   - Org Settings → Repository → Rulesets → New ruleset → Required workflows.
   - Target: all repositories (or a `~ALL` include pattern; exclude
     `superhumn/.github` itself if desired).
   - Required workflow:
     `superhumn/.github/.github/workflows/claude-agent.yml@main`.
   - Enforcement status: Active.
4. **Merge this branch to `main`** — required workflows are sourced from the
   default branch of `superhumn/.github`.
5. **Smoke test** — open a trivial issue and a trivial PR in a sandbox repo
   and confirm both jobs run end-to-end.

## Behavior on protected branches

If a target repo's PR branch is protected (required reviews, signed commits,
or contributions from a fork), Claude will open a `claude/fixup-<pr-number>`
PR instead of pushing directly.
