---
name: pr-review
description: >
  AI-powered PR review for MyRiva developers. Fetches Jira context from the ticket linked
  in the branch name, analyses the diff for bugs, security issues, vulnerable dependencies,
  and logic errors, then posts a structured review comment on the Bitbucket PR.
  Use when: "review this PR", "do a code review", "review my pull request", "check my PR".
argument-hint: <pr-id>
disable-model-invocation: true
metadata:
  model: bedrock:global.amazon.nova-2-lite-v1:0
---
## IMPORTANT: Agent Calling Convention

**This skill should be called using the `task` tool** rather than direct invocation.
When implementing this skill in a system that supports the `task` tool, use:

```python
task(
    description="Review pull request #{pr_id} with comprehensive analysis",
    subagent_type="general-purpose"
)
```

The skill implementation will handle fetching Jira context, analyzing diffs for bugs,
security issues, vulnerable dependencies, and logic errors, then posting a structured 
review comment on the Bitbucket PR.
# PR Review Skill

## Description

Performs a thorough, high-signal code review on a Bitbucket pull request. Fetches Jira context
from the ticket linked in the branch name, analyses the diff for bugs, security issues, vulnerable
dependencies, and logic errors, then posts a rich structured review comment on the Bitbucket PR.

Uses `atlassian-jira-bitbucket-mcp` exclusively. No `gh` CLI, `curl`, or any other HTTP tool.

---

## Agent Assumptions (applies to all agents and subagents)

- All MCP tools are functional and will work without error. Do not test tools or make exploratory calls.
- Only call a tool if it is required to complete the task. Every tool call must have a clear purpose.
- Never hallucinate PR data, Jira data, or file contents. If a tool call fails, report the failure — do not invent a substitute.
- Do not re-fetch data that has already been retrieved in a parent agent and passed down.

---

## Trigger

Use this skill when the user says any of the following (or close variants):

- "review this PR"
- "do a code review"
- "review my pull request"
- "check my PR"

---

## Available MCP Tools (exact names — use these verbatim)

### Bitbucket

| Tool | Purpose |
|---|---|
| `bitbucket_list_workspaces` | List available workspaces |
| `bitbucket_list_repositories` | List repos in a workspace |
| `bitbucket_get_repository` | Get repo details |
| `bitbucket_list_branches` | List branches in a repo |
| `bitbucket_get_branch` | Get a single branch's details |
| `bitbucket_list_pull_requests` | List open/merged PRs |
| `bitbucket_get_pull_request` | Get PR details (title, desc, author, status, source/target branch) |
| `bitbucket_get_pr_commits` | Get commits in a PR |
| `bitbucket_get_pr_diff` | Get full diff of a PR |
| `bitbucket_get_pr_diffstat` | Get list of changed files + line counts |
| `bitbucket_get_pr_review_summary` | Get existing review summary on a PR |
| `bitbucket_list_pr_comments` | List all comments on a PR |
| `bitbucket_create_pr_comment` | Post a comment on a PR |
| `bitbucket_get_file_content` | Read a file at a given ref — use sparingly, only when diff context is insufficient |

### Jira

| Tool | Purpose |
|---|---|
| `jira_SearchIssues` | Search issues by JQL — **the only way to fetch ticket data** |

> ⚠️ **Critical — no `jira_GetIssue` tool exists.**
> To fetch a specific ticket (e.g. `PROJ-123`), always use:
> `jira_SearchIssues(jql='issue = "PROJ-123"')`
> Parse `summary`, `description`, `status`, `assignee`, and `comment` from the result.

---

## Configuration (read from CLAUDE.md)

Before constructing any Jira URL, read `CLAUDE.md` in the repo root and extract:

- `ATLASSIAN_DOMAIN` — e.g. `mycompany.atlassian.net`
- `ATLASSIAN_CLOUD_ID` — UUID from `/_edge/tenant_info`

Jira ticket URL format: `https://{ATLASSIAN_DOMAIN}/browse/{TICKET}`

