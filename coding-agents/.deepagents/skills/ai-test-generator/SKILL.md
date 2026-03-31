---
name: ai-test-generator
description: >
  AI-driven unit test generation for Java and Kotlin components. Orchestrates the
  test-analyzer and test-refiner subagents in a human-in-the-loop review loop.
  Writes approved tests directly to src/test/java or src/test/kotlin.
  Triggers on: "generate tests for", "create unit tests", "write test cases for",
  "automate QA for this component", or any request for unit test generation or
  QA automation on a specific file or folder.
argument-hint: <target-path> [jira-ticket] [base-branch]
disable-model-invocation: true
metadata:
  allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---
## IMPORTANT: Agent Calling Convention

**This skill should be called using the `task` tool** rather than direct invocation.
When implementing this skill in a system that supports the `task` tool, use:

```python
task(
    description="Generate tests for the specified target path with optional Jira ticket and base branch",
    subagent_type="general-purpose"
)
```

The skill implementation will handle the orchestration of test-analyzer and test-refiner subagents.
# Generate Tests

## ⚙️ Configuration — set before first use

```
CLOUD_ID = bbfdcd85-941a-4950-bf7b-b6136d41bc35
```

Pass this as `cloudId` in every Atlassian MCP tool call.
All downstream agents also receive this value — pass it explicitly when delegating.
Find your cloud ID at: `https://<your-domain>.atlassian.net/_edge/tenant_info`

---

## Available MCP tools — exact names, use verbatim

> Do not invent tool names. Do not make exploratory calls.
> Only call a tool if it is required to complete the task.
> Never hallucinate results. If a call fails, report it — do not invent a substitute.

### Jira

| Tool | Purpose | Key params |
|------|---------|-----------|
| `SearchIssues` | Fetch ticket data via JQL — **only way to get ticket data** | `cloudId`, `jql`, `fields`, `maxResults` |
| `addComment` | Post a comment on a ticket | `cloudId`, `issueIdOrKey`, `body` (ADF) |

> ⚠️ No `getIssue` or `getJiraIssue` tool exists. To fetch a ticket always use:
> `SearchIssues(cloudId="bbfdcd85-...", jql='issue = "MYRIVA-247"', fields=["summary","description","status","assignee","comment","acceptance criteria"])`

### Confluence

| Tool | Purpose | Key params |
|------|---------|-----------|
| `searchConfluenceUsingCql` | Search pages by CQL | `cloudId`, `cql`, `limit` |
| `getConfluencePage` | Fetch a page by ID | `cloudId`, `id` |

### ADF body format (required for `addComment`)

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

Parse from `$ARGUMENTS`: `target-path`, `jira-ticket`, `base-branch`

Example: `/generate-tests src/main/java/com/myriva/booking MYRIVA-247 main`

If `target-path` is missing or does not exist on disk, ask for it before doing anything else.

## Git diff (injected at invocation)

```diff
!`git diff ${ARGUMENTS[2]:-main}..HEAD -- $ARGUMENTS[0] 2>/dev/null || echo "(no diff available — check branch name)"`
```

---

## Phase 0 — Developer context intake

Before using any tools, ask the developer:

> "Before I start analysing — a few things that will help me generate better tests:
>
> 1. What is this component responsible for? (one sentence is enough)
> 2. Are there known edge cases or error scenarios this code should handle?
> 3. Are there any Confluence pages with test standards or coding conventions I should follow?
>    Paste titles or URLs — or I can search automatically using the ticket keywords.
> 4. Any existing test files in this area I should be consistent with?"

Wait for the response before proceeding.

---

## Phase 1 — Context gathering

### 1a. Fetch the Jira ticket (if provided)

