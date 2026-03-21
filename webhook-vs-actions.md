# Webhook + External Service vs GitHub Actions

## Overview

Two main approaches to automate CI/CD and governance on GitHub repositories:
1. **GitHub Actions** — workflow scripts stored in the repo (`.github/workflows/`)
2. **Webhook + External Service** — an external server receives GitHub events and processes them

---

## Comparison

| Aspect | Webhook + External Service | GitHub Actions |
|--------|---------------------------|----------------|
| **Code Visibility** | Hidden from developers | Visible in repo (`.github/workflows/`) |
| **Tampering** | Cannot be modified by developers | Can be modified by anyone with write access |
| **Ecosystem Integration** | Direct (Slack, Jira, Jenkins, etc.) | Needs extra steps/marketplace actions |
| **Status Reporting** | GitHub Status API / Check Runs API | Built-in `$GITHUB_STEP_SUMMARY` |
| **Real-time Updates** | Yes — update check run anytime | Only after step/job completes |
| **Cost** | You host & maintain the service | Free (public repos) / included minutes |
| **Maintenance** | You maintain infra + code | GitHub maintains runner infra |
| **Secrets** | Fully controlled on your server | Stored in GitHub (encrypted) |
| **Scaling** | You manage | GitHub manages |
| **Debugging** | Your own logging system | Built-in logs in Actions UI |
| **Multi-platform** | Same service works for GitHub, GitLab, Bitbucket | GitHub only |

---

## When Webhook + External Service is Better

- **Mandatory enforcement** — developers can't see or alter the logic
- **Multi-system orchestration** — trigger Jenkins, Jira, Slack, deploy pipelines from one place
- **Real-time progress** — update check runs live, not after step finishes
- **Sensitive logic** — security scans, compliance checks you don't want exposed
- **Cross-platform reuse** — same service works for GitHub, GitLab, Bitbucket
- **Full API control** — anything Actions can display, you can do via the GitHub API

## When GitHub Actions is Better

- **Quick setup** — no infrastructure to maintain
- **Simple CI/CD** — build, test, deploy within GitHub
- **Open source projects** — community can see and contribute to workflows
- **Free compute** — no server costs for public repos
- **Built-in marketplace** — thousands of pre-built actions available

---

## Webhook Approach: Key GitHub APIs

### Receiving Events

Configure a webhook in **Settings → Webhooks** to receive events at your endpoint:
```
POST https://your-server.com/github-webhook
```

GitHub sends a JSON payload with full event details (same as `github.event` in Actions).

### Reporting Status Back

#### Check Runs API (Rich display with Markdown)
```
POST /repos/{owner}/{repo}/check-runs
```
```json
{
  "name": "Security Scan",
  "head_sha": "abc123",
  "status": "in_progress",
  "output": {
    "title": "Scanning...",
    "summary": "## Results\n| Check | Status |\n|-------|--------|\n| XSS | ✅ Pass |"
  }
}
```

Update in real-time:
```
PATCH /repos/{owner}/{repo}/check-runs/{check_run_id}
```
```json
{
  "status": "completed",
  "conclusion": "success",
  "output": {
    "title": "All checks passed",
    "summary": "## ✅ Security Scan Complete\n\n| Check | Status |\n|-------|--------|\n| XSS | ✅ Pass |\n| SQLi | ✅ Pass |"
  }
}
```

#### Commit Status API (Simple pass/fail)
```
POST /repos/{owner}/{repo}/statuses/{sha}
```
```json
{
  "state": "success",
  "target_url": "https://your-server.com/results/123",
  "description": "All checks passed",
  "context": "security/scan"
}
```

#### PR Comments API (Post detailed results)
```
POST /repos/{owner}/{repo}/issues/{pr_number}/comments
```
```json
{
  "body": "## 📋 PR Review Summary\n\n| Check | Result |\n|-------|--------|\n| Build | ✅ |\n| Tests | ✅ |\n| Security | ✅ |"
}
```

---

## GitHub Actions Approach: Key Features

### Display via `$GITHUB_STEP_SUMMARY`
```yaml
- name: Show results
  run: |
    echo "## Results" >> $GITHUB_STEP_SUMMARY
    echo "| Check | Status |" >> $GITHUB_STEP_SUMMARY
    echo "|-------|--------|" >> $GITHUB_STEP_SUMMARY
    echo "| Build | ✅ Pass |" >> $GITHUB_STEP_SUMMARY
```

### Pass Variables Between Steps
```yaml
# Via step outputs (GITHUB_OUTPUT)
echo "result=pass" >> $GITHUB_OUTPUT

# Via environment (GITHUB_ENV)
echo "MY_VAR=value" >> $GITHUB_ENV
```

### Live Log Annotations
```yaml
- run: |
    echo "::notice::Phase 1 complete"
    echo "::warning::Slow build detected"
    echo "::error::Test failed"
```

---

## Recommended Architecture

### For Enterprise / Organization Governance

Use **both** approaches together:

```
┌─────────────────────────────────────────────┐
│              GitHub Repository               │
│                                              │
│  .github/workflows/       (Developer CI/CD)  │
│    - build.yml             Build & test      │
│    - deploy.yml            Deploy to staging │
│                                              │
│  Webhook ──► External Svc  (Governance)      │
│    - Security scan         Hidden logic      │
│    - Compliance check      Can't be bypassed │
│    - Jira integration      Multi-system      │
│    - Slack notification    Real-time updates │
└─────────────────────────────────────────────┘
```

- **GitHub Actions** → developer-facing CI/CD (build, test, deploy)
- **Webhook service** → platform-level governance (mandatory checks, integrations)
- **Branch protection / Rulesets** → require both to pass before merge

### Enforcement Layers

| Layer | Purpose |
|-------|---------|
| **Org-level Rulesets** | Repo admins cannot override |
| **Required Status Checks** | Must pass before merge |
| **CODEOWNERS on `.github/workflows/`** | Workflow changes need security team approval |
| **Webhook-based checks** | Logic hidden from developers |
| **Audit Logging** | Track who changed what |

---

## Summary

| Use Case | Recommended Approach |
|----------|---------------------|
| Build & test code | GitHub Actions |
| Deploy to environments | GitHub Actions |
| Mandatory security scans | Webhook + External Service |
| Compliance enforcement | Webhook + External Service |
| Multi-tool integration (Jira, Slack, etc.) | Webhook + External Service |
| Real-time progress reporting | Webhook + External Service |
| Open source community workflows | GitHub Actions |
| Cross-platform (GitHub + GitLab + Bitbucket) | Webhook + External Service |
