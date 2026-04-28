# Fork To Upstream MR

Use this guide when the local checkout is a fork but the merge request should target the upstream repository.

## Workflow

1. Run the skill preflight checks.
2. Inspect remotes:

```bash
git remote -v
```

3. Confirm the current branch:

```bash
git branch --show-current
```

4. Make sure the branch is pushed to the fork remote.
5. Create the MR against the upstream repo:

```bash
glab mr create --title "feat: short summary" --target-branch master --repo front-end/afsui
```

## Important

- Do not pass `namespace/project:branch` to `--source-branch`.
- Let `glab` use the current branch by default unless there is a clear reason to override it.
- Verify that `--repo` points to the upstream project, not the fork.

## Sanity Checks

Before creating the MR, confirm:

- source branch is the intended local branch
- upstream repo path is correct
- target branch name is correct for the upstream project