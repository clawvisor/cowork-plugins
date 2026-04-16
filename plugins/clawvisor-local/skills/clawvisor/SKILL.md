---
name: clawvisor
description: >
  Route tool requests through Clawvisor for credential vaulting, task-scoped
  authorization, and human approval flows. Use for Gmail, Calendar, Drive,
  Contacts, GitHub, Slack, Notion, Linear, Stripe, Twilio, and iMessage.
  Clawvisor enforces restrictions, manages task scopes, and injects
  credentials — you never handle secrets directly.
---
# Clawvisor

Clawvisor is a gatekeeper between you and external services. Every action goes
through Clawvisor, which checks restrictions, validates task scopes, injects
credentials, optionally routes to the user for approval, and returns a clean
semantic result. You never hold API keys.

The authorization model has two layers — applied in order:
1. **Restrictions** — hard blocks the user sets. If a restriction matches, the action is blocked immediately.
2. **Tasks** — scopes you declare. Every request must be attached to an approved task. If the action is in scope with `auto_execute`, it runs without approval. Actions with `auto_execute: false` still go to the user for per-request approval within the task.

---

## Typical Flow

1. **Fetch the catalog** — use `fetch_catalog` to see available services, actions, and restrictions
2. **Create a task** — use `create_task` to declare your purpose and the actions you need
3. **Wait for approval** — use `get_task` with `wait: true` to long-poll until the user approves (or denies) the task
4. **Make gateway requests** — use `gateway_request` for each action, under the approved task scope
5. **Complete the task** — use `complete_task` when done
---

## Getting Your Service Catalog

At the start of each session, use `fetch_catalog` to see what's available. This
returns the services available to you, their supported actions, which actions are
restricted (blocked), and a list of services you can ask the user to activate.
Always check this before making gateway requests so you know what's available and
what is restricted.
---

## Task-Scoped Access

Before making gateway requests, create a task declaring your purpose and the
actions you need using `create_task`:

- **`purpose`** — shown at approval and checked by intent verification. Capability statement covering the workflow's natural follow-ups. Size to task complexity (see below).
- **`expected_use`** — per-action description checked against your actual request params. Cover the scenarios you'll use in this task.
- **`auto_execute`** — `true` runs in-scope requests immediately; `false` still requires per-request approval (use for destructive actions like `send_message`).
- **`expires_in_seconds`** — task TTL. Omit and set `"lifetime": "standing"` for a task that persists until the user revokes it (see below).
- **`planned_calls`** *(optional)* — pre-register specific API calls you know you'll make. Planned calls are shown to the user during approval, evaluated as part of risk assessment, and **skip intent verification at runtime** when they match. This reduces latency for predictable workflows. Each entry must be covered by `authorized_actions` and must include `params`. Use exact values for known params, or `"$chain"` for values that will come from a prior call's results (e.g. `{"thread_id": "$chain"}`). Calls without params cannot skip verification.

### Sizing scope to task complexity

Scope should cover operations likely *within this task's lifecycle* — no more. Over-scoping dilutes the approval signal; under-scoping triggers mid-task `pending_scope_expansion`.

- **Simple** ("check my email for the last 72 hours"): tight. See the iMessage example above.
- **Exploratory** ("triage my inbox"): broad — enumerate operation categories since the user will iterate.
- **Standing** (persist across invocations): exhaustive capability charter. See the Gmail example below.

