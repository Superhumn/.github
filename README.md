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

Auto-merge always goes through GitHub's `--auto` flag, so branch protection
and required status checks remain authoritative. Claude never bypasses
branch protection.

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
     `GITHUB_TOKEN`.
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
