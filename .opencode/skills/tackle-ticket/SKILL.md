---
name: tackle-ticket
description: Workflow for tackling a GitHub issue - fetch ticket, branch, implement, commit, PR
compatibility: opencode
---

## What I do

Follow this exact workflow whenever the user asks to tackle, work on, or fix a GitHub issue.

## Workflow

### 1. Fetch the ticket

- If the user provided an issue number, fetch it with `gh issue view <number>`
- If the issue does not exist, create it first using `gh issue create`, add it to the project board (project #1, owner `denisecodes`), set status to **Backlog**, and add appropriate labels before proceeding
- Read the issue body carefully — note any instructions about not committing until local testing is confirmed

### 2. Confirm with the user

- Summarise what the ticket is asking for
- Confirm the plan before making any changes

### 3. Prepare the branch

Always run these three steps in order — never skip any:

```bash
git checkout main
git pull
git checkout -b feature/<issue-number>-<short-description>
```

### 4. Implement the changes

- Make all necessary code, config, and doc changes
- Use the TodoWrite tool to track progress through the tasks

### 5. Commit and push — with one important exception

**Check for a hold before committing:**
- If the ticket says "do not commit until local testing is completed", OR
- If the user explicitly says to wait before committing

→ Stop after making the changes, show a summary of what was changed, and ask the user to confirm local testing is done before proceeding.

Otherwise, commit and push normally:

```bash
git add <files>
git commit -m "<conventional-commit-prefix>: <description>"
git push -u origin feature/<issue-number>-<short-description>
```

Commit prefix guide:
- `feat:` — new feature
- `fix:` — bug fix
- `chore:` — maintenance, config, docs
- `ci:` — CI/CD workflow changes
- `test:` — tests

### 6. Open a PR

```bash
gh pr create \
  --title "<same as commit message>" \
  --body "..."
```

PR body must include:
- `## Summary` with 1–3 bullet points
- `Closes #<issue-number>` at the end

### 7. Labels and project board

- All new issues must be added to GitHub Project #1 (owner: `denisecodes`) with status **Backlog**
- Add relevant labels from the available set: `github-actions`, `ansible`, `argocd`, `k3s`, `linux`, `longhorn`, `traefik`, `cert-manager`, `sealed-secrets`, `dns`, `monitoring`, `security`, `backup`, `dashboard`, `gitea`, `vaultwarden`, `nextcloud`, `jellyfin`, `hardware`, `documentation`, `testing`, `gitops`, `infrastructure`, `hardening`, `app-deployment`

## Project conventions

- Branch naming: `feature/<issue-number>-<short-description>`
- Do NOT edit files inside `k3s-ansible/` (it is a submodule)
- Do NOT `git add`, `git commit`, or `git push` if a hold is in place
- GPG signing is active — always use `git add` then `git commit` directly, never `--no-verify`
- Always include `Closes #<issue-number>` in PR descriptions
- When adding new issues to project board, set status to **Backlog** using the GraphQL mutation:
  - Project ID: `PVT_kwHOBxiYFs4BU2Fn`
  - Status field ID: `PVTSSF_lAHOBxiYFs4BU2FnzhFVVXg`
  - Backlog option ID: `33ae66f6`
  - In Progress option ID: `787c05c0`
  - Done option ID: `aabf23c4`