**If either value is missing or still a placeholder:** stop and tell the user:
> "Please set `ATLASSIAN_DOMAIN` and `ATLASSIAN_CLOUD_ID` in your root `CLAUDE.md` before running a review.
> Find your values at: `https://{YOUR_DOMAIN}.atlassian.net/_edge/tenant_info`"

---

## Steps

### Step 0 — Create a todo list

Before doing anything else, create a todo list of all steps below. Mark items complete as you go.

---

### Step 1 — Read config and get current branch

1. Read `CLAUDE.md` and extract `ATLASSIAN_DOMAIN` and `ATLASSIAN_CLOUD_ID`. Validate both (see Configuration above).
2. Run `Bash(git branch --show-current)` to get the current branch. This is the source of truth — do not use any MCP tool for this.

If the command returns empty or fails, stop and tell the user:
> "Could not determine the current branch. Make sure you are inside a git repository."

Store: `current_branch`, `atlassian_domain`.

---

### Step 2 — Find the open PR for the current branch

1. If workspace or repo slug is not yet known, call `bitbucket_list_workspaces` then `bitbucket_list_repositories` to resolve them. Ask the user only if genuinely ambiguous.
2. Call `bitbucket_list_pull_requests` with `state=OPEN`.
3. Find a PR whose source branch exactly matches `current_branch`.

**No matching PR found:** stop immediately and tell the user:
> "No open pull request found for branch `<current_branch>`. Please create a PR first, then re-run the review."

Do not proceed. Do not search for other PRs or guess.

**PR found:** call `bitbucket_get_pull_request` for full details. Store:
`pr_id`, `pr_title`, `pr_description`, `source_branch`, `destination_branch`, `author`

---

### Step 3 — Check if a review has already been posted

Call `bitbucket_list_pr_comments`. Scan all bodies for the marker `<!-- pr-review-skill -->`.

If found, stop:
> "A review has already been posted on PR #`<pr_id>`. Remove it or pass `--force` to post again."

---

### Step 4 — Extract Jira ticket from all available sources

Apply regex `[A-Z][A-Z0-9]+-[0-9]+` to each of the following sources, in order. Stop at the first match.

1. `source_branch` — the branch name
2. `pr_title` — the PR title
3. `pr_description` — the PR description body
4. PR comments — scan all comment bodies from `bitbucket_list_pr_comments` (already fetched in Step 3, no extra call needed)

**Found in any source:** log where it was found (e.g. "Found ticket PROJ-123 in PR description"), then proceed to Step 5.

**Not found in any source:** stop and present the user with exactly this message:

> Hey, I wasn't able to deduce the Jira ticket from the branch name, PR title, PR description, or PR comments.
>
> 1. Provide the Jira ticket URL (e.g. `https://mycompany.atlassian.net/browse/PROJ-123`)
> 2. Skip Jira context and continue with code review only

Wait for the user to reply. Do not assume, auto-skip, or proceed without a response.

- User picks **1** and provides a URL → extract the ticket key from the last path segment of the URL, proceed to Step 5.
- User picks **2** → set `jira_context = null`, proceed to Step 6.

---

### Step 5 — Fetch Jira context

Call `jira_SearchIssues(jql='issue = "<TICKET>"')`.

Build `jira_context`:
- `ticket`, `title` (summary), `description`, `status`, `assignee`
- `comments` — last 10, chronological (body + author)
- `jira_url` — `https://{ATLASSIAN_DOMAIN}/browse/{TICKET}`

If no results: warn the user, set `jira_context = null`, continue.

---

### Step 6 — Fetch PR diff and changed files (parallel)

1. `bitbucket_get_pr_diff` → full unified diff
2. `bitbucket_get_pr_diffstat` → changed files with line counts

---

### Step 7 — Parallel review (4 agents simultaneously)

Pass each agent: `pr_title`, `pr_description`, full diff, file list, `jira_context` (or null).
Instruction to each: *"All tools are functional. Do not make exploratory calls. Only flag issues you are certain about."*

