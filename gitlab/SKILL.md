---
name: gitlab
description: Use the glab CLI to work with GitLab repositories, merge requests, and GitLab-hosted code. Use when the user mentions GitLab, glab, merge requests, MR review, repo info, or GitLab API workflows.
triggers:
  - gitlab
  - git
  - mr
  - merge request
---

# GitLab Skill

Use this skill for GitLab work that is best handled through `glab`.

## Required References

Before taking action, read the matching reference file:

- **Create an MR in the same repo** -> read `references/create-mr-guide.md`
- **Create an MR from a fork to upstream** -> read `references/fork-to-upstream.md`
- **If anything fails or looks odd** -> read `references/troubleshooting.md`
- **If the task involves pushing a branch** -> read `references/git-push-guide.md`

Do not skip the relevant reference read.

## Preflight Checks

Run these checks in order before GitLab operations:

1. Check `glab` installation:

```bash
which glab && glab --version
```

If missing on macOS, install with:

```bash
brew install glab
```

2. Check auth status for the company GitLab:

```bash
glab auth status --hostname gitlab.xxx.com
```

`glab` can use `GITLAB_TOKEN` from the shell environment, commonly set in `~/.zshrc`.

If a token must be created, use:
`https://gitlab.xxx.com/-/user_settings/personal_access_tokens`

## Common Commands

```bash
# List the MR for the current branch
glab mr list --source-branch="$(git branch --show-current)"

# View MR details
glab mr view <mr_number>

# Create an MR in the current repo
glab mr create --title "feat: description" --description "details"

# Create an MR from a fork to upstream
# Keep --source-branch as the current local branch name
glab mr create --title "feat: description" --target-branch master --repo front-end/afsui

# Show repo info
glab repo view

# Open an MR in the browser
glab mr view <mr_number> --web
```

## Review Guidance

When the user asks for a code review on a GitLab MR:

1. Run the preflight checks.
2. Fetch the MR branch and identify the target branch.
3. Review the diff against the correct base.
4. Report findings first, ordered by severity, with file and line references when possible.
5. Call out missing tests, behavior changes, and rollout risks.

## Guardrails

- Prefer `glab` over raw API calls when both can solve the task.
- Avoid destructive Git commands unless the user explicitly asks.
- For fork workflows, double-check the target repo and branch before creating the MR.
- If auth, remotes, or target branch look wrong, stop and use `references/troubleshooting.md`.