Call `SearchIssues` with:
- `cloudId` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`
- `jql` = `issue = "$ARGUMENTS[1]"`
- `fields` = `["summary", "description", "status", "assignee", "comment"]`

Extract: summary, description, status, assignee, acceptance criteria, and any diagnostic
info in comments. If the call returns no results, warn the developer and continue without
Jira context — do not stop.

### 1b. Search Confluence for test standards

If the developer provided Confluence pages in Phase 0:
- Page ID provided → call `getConfluencePage(cloudId=..., id=<id>)`
- Title provided → call `searchConfluenceUsingCql(cloudId=..., cql='title = "<title>"')`

If no pages were provided, run two CQL searches:
1. `searchConfluenceUsingCql(cloudId=..., cql='text ~ "unit test" AND text ~ "java" AND space.type = "global" ORDER BY lastModified DESC', limit=5)`
2. `searchConfluenceUsingCql(cloudId=..., cql='text ~ "testing standards" AND space.type = "global"', limit=3)`

Extract any test conventions, coverage standards, or architectural patterns found.
Summarise in 3–5 bullet points. If nothing relevant is found, note it and continue.

### 1c. Delegate to research-agent (once, upfront)

Delegate to `@research-agent`. Pass:
- `MODE` = `upfront`
- `CALLING_SKILL` = `generate-tests`
- `LANGUAGE` = `java`, `kotlin`, or `mixed` (detect from files in `$ARGUMENTS[0]`)
- `QUESTION` — "What are the most important unit testing patterns for this component type based on its annotations, dependencies, and framework usage?"
- `TICKET_SUMMARY` — from Jira (or empty)
- `CONFLUENCE_EXCERPTS` — summaries from 1b (or empty)
- `CLOUD_ID` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`

Store the returned **research context package** — it is injected into every downstream agent.
`@research-agent` is the only agent that calls external URLs.
Do not run web searches yourself at any point.

> `@research-agent` is a shared agent also used by `bug-fix-suggestion`.
> If a session is already open and a context package was recently produced,
> you may reuse it rather than re-delegating — ask the developer first.

### 1d. Fallback — thin context

If critical context is still missing after 1a–1c, ask the developer for the specific gap.
Do not hallucinate context. Do not proceed with invented assumptions.

---

## Phase 2 — Analysis

Delegate to `@test-analyzer`. Pass:
- `TARGET_PATH` = `$ARGUMENTS[0]`
- `BASE_BRANCH` = `$ARGUMENTS[2]` (default: `main`)
- `JIRA_TICKET` = `$ARGUMENTS[1]` (if provided)
- `GIT_DIFF` = the diff injected above — do not ask the subagent to re-run git diff
- `RESEARCH_CONTEXT` = the context package from Phase 1c
- `CONFLUENCE_STANDARDS` = the Confluence excerpts from Phase 1b
- `CLOUD_ID` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`

The subagent returns a JSON array of prioritised test candidates.

Once it returns, present to the developer:
- How many candidates were found
- A summary table: `id | method | type | priority | rationale`
- Any coverage gaps it explicitly flagged

Ask before proceeding: *"I found N test candidates. Ready to start reviewing them one by one? (yes / no / adjust)"*

If the developer wants to add, remove, or re-prioritise candidates before starting,
incorporate their input and show the updated list before asking again.

---

## Phase 3 — Test generation and review loop

### Attempt tracking

Track denials per candidate. Max 3 revision attempts per test.
After 3 denials, log as requiring manual authoring and move on — never loop indefinitely.

### For each candidate, in order:

**Step A — Generate**

Read `references/test-conventions.md`.
Use `source_language` from the candidate to pick Java or Kotlin conventions.
Inject the research context package from Phase 1c.
Generate a complete `@Test` method with `// Given`, `// When`, `// Then` structure.

