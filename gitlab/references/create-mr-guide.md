# Create MR Guide

Use this guide when creating a merge request in the same GitLab repository.

## Checklist

1. Confirm `glab` is installed.
2. Confirm auth with `gitlab.xxx.com`.
3. Confirm the current repo and branch:

```bash
git remote -v
git branch --show-current
git status --short
```

4. Check whether an MR already exists for the branch:

```bash
glab mr list --source-branch="$(git branch --show-current)"
```

5. Create the MR:

```bash
glab mr create --title "feat: short summary" --description "details"
```

## Title Tips

- `feat:` for new functionality
- `fix:` for bug fixes
- `refactor:` for internal cleanup
- Keep the title short and specific

## Description Tips

Include:

- What changed
- Why it changed
- Testing done
- Any known risks or follow-up work

## Before Submitting

Confirm:

- target branch is correct
- branch has been pushed
- working tree is clean enough for the intended push/MR flow