Each agent returns issues with: `category`, `file`, `line`, `description`, `impact`, `confidence` (High/Medium only — discard Low).

---

**Agent 1 — Logic & Bug Review**
- Logic errors producing wrong results regardless of input
- Null/undefined/nil dereferences that will crash at runtime
- Incorrect conditionals: off-by-one, inverted boolean, wrong operator
- Broken control flow: unreachable code, missing return in non-void path, infinite loop

---

**Agent 2 — Security Review**
- Hardcoded secrets, credentials, tokens, API keys (not placeholders)
- Injection: SQL, OS command, path traversal, LDAP
- Insecure direct object references (IDOR)
- Missing auth/authorisation on sensitive operations
- Sensitive data in logs or error responses
- Weak crypto: MD5/SHA1 for passwords, hardcoded IVs/salts, insecure RNG

---

**Agent 3 — Dependency & Vulnerability Review**

Manifests to scan: `package.json`, `package-lock.json`, `yarn.lock`, `requirements.txt`,
`Pipfile`, `Pipfile.lock`, `pom.xml`, `build.gradle`, `go.mod`, `go.sum`, `Gemfile`, `composer.json`

Flag:
- Dependencies with specific known CVEs (do not guess)
- Unmaintained, deprecated, or known-malicious packages
- Major version downgrades on security-sensitive packages
- `latest`, `*`, or unpinned versions in production dependency files

Do not flag dev-only deps unless they run in CI/CD release pipelines.

---

**Agent 4 — Correctness & Regression Risk**
- API contract violations: wrong HTTP method, missing required fields, wrong response shape
- Config errors: wrong env var names, invalid values, missing required keys
- Breaking changes to public interfaces without a version bump
- Incorrect SDK/library API usage visible in the diff

---

### Step 8 — Validate flagged issues (parallel)

For every issue from Step 7, launch a parallel validation subagent with:
- Issue details + relevant diff snippet with surrounding context
- `pr_title`, `pr_description`, `jira_context`

Question: **"Is this definitely a real issue with high confidence?"**

**Auto-discard:**
- Pre-existing issues not in this PR's diff
- Style, formatting, naming, documentation
- Linter-catchable issues
- Nitpicks a senior engineer would skip
- Issues requiring unknown runtime state to manifest

**Confirm only if:**
- Code will definitively crash or produce wrong output from the diff alone
- Secret is unambiguously hardcoded (not a placeholder)
- Specific known-vulnerable dependency version is explicitly introduced
- Clear security vulnerability with no apparent mitigation in the changed lines

Opus agents → bugs, logic, security. Sonnet agents → dependencies, config.

---

### Step 9 — Filter to confirmed issues only

Discard unconfirmed. Empty list = clean review.

---

### Step 10 — Print terminal summary

```
## PR Review Summary — PR #<pr_id>: <pr_title>
Jira: <ticket> — <title>   |   Status: <status>   |   Assignee: <assignee>
(or "No Jira context" if null)

Issues found: <n>
[CATEGORY] <description> — <file>:<line>
...
```

---

### Step 11 — Post review comment

Call `bitbucket_create_pr_comment` with the body below.
`<!-- pr-review-skill -->` **must be the first line** (duplicate detection marker).

---

#### Comment template — issues found

