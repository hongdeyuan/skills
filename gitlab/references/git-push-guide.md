# Git Push Guide

Use this guide before a workflow that needs to push commits or branches.

## Pre-Push Checks

```bash
git status --short
git branch --show-current
git remote -v
```

Confirm:

- you are on the expected branch
- the remote is correct
- there are no unexpected local changes

## Push Current Branch

```bash
git push -u origin "$(git branch --show-current)"
```

## Fork Workflow Notes

- Push to the fork remote first.
- Then create the MR against upstream.
- If remote names are unusual, inspect `git remote -v` before pushing.

## Safety

- Do not force-push unless the user explicitly asks.
- If branch protection, rejected pushes, or credential prompts appear, stop and surface the exact issue.