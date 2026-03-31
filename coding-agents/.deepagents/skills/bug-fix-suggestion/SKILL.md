---
name: bug-fix-suggestion
description: >
  AI-powered bug fix assistant for MyRiva developers. Use when a developer invokes
  /bug-fix-suggestion with a Jira ticket key, file path, and language (java or javascript).
  Also triggers on: "help me fix this ticket", "suggest a fix for [JIRA-key]",
  "analyse this bug", or when the user pastes a stack trace and asks what to do.
  Orchestrates research, fix suggestion, code implementation, and optional test generation.
argument-hint: <ticket> <file> <language>
disable-model-invocation: true
metadata:
  allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---
## IMPORTANT: Agent Calling Convention

**This skill should be called using the `task` tool** rather than direct invocation.
When implementing this skill in a system that supports the `task` tool, use:

```python
task(
    description="Suggest bug fix for ticket {ticket} in file {file} with language {language}",
    subagent_type="general-purpose"
)
```

The skill implementation will handle the orchestration of research, fix suggestion, 
code implementation, and optional test generation subagents.
# Bug fix suggestion

## ⚙️ Configuration — set before first use

Every Atlassian MCP tool call requires a `cloudId`. Set this once here:

```
ATLASSIAN_CLOUD_ID = bbfdcd85-941a-4950-bf7b-b6136d41bc35
```

**Pass this as the `cloudId` parameter in every Atlassian MCP tool call.**
All downstream agents also receive this value — pass it explicitly when delegating.
If your workspace uses a different cloud ID, update this value before running.
Find your cloud ID at: `https://<your-domain>.atlassian.net/_edge/tenant_info`

---

## Available MCP tools — exact names

> Use these verbatim. Do not invent tool names. Do not make exploratory calls.
> Only call a tool if it is required to complete the task.
> Never hallucinate tool results. If a call fails, report it — do not invent a substitute.

### Jira tools

| Tool | Purpose | Key params |
|---|---|---|
| `SearchIssues` | Search by JQL — **only way to fetch ticket data** | `jql`, `fields`, `maxResults` |
| `addComment` | Post a comment to a ticket | `issueIdOrKey`, `body` (ADF) |
| `createIssue` | Create a new issue | `fields.project.key`, `fields.issuetype.name`, `fields.summary` |
| `DoTransition` | Change ticket status | `issueIdOrKey`, `transition.id` |
| `GetAllBoards` | List all boards | `maxResults` |

> ⚠️ There is no `getIssue` or `getJiraIssue` tool. To fetch a specific ticket use:
> `SearchIssues(jql='issue = "MYRIVA-412"', fields=["summary","description","status","priority","assignee","comment","labels"])`

### Confluence tools

| Tool | Purpose | Key params |
|---|---|---|
| `searchConfluenceUsingCql` | Search pages by CQL | `cql`, `limit` |
| `getConfluencePage` | Fetch a page by ID | `id` (integer), `body-format` |
| `getPages` | List pages with filters | `space-id`, `title`, `limit` |
| `getSpaces` | List available spaces | `type`, `status`, `limit` |

### ADF body format (required for `addComment` and `createIssue` descriptions)

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    { "type": "paragraph", "content": [{ "type": "text", "text": "Your text here" }] }
  ]
}
```

---

## Inputs

Parse from `$ARGUMENTS` in this order: `ticket`, `file`, `language`.

Example: `/bug-fix-suggestion MYRIVA-412 src/main/java/BookingService.java java`

If any of the three inputs are missing, ask for them before doing anything else.

---

## Phase 0 — Developer context intake

Before using any tools, ask the developer:

> "Before I dig in — a few things that will help me give you a better fix:
>
> 1. Can you briefly describe what the bug is doing? (one sentence is enough)
> 2. Do you have any logs, stack traces, or error messages to paste?
> 3. Are there any Confluence pages or coding standards I should consider?
>    Paste titles or URLs — or I can search using the ticket keywords."

Wait for the response before proceeding.

---

## Phase 1 — Jira and Confluence context

### 1a. Fetch the Jira ticket

Call `SearchIssues` with:
- `jql` = `issue = "MYRIVA-412"` (replace with actual ticket from $ARGUMENTS)
- `fields` = `["summary", "description", "status", "priority", "assignee", "comment", "labels"]`

Extract: summary, description, status, priority, labels, and any diagnostic info in comments.

### 1b. Search Confluence

If the developer provided pages in Phase 0:
- Page ID provided → call `getConfluencePage(id=<id>)`
- Title provided → call `searchConfluenceUsingCql(cql='title = "<title>"')`

If no pages were provided, run two CQL searches:
1. `searchConfluenceUsingCql(cql='text ~ "<ticket keywords>" AND space.type = "global" ORDER BY lastModified DESC', limit=5)`
2. `searchConfluenceUsingCql(cql='text ~ "<filename>" AND space.type = "global"', limit=5)`

### 1c. Call the research agent

Delegate to `research-agent`. Pass:
- `ticket_summary`, `file`, `language`, `error_text`, `confluence_excerpts`
- `atlassian_cloud_id` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`

