---
name: compile-verifier
description: Push local repo changes to the ISAWarden/llama-cpp-isa GitHub repository using an explicit SSH deploy key, trigger the Release GitHub Actions workflow with gh, wait for failures, inspect failed job logs, fix compile or workflow errors, commit and push fixes, and retry until the release build passes or a real blocker remains. Use when the user asks Codex to verify changes through GitHub Actions release builds, diagnose failed release jobs, or run a push-trigger-fix retry loop.
---

# Compile Verifier

## Core Rules

- Treat invocation of this skill as permission to commit and push changes needed for the requested compile verification loop.
- Use only an explicit SSH private key supplied by the user for Git write access. Never print the key, commit it, or store it in the repo.
- Preserve unrelated local changes. Before committing, inspect `git status --short` and stage only files that belong to the current fix.
- Prefer fixing compiler errors and deterministic workflow bugs. Report infrastructure outages, quota limits, runner cancellations, or missing secrets as blockers unless they are clearly repo-fixable.
- Keep retry loops bounded: after three failed release attempts with the same unresolved class of failure, stop and report the blocker.

## Setup

1. Confirm the repo and branch:

```bash
git remote -v
git branch --show-current
git status --short
gh repo view --json nameWithOwner,url
```

2. Install the provided SSH private key outside the repo:

```bash
mkdir -p ~/.ssh
umask 077
cat > ~/.ssh/llama-cpp-isa-release-key
# paste key, then end input
chmod 600 ~/.ssh/llama-cpp-isa-release-key
ssh-keyscan github.com >> ~/.ssh/known_hosts
export GIT_SSH_COMMAND='ssh -i ~/.ssh/llama-cpp-isa-release-key -o IdentitiesOnly=yes'
```

3. Verify GitHub access without exposing secrets:

```bash
ssh -T git@github.com || true
gh auth status
git ls-remote git@github.com:ISAWarden/llama-cpp-isa.git HEAD
```

If `gh` is not authenticated for Actions, ask the user for a token or login method before triggering workflows.

## Push And Trigger

1. Run the strongest practical local check before pushing. For this repo, prefer:

```bash
cmake -B build -DGGML_VULKAN=1
cmake --build build --config Release -j 16
```

Use additional focused builds when the failing area requires it.

2. Stage only intended files and commit:

```bash
git diff --check
git status --short
git add <intended files>
git commit -m "<concise fix message>"
git push origin HEAD
```

If the branch has no upstream, use:

```bash
git push -u origin HEAD
```

3. Trigger the release workflow manually:

```bash
gh workflow run release.yml --repo ISAWarden/llama-cpp-isa --ref "$(git branch --show-current)" -f create_release=true
```

4. Find the new run for the pushed SHA:

```bash
SHA=$(git rev-parse HEAD)
gh run list --repo ISAWarden/llama-cpp-isa --workflow release.yml --commit "$SHA" --limit 5
```

Use the newest run matching the SHA.

## Watch And Diagnose

Watch the run until completion or until failed jobs appear:

```bash
gh run watch <run-id> --repo ISAWarden/llama-cpp-isa --exit-status
```

If `gh run watch` exits on failure, list failed jobs:

```bash
gh run view <run-id> --repo ISAWarden/llama-cpp-isa --json status,conclusion,jobs \
  --jq '.status + " " + (.conclusion // "") + "\n" + (.jobs[] | select(.conclusion=="failure" or .conclusion=="cancelled") | [.databaseId,.conclusion,.name] | @tsv)'
```

Download failed job logs when `gh run view --log-failed` is too noisy or incomplete:

```bash
mkdir -p /tmp/llama-ci-<run-id>
gh api /repos/ISAWarden/llama-cpp-isa/actions/jobs/<job-id>/logs > /tmp/llama-ci-<run-id>/<job-id>.log
rg -n "error:|fatal error:|FAILED:|undefined reference|undeclared|missing field|Process completed with exit code|BUILD FAILED|xcodebuild|CompileC|SwiftCompile" /tmp/llama-ci-<run-id>/<job-id>.log
```

Classify each failure:

- Source compile error: inspect the exact file and surrounding code; fix locally.
- Warning-as-error: fix the warning rather than suppressing it, unless suppression is already the project pattern.
- Workflow script failure: patch the workflow or composite action only when the failure is deterministic and repo-controlled.
- Cache deletion `403`: make cleanup non-fatal.
- Cancelled job: ignore unless logs show a real failure before cancellation.
- Runner, network, GitHub outage, missing permission, or missing secret: report as blocker.

## Fix And Retry Loop

For each actionable failure:

1. Patch the smallest affected surface.
2. Run `git diff --check`.
3. Run the closest local build available. Keep the requested Vulkan build as the baseline:

```bash
cmake -B build -DGGML_VULKAN=1
cmake --build build --config Release -j 16
```

4. Commit only the fix files.
5. Push.
6. Trigger `release.yml` again with `create_release=true`.
7. Repeat diagnosis on the new run.

Stop when:

- The release workflow passes.
- Only non-repo infrastructure failures remain.
- The same failure class persists through three fix attempts.

## Reporting

Report:

- Branch, pushed commit SHA, and release run URL.
- Failed jobs inspected and the root cause of each.
- Files changed and why.
- Local verification commands run.
- Whether another release attempt was triggered and its result.
- Any remaining blockers or unverified platforms.