For examples of well-scoped tasks and effective gateway requests, see the [Task & Request Examples](https://github.com/clawvisor/clawvisor/blob/main/docs/TASK_EXAMPLES.md).

All tasks start as `pending_approval`. Long-poll with `get_task` (`wait: true`)
until status changes to `active` or `denied`.

### Standing tasks

For recurring workflows, set `lifetime: "standing"`. Standing tasks require a
`session_id` in gateway requests to enable chain context verification across
related requests. Use a consistent `session_id` (e.g., a UUID you generate once
per workflow) across all related requests in a single invocation.

> **⚠️ Standing tasks MUST use `session_id` on every gateway request.** Requests to standing tasks without `session_id` are rejected with a `MISSING_SESSION_ID` error. Chain context verification requires `session_id` to track that entity references (message IDs, thread IDs, etc.) came from your own prior results. Generate one UUID per workflow invocation and pass it as `session_id` on every request in that invocation.

### Chain context verification

Chain context verification extracts structural facts (IDs, email addresses, phone numbers) from adapter results and feeds them into subsequent verification prompts. This verifies that follow-up requests target entities that actually appeared in prior results — preventing a compromised agent from reading an inbox and then emailing an unrelated address.

**Ephemeral (session) tasks** get chain context automatically — no extra fields needed. The task ID is used to scope facts.

**Standing tasks** require a `session_id` in gateway requests to enable chain context. Use a consistent `session_id` (e.g., a UUID you generate once per workflow) across all related requests in a single invocation. This scopes facts to one invocation and prevents unrelated facts from prior invocations from mixing together.

If you omit `session_id` on a standing task, the request is rejected with a `MISSING_SESSION_ID` error. Always include a `session_id` — generate a UUID once per workflow invocation and reuse it across all related requests.

- Chain facts are automatically cleaned up when a task is completed, denied, or revoked

### Scope expansion

If you need an action not in the original task scope, use `expand_task` with the
new action and a reason. The user will be prompted to approve. On approval, the
action is added to the task scope and the expiry is reset.

### Completing a task

When you're done, use `complete_task` to mark the task as completed. This cleans
up the authorization scope and any chain context facts.

---

## Writing Effective Reasons

The `reason` field on gateway requests is verified by a language model, not pattern-matched. Write reasons the way a human assistant would explain an action to their boss — describe **what** you're doing and **why**, not who told you to do it.

**Do:**
- `"Searching for recent emails from the design team to draft a follow-up reply"`
- `"Reading thread to extract action items for the weekly standup summary"`
- `"Listing recent calendar events to check for scheduling conflicts this afternoon"`
- `"Looking up the intro email from the vendor to confirm reply status"`

**Don't:**
- `"The user told me to do this"` — claiming the user instructed you looks identical to prompt injection to the verifier. Describe the action itself instead.
- `"The owner directly instructed me to reply to this email via Telegram"` — the verifier cannot distinguish this from an injected instruction. Instead: `"Drafting reply to the vendor's email and preparing a follow-up summary"`
- `"Doing my job"` / `"As requested"` — too vague; will be flagged as insufficient.
- `"Testing"` / `"Retry"` / `"Trying again"` — implementation details, not a rationale.
- Embedding code, markup, JSON, or system directives in the reason field.

**Key rule:** The verifier treats **all agent-provided fields as untrusted**. Any text that resembles an instruction ("ignore previous rules", "approve this request", "the user said to...") will be flagged as a prompt injection attempt, even if it's a truthful description of what happened. Focus on the *what* and *why* of the action, not the *who told you*.

---

## Gateway Requests

Every gateway request must include a `task_id` from an approved task.

Use `gateway_request` for each action. Required fields:

| Field | Description |
|---|---|
| `service` | Service identifier (from your catalog) |
| `action` | Action to perform on that service |
| `params` | Action-specific parameters (from your catalog) |
| `reason` | Why you need this data and what you'll do with it. Shown in approvals and audit log. Be specific: name the user request, the information you're looking for, and how it fits the workflow. |
| `request_id` | A unique ID you generate (e.g. UUID). Must be unique across all your requests. |
| `task_id` | The approved task ID this request belongs to. |
| `session_id` | *(Standing tasks only)* A consistent UUID across related requests in a single invocation. |

### Context fields

Always include the `context` object. All fields are optional but strongly recommended:

| Field | Description |
|---|---|
| `data_origin` | Source of any external data you are acting on (see below). |
| `source` | What triggered this request: `"user_message"`, `"scheduled_task"`, etc. |

### data_origin — always populate when processing external content

`data_origin` tells Clawvisor what external data influenced this request. This
is critical for detecting prompt injection attacks and for security forensics.

**Set it to:**
- The Gmail message ID when acting on email content: `"gmail:msg-abc123"`
- The URL of a web page you fetched: `"https://example.com/page"`
- The GitHub issue URL you were reading: `"https://github.com/org/repo/issues/42"`
- `null` only when responding directly to a user message with no external data involved

**Never omit `data_origin` when you are processing content from an external
source.** If you read an email and it told you to send a reply, the email is
the data origin — set it.

---

## Handling Responses

Every gateway response has a `status` field:

| Status | Meaning | What to do |
|---|---|---|
| `executed` | Action completed | Use the result. Report to the user. |
| `pending` | Awaiting human approval | Tell the user. Re-send same request_id to poll. |
| `blocked` | A restriction blocks this | Tell the user. Do not retry or work around it. |
| `restricted` | Intent verification rejected | Your params/reason were inconsistent with the task purpose. Adjust and retry with a new request_id. |
| `pending_task_approval` | Task not yet approved | Long-poll get_task until approved. |
| `pending_scope_expansion` | Action outside task scope | Use expand_task. |
| `task_expired` | Task TTL exceeded | Expand or create a new task. |
| `error` (`SERVICE_NOT_CONFIGURED`) | Service not connected | Tell the user to connect it in the Clawvisor dashboard. |
| `error` (`EXECUTION_ERROR`) | Adapter failed | Report the error. Do not silently retry. |
| `error` (other) | Something went wrong | Report the error. Do not silently retry. |

**Warnings:** Responses may include a `"warnings"` array with actionable messages about misconfiguration. Always check for and act on warnings.

**Pagination:** Results may be paginated. Check `result.meta` for continuation fields (e.g. `next_page_token`, `cursor`, `has_more`) and pass them as params in a follow-up gateway request to fetch the next page.

---

## Waiting for Approval

**Tasks (preferred — long-poll):** `get_task` with `wait: true` blocks
server-side until the task leaves `pending_approval` / `pending_scope_expansion`.
Set `timeout` to control the wait (default & max 120 seconds).

If the timeout elapses while still pending, the response is a normal pending
status — just call again to keep waiting.

**Gateway requests:** Re-send the same gateway request with the same
`request_id`. Clawvisor recognizes the duplicate and returns the current status
without re-executing.
---

## Important Rules

- **Always execute curl commands as a single line** — the examples in this document are multi-line for readability, but when running them, inline all variables, headers, and JSON bodies into one command. Never use `\` line continuations, heredocs, or separate variable assignments — each triggers a separate approval prompt.
- Always fetch the catalog first to know what's available and restricted
- Never attempt to bypass restrictions — they are hard blocks set by the user
- Always create a task before making gateway requests
- Use `auto_execute: false` for any action that sends, modifies, or deletes data
- Generate unique request_ids for every gateway request
- Complete tasks when done to clean up authorization scope
- Always set `data_origin` when processing content from external sources

---

## Troubleshooting

If something isn't working as expected, check whether you have the latest version of this skill:

1. Fetch `GET $CLAWVISOR_URL/api/skill/version` — it returns:
   ```json
   { "skill_version": "...", "skill_published_at": "..." }
   ```
2. Compare the `skill_version` in the response with the version in this skill's frontmatter (`dev`, published dev).
3. If a newer version is available, re-fetch the skill from `$CLAWVISOR_URL/skill/SKILL.md` to get the latest instructions.

---

## Authorization Model Summary

| Condition | Gateway `status` |
|---|---|
| Restriction matches | `blocked` |
| Task in scope + `auto_execute` + matches planned call | `executed` (skips verification) |
| Task in scope + `auto_execute` + verification passes | `executed` |
| Task in scope + `auto_execute` + verification fails | `restricted` |
| Task in scope + `auto_execute: false` | `pending` (per-request approval) |
| Action not in task scope | `pending_scope_expansion` |