Store the returned **context package** — injected into every downstream agent.
The research agent is the only agent that calls web search. Do not search yourself.

### 1d. Fallback — thin context

If context is still insufficient, stop and ask the developer for the specific gap.
Do not hallucinate context.

---

## Phase 2 — Fix agent delegation

Based on `language`, delegate using the Agent tool:
- `java` → `java-fix-agent`
- `javascript` → `javascript-fix-agent`
- `python` → `python-fix-agent` ⚠️ *(temporary — provisional support)*

Pass:
- Full context package (header: `CONTEXT PACKAGE — use this, do not search the web:`)
- `atlassian_cloud_id` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`
- `ticket`, `file`, `language`

Returns: `ROOT CAUSE` · `FIX` · `CODE PATH` · `IMPLEMENTATION INSTRUCTIONS` · `RISKS`

---

## Phase 3 — Explain the fix and wait for confirmation

Present:

```
## Fix for [TICKET-KEY] — [ticket summary]

### What is going wrong
[ROOT CAUSE in plain language]

### Affected code path
[CODE PATH]

### What will change
[IMPLEMENTATION INSTRUCTIONS — numbered steps]

### The fix
[FIX diff or code block]

### Things to verify before merging
[RISKS — if empty, write "No significant risks identified."]
```

Ask: *"Does this look right? Say yes to apply it, or tell me what to change."*

Do not write to any file until confirmed.

### Feedback loop (pre-implementation)

If not confirmed: acknowledge → re-run fix agent with developer feedback + previous attempt
appended. Repeat up to **3 times**. Do not re-run the research agent. Reuse context package.

---

## Phase 4 — Code implementation

Once confirmed:
1. Read the file — store as `original_file_contents`
2. Apply the fix exactly as specified
3. Confirm what changed

Ask: *"Does this look good, or redo with a different approach? Also — want me to generate tests? (unit / integration / both / skip)"*

### Feedback loop (post-implementation)

If redo requested: restore `original_file_contents` → acknowledge → re-run fix agent with
feedback appended → return to Phase 3. Counts toward the 3-attempt limit.

---

## Phase 5 — Test agent delegation (opt-in)

| Developer says | language=java | language=javascript | language=python ⚠️ |
|---|---|---|---|
| "unit" | `java-unit-test-agent` | `javascript-unit-test-agent` | `python-unit-test-agent` |
| "integration" | `java-integration-test-agent` | `javascript-integration-test-agent` | `python-integration-test-agent` |
| "both" / "all" | Both sequentially | Both sequentially | Both sequentially |
| "no" / "skip" | Skip → Phase 6 | Skip → Phase 6 | Skip → Phase 6 |

> ⚠️ Python support is temporary and provisional. Agents will be removed in a future version.

Pass to each: `file`, `ROOT CAUSE`, `FIX`, `CODE PATH`, context package,
`atlassian_cloud_id` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`

Each agent presents tests → gets confirmation → writes file → reports path back.

---

## Phase 6 — Jira comment (opt-in only)

Offer once at the end: *"Want me to post this fix as a comment on [TICKET-KEY]?"*

If yes, call `addComment`:
- `issueIdOrKey` = ticket key
- `body` = ADF with root cause summary + fix code block + AI-suggested disclaimer
- Omit test code and full code path from the comment

---

## General principles

- Never write to a file without developer confirmation.
- Never call `DoTransition`, `createIssue`, or `addComment` unless explicitly asked.
- Never hallucinate Jira or Confluence data. Report failures, do not substitute invented data.
- If fix confidence is low, say so explicitly.
- Fallback at any phase — thin context means ask, not guess.