```markdown
<!-- pr-review-skill -->

---
## 🔍 Automated PR Review

| | |
|---|---|
| **PR** | #pr_id — pr_title |
| **Author** | author |
| **Branch** | `source_branch` → `destination_branch` |
| **Jira** | [TICKET](jira_url) — jira_title · `STATUS` |
| **Reviewed at** | YYYY-MM-DD HH:MM UTC |

> Analysed for bugs, security vulnerabilities, dependency risks, and correctness.

---

## 📊 Review Summary

| Category | Count | Severity |
|---|---|---|
| 🐛 Bug | N | 🔴 High / 🟡 Medium |
| 🔒 Security | N | 🔴 High |
| 📦 Dependency | N | 🟡 Medium |
| ⚙️ Config | N | 🟡 Medium |
| ✅ Correctness | N | 🟡 Medium |

**Files reviewed:** N changed files · **Issues found:** N total

---

## 🚨 Issues

---

### 🔴 [BUG] — `path/to/file.ext`

> **Line N** · Confidence: High

**What's wrong:**
Clear, specific description of the bug.

**Impact:**
What breaks, crashes, or produces wrong output — and when.

**Suggested fix:**
```language
// only include this block if the fix is small, self-contained, and complete
// omit entirely for larger or multi-location fixes
```

---

### 🔴 [SECURITY] — `path/to/file.ext`

> **Line N** · Confidence: High

**What's wrong:**
Description of the vulnerability.

**Impact:**
What is exposed or exploitable.

**Suggested fix:**
```language
// corrected code if self-contained
```

---

### 🟡 [DEPENDENCY] — `package.json`

> **Dependency:** `package-name@x.y.z` · Confidence: High

**What's wrong:**
Description of the CVE or risk.

**Impact:**
What this enables or exposes.

**Recommendation:**
Upgrade to `package-name@a.b.c` or later.

---

_(repeat a block per issue, using the appropriate severity emoji and category label)_

---

## 📋 Jira Context Used

| Field | Value |
|---|---|
| **Ticket** | [TICKET](jira_url) |
| **Summary** | jira_title |
| **Status** | status |
| **Assignee** | assignee |

> ℹ️ Jira context was used to understand author intent and distinguish intentional changes from bugs.

---

*Review generated by `pr-review-skill` · Powered by Claude*
```

---

#### Comment template — no issues found

```markdown
<!-- pr-review-skill -->

---
## 🔍 Automated PR Review

| | |
|---|---|
| **PR** | #pr_id — pr_title |
| **Author** | author |
| **Branch** | `source_branch` → `destination_branch` |
| **Jira** | [TICKET](jira_url) — jira_title · `STATUS` |
| **Reviewed at** | YYYY-MM-DD HH:MM UTC |

---

## ✅ No Issues Found

This PR was reviewed across the following categories and no confirmed issues were found:

| Category | Result |
|---|---|
| 🐛 Bugs & Logic | ✅ Clean |
| 🔒 Security | ✅ Clean |
| 📦 Dependencies | ✅ Clean |
| ⚙️ Config | ✅ Clean |
| ✅ Correctness | ✅ Clean |

**Files reviewed:** N changed files

---

## 📋 Jira Context Used

| Field | Value |
|---|---|
| **Ticket** | [TICKET](jira_url) |
| **Summary** | jira_title |
| **Status** | status |
| **Assignee** | assignee |

---

*Review generated by `pr-review-skill` · Powered by Claude*
```

> If `jira_context` is null, replace the entire **Jira Context Used** section with:
> `> ℹ️ No Jira context was available for this review.`

---

## Category Labels & Severity Badges

| Label | Emoji | Severity | Use for |
|---|---|---|---|
| `BUG` | 🐛 | 🔴 High | Logic error, crash, wrong output |
| `SECURITY` | 🔒 | 🔴 High | Vulnerability, hardcoded secret, missing auth |
| `DEPENDENCY` | 📦 | 🟡 Medium | Vulnerable, malicious, or unpinned package |
| `CONFIG` | ⚙️ | 🟡 Medium | Wrong env var, invalid value, missing key |
| `CORRECTNESS` | ✅ | 🟡 Medium | API contract violation, breaking interface change |

---

## Hard Rules — Never Include in the Final Comment

- Code style, formatting, or naming conventions
- Missing comments or documentation
- Test coverage gaps (unless `CLAUDE.md` explicitly requires them)
- Hypothetical bugs requiring specific unknown runtime state
- Pre-existing issues not introduced by this PR
- Any finding with confidence below High
