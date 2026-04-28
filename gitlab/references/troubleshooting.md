# Troubleshooting

Use this guide whenever a GitLab or `glab` workflow fails, prompts unexpectedly, or returns confusing output.

## glab Not Installed

```bash
which glab && glab --version
```

If missing on macOS:

```bash
brew install glab
```

## Auth Problems

Check auth:

```bash
glab auth status --hostname gitlab.xxx.com
```

If auth is missing or invalid:

- verify `GITLAB_TOKEN` is exported in the shell
- verify the token is valid for `gitlab.xxx.com`
- create a new token if needed: `https://gitlab.xxx.com/-/user_settings/personal_access_tokens`

## Wrong Repo Or Branch

```bash
git remote -v
git branch --show-current
git status --short
```

Check whether:

- the local repo is the intended project
- the current branch is the branch you meant to operate on
- there are uncommitted changes affecting the workflow

## MR Creation Fails

Check whether an MR already exists:

```bash
glab mr list --source-branch="$(git branch --show-current)"
```

If using a fork-to-upstream flow:

- confirm `--repo` points to upstream
- do not use `namespace/project:branch` as `--source-branch`

## Browser/Open Issues

If `--web` does not work, fall back to plain `glab mr view <mr_number>` and share the URL or summary instead.

## When To Stop

Stop and ask for confirmation if:

- the target repo or branch is ambiguous
- auth would require creating or rotating credentials
- the workflow would push, create, or modify resources in the wrong project