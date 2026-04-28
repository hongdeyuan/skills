---
name: fix-jira-issue
description: Fix Jira issues with the jira CLI by checking local prerequisites, fetching issue details and attachments, assessing whether context is sufficient, creating a branch, implementing a minimal code fix, optionally preparing a GitLab MR, and optionally advancing Jira workflow status. Use when the user asks to fix a Jira issue, handle a Jira bug or feature ticket, investigate an issue from a Jira key, or continue from Jira attachments/logs/screenshots.
---

# Fix Jira Issue

Use this skill for Jira-driven implementation work in a local codebase.

## Required References

Read the matching reference file before taking the related action:

- For Jira CLI usage and issue retrieval patterns, read `references/help.md`
- For Jira workflow transitions after the fix is ready, read `references/transitions.md`
- For MR creation in GitLab, also use the `gitlab` skill

## Workflow

### 1. Check prerequisites

Run these checks in parallel:

```bash
jira --version
echo "JIRA_HOST: $JIRA_HOST"
echo "JIRA_API_VERSION: $JIRA_API_VERSION"
echo "JIRA_API_TOKEN set: $([ -n "$JIRA_API_TOKEN" ] && echo 'yes' || echo 'no')"
```

If something is missing:

- If `jira` is not installed, install it with `npm install -g @acring/jira-cli`
- If `JIRA_HOST` or `JIRA_API_VERSION` is missing, add this to `~/.zshrc` or `~/.bashrc`

```bash
export JIRA_HOST=https://issue.xxx.com
export JIRA_API_VERSION=2
```

- If `JIRA_API_TOKEN` is missing, ask the user to create a Personal Access Token at `https://issue.xxx.com/secure/ViewProfile.jspa?selectedTab=com.atlassian.pats.pats-plugin:jira-user-personal-access-tokens` and add it to their shell config before continuing

Do not fake Jira access. Stop and tell the user exactly what is missing.

### 2. Fetch the issue and attachments

Run these in parallel after the environment is ready:

```bash
jira issue view <issue-key>
jira issue attachment download <issue-key> --output ./tmp/
```

Then:

- Read issue summary, description, comments, and visible metadata needed for implementation
- Inspect downloaded files in `./tmp/`, including screenshots, logs, and other attachments
- If the context is insufficient to make a reliable fix, stop and ask the user for the missing details instead of guessing

### 3. Choose the branch name

Pick the branch prefix from the issue type:

- Bug fix -> `fix/<issue-key>`
- New feature -> `feat/<issue-key>`

Create the branch:

```bash
git checkout -b fix/<issue-key>
```

Use the matching prefix when the issue is a feature.

### 4. Implement the fix

Keep the change focused and minimal:

- Locate the relevant code by searching for the failing path, related types, or feature area
- Follow existing code patterns in the repository
- Limit unrelated refactors unless they are required to complete the fix safely
- Validate with the smallest useful test or verification flow available in the repo

### 5. Finish the coding work

After the fix is complete, ask the user whether to:

- Commit the code with `git commit`
- Push and create an MR through the `gitlab` skill
- Clean temporary files in `./tmp/`

When preparing an MR, use this title format:

- Bug fix: `fix(<issue-key>): 简要描述`
- Feature: `feat(<issue-key>): 简要描述`

Use this MR description template:

```markdown
## Summary

- 简要描述本次变更的目的和内容

## Issue

- [<issue-key>](https://issue.xxx.com/browse/<issue-key>)

## Changes

- 具体变更 1
- 具体变更 2

## Test plan

- [ ] 测试用例 1
- [ ] 测试用例 2

🤖 Generated with [Agent]
```

### 6. Optionally advance the Jira workflow

After the MR is ready, ask whether the user wants to move the issue to the next state such as `提交审核`.

If yes:

- Read `references/transitions.md`
- Show the available transitions to the user
- Pre-fill candidate field values from the issue and code changes
- Confirm the final payload with the user before running `jira issue transition`

## Guardrails

- Do not assume Jira configuration if CLI access or environment variables are missing
- Do not guess missing field values, usernames, or version names during transitions
- Do not over-fix unrelated code while handling a Jira issue
- Prefer reading attachments and issue details before touching code
- Use `./tmp/` only as a working directory for downloaded attachments and temporary outputs