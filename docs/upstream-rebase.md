# Upstream Auto Rebase (`suntianc/paseo` Fork)

`.github/workflows/upstream-rebase.yml` keeps the fork's custom commits on `main` rebased onto `getpaseo/paseo:main`.

## Schedule and Manual Trigger

- **Scheduled**: Runs daily at `01:00 UTC` (`09:00 CST`).
- **Manual (`workflow_dispatch`)**: Trigger from GitHub Actions with optional inputs:
  - `upstream_repo` (default `getpaseo/paseo`)
  - `upstream_branch` (default `main`)
  - `base_branch` (default `main`)
  - `sync_mode`: `update-base` (default, backs up and updates `base_branch`) or `branch-only` (only pushes the rebased branch).

## How It Works

1. Fetches `origin/<base_branch>` and `upstream/<upstream_branch>`.
2. Exits immediately if `upstream/<upstream_branch>` is already an ancestor of `origin/<base_branch>`.
3. Checks out `rebase/upstream-<YYYYMMDD>-<upstream_short_sha>` from `origin/<base_branch>` and runs `git rebase --empty=drop upstream/<upstream_branch>`:
   - **Clean rebase**:
     - Pushes `rebase/upstream-<YYYYMMDD>-<upstream_short_sha>`.
     - When `sync_mode` is `update-base`, saves the previous head to `backup/<base_branch>-<YYYYMMDD>-<old_short_sha>` and updates `<base_branch>` with `--force-with-lease`.
     - Closes any open `upstream-rebase-conflict` issues.
   - **Conflict**:
     - Aborts the rebase.
     - Creates or updates an open Issue labeled `upstream-rebase-conflict` with the conflicting commit, file list, rebase output, and the exact prompt/commands for a local AI agent to resolve and push.

## Branch Rules for Custom Changes

1. **Keep custom commits linear on `main`**:
   - Commit custom changes directly on `main` or rebase feature branches onto `main`.
   - Do not run `git merge upstream/main` into `main`; let the workflow (or `git rebase upstream/main`) keep history linear.
2. **Opening PRs to upstream (`getpaseo/paseo`)**:
   - Branch directly from `upstream/main`:
     ```bash
     git fetch upstream main
     git checkout -b fix/<topic> upstream/main
     git cherry-pick <commit-sha>
     ```
   - Once upstream merges the PR, the next `git rebase --empty=drop` drops the duplicate commit automatically.
3. **Resolving a conflict Issue**:
   - Copy the **Local AI Handoff Prompt / Commands** block from the `upstream-rebase-conflict` GitHub Issue into your local coding agent (Claude Code / Codex / Pi).
   - Once pushed to `main`, the issue closes automatically on the next run (or close it manually).

## Required Fork Settings (`suntianc/paseo`)

Upstream commits frequently touch `.github/workflows/*`. GitHub rejects pushes that modify workflow files when authenticated only with the default `GITHUB_TOKEN`. Configure one of the following in `suntianc/paseo`:

1. **SSH Deploy Key (recommended, repo-scoped, no expiration)**:
   - Generate an SSH key pair: `ssh-keygen -t ed25519 -C "upstream-rebase" -f ./upstream_sync_key -N ""`
   - Add `upstream_sync_key.pub` to `Settings -> Deploy keys` with **Allow write access** enabled.
   - Add the private key `upstream_sync_key` to `Settings -> Secrets and variables -> Actions` as `UPSTREAM_SYNC_SSH_KEY`.
2. **Personal Access Token (alternative)**:
   - Create a PAT with `repo` and `workflow` scopes and save it as the Actions secret `UPSTREAM_SYNC_TOKEN`.
3. **Enable Issues on the fork**:
   - Enable **Issues** under `Settings -> General -> Features` so conflict issues can be created.