If unsure about the correct assertion, mock, or framework pattern for this specific candidate,
delegate to `@research-agent` with:
- `MODE` = `targeted`
- `CALLING_SKILL` = `generate-tests`
- `LANGUAGE` = (from candidate's `source_language`)
- `QUESTION` = the specific question
Do not guess — ask the researcher.

**Step B — Present**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Test TC-001 of N  [Java]
Class:    BookingService
Method:   processBooking(BookingRequest)
Type:     happy_path  |  Priority: high
Rationale: Changed in diff + @Transactional + Jira AC mentions success flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

<generated test method here>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Approve  |  ❌ Deny (tell me why)  |  ⏭ Skip
```

**Step C — Handle the response**

**Approve** → queue for writing (do not write yet — see Phase 4), continue to next candidate

**Skip** → log it with reason, continue to next candidate

**Deny** → ask for feedback, then delegate to `@test-refiner`:
```
ORIGINAL_CANDIDATE:    <candidate JSON>
REJECTED_CODE:         <the test that was rejected>
DEVELOPER_FEEDBACK:    <what the developer said>
ITERATION:             <current attempt count, starting at 1>
RESEARCH_CONTEXT:      <context package from Phase 1c>
CLOUD_ID:              bbfdcd85-941a-4950-bf7b-b6136d41bc35
```
Display the revised test. Repeat Step B. Count attempts.
After 3 denials: log as `manual_required`, move to next candidate.

---

## Phase 4 — Pre-write confirmation and file writing

After all candidates have been reviewed, present a summary before writing anything:

```
## Ready to write

Approved tests: N
  - BookingService.processBooking → happy_path
  - BookingService.processBooking → error_handling
  - PaymentValidator.validate → edge_case
  ...

Files that will be created or modified:
  - src/test/java/com/myriva/booking/BookingServiceTest.java  [CREATE]
  - src/test/java/com/myriva/payment/PaymentValidatorTest.java  [CREATE]

Skipped: N  |  Flagged for manual: N
```

Ask: *"Shall I write these N tests to src/test? (yes / no / review first)"*

Do not write any file until the developer says yes or equivalent.

### Delegate to the shared test writing agent

Once confirmed, delegate to the appropriate shared agent based on language:

| Language | Agent |
|----------|-------|
| Java | `@java-unit-test-agent` |
| Kotlin | `@java-unit-test-agent` (handles Kotlin — see conventions) |

> These agents are shared with `bug-fix-suggestion`. They already know the MyRiva
> test conventions, follow the confirm-before-write pattern, and report back with
> file paths and test counts.

Pass to the agent:
- `APPROVED_CANDIDATES` — the full list of approved candidate JSONs
- `GENERATED_TESTS` — the generated test methods matched to each candidate
- `RESEARCH_CONTEXT` — context package from Phase 1c
- `CLOUD_ID` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`

The agent will:
1. Present the complete test class(es) for final review
2. Wait for explicit confirmation
3. Write files to `src/test/java/...` or `src/test/kotlin/...`
4. Report back: file paths written + test count per file

After writing, confirm what was written to the developer.

---

## Phase 5 — Coverage report

After writing, generate and write to `test-gen-reports/$ARGUMENTS[1]/`
(or `test-gen-reports/no-ticket/` if no ticket):

**coverage-estimate.md** — following `references/coverage-rules.md`:
- Per-class table: Class | Total methods | Tests written | Coverage %
- Overall % and whether the 50% WBS target was met
- List of uncovered methods flagged for awareness

**review-log.md** — session summary:
- Counts: approved / denied-manual / skipped
- Approved: method + test type + iterations needed
- Flagged: method + last feedback received
- Skipped: method + reason

Print the coverage table to the terminal before offering Phase 6.

---

## Phase 6 — Jira comment (opt-in only)

Offer once: *"Want me to post the coverage summary as a comment on $ARGUMENTS[1]?"*

If yes, call `addComment`:
- `cloudId` = `bbfdcd85-941a-4950-bf7b-b6136d41bc35`
- `issueIdOrKey` = ticket key from `$ARGUMENTS[1]`
- `body` = ADF format containing:
  - Coverage table (class | methods | tests | %)
  - List of approved test names
  - List of methods flagged for manual authoring
  - Disclaimer: `Generated by ai-test-generator — tests reviewed and approved by developer`

Do not post to Jira unless explicitly confirmed. Do not call `DoTransition` or any other
write tool unless the developer explicitly asks.

---

## General principles

- Never write to any file without developer confirmation (Phase 4 gate).
- Never call `addComment` unless explicitly asked (Phase 6 opt-in).
- Never hallucinate Jira, Confluence, or code content. Report failures, do not invent data.
- If context is thin at any phase — ask, do not guess.
- The research agent (`@test-researcher`) runs once in Phase 1c. Do not re-invoke it
  unless the developer asks to research a specific new question mid-session.
- All three subagents receive the same `CLOUD_ID` and `RESEARCH_CONTEXT` — pass them explicitly.

---

## Reference files

Load only when needed:
- `references/test-conventions.md` — Java + Kotlin test conventions for MyRiva
- `references/coverage-rules.md` — Coverage calculation rules and 50% target definition