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

Authentication is handled automatically via OAuth when the plugin is installed.
You interact with Clawvisor entirely through MCP tools — no manual URLs or
tokens required. All traffic is end-to-end encrypted (X25519 ECDH + AES-256-GCM)
— the relay never sees plaintext requests or responses.

The authorization model has two layers — applied in order:
1. **Restrictions** — hard blocks the user sets. If a restriction matches, the action is blocked immediately.
2. **Tasks** — scopes you declare. Every request must be attached to an approved task. If the action is in scope with `auto_execute`, it runs without approval. Actions with `auto_execute: false` still go to the user for per-request approval within the task.

---

## Typical Flow

1. Call `fetch_catalog` — confirm the service is active and the action isn't restricted
2. Call `create_task` declaring your purpose and the actions you need
3. Tell the user to approve it; call `get_task` with `wait: true` until approved
4. Call `gateway_request` under the task — in-scope actions execute automatically
5. Call `complete_task` when done

---

## Getting Your Service Catalog

At the start of each session, call the `fetch_catalog` tool. This returns the
services available to you, their supported actions, which actions are restricted
(blocked), and a list of services you can ask the user to activate. Always fetch
this before making gateway requests so you know what's available and what is
restricted.

---

## Task-Scoped Access

Before making gateway requests, declare a task scope with `create_task`:

```json
{
  "purpose": "Review last 30 iMessage threads and classify reply status",
  "authorized_actions": [
    {"service": "apple.imessage", "action": "list_threads", "auto_execute": true, "expected_use": "List recent iMessage threads to find ones needing replies"},
    {"service": "apple.imessage", "action": "get_thread", "auto_execute": true, "expected_use": "Read individual thread messages to classify reply status"}
  ],
  "expires_in_seconds": 1800
}
```

- **`purpose`** — shown to the user during approval and used by intent verification to ensure requests stay consistent with declared intent. Be specific.
- **`expected_use`** — per-action description of how you'll use it. Shown during approval and checked by intent verification against your actual request params. Be specific: "Fetch today's calendar events" is better than "Use the calendar API."
- **`auto_execute`** — `true` runs in-scope requests immediately; `false` still requires per-request approval (use for destructive actions like `send_message`).
- **`expires_in_seconds`** — task TTL. Omit and set `"lifetime": "standing"` for a task that persists until the user revokes it (see below).

All tasks start as `pending_approval` — the user is notified to approve the
scope before it becomes active. Call `get_task` with `wait: true` to long-poll
until `status` changes to `active` (or `denied`).

### Standing tasks

For recurring workflows, create a **standing task** that does not expire:

```json
{
  "purpose": "Ongoing email triage",
  "lifetime": "standing",
  "authorized_actions": [
    {"service": "google.gmail", "action": "list_messages", "auto_execute": true, "expected_use": "List recent emails to identify ones needing attention"},
    {"service": "google.gmail", "action": "get_message", "auto_execute": true, "expected_use": "Read individual emails to triage and summarize"}
  ]
}
```

Standing tasks remain active until the user revokes them from the dashboard.

### Scope expansion

If you need an action not in the original task scope, call `expand_task`:

```json
{
  "task_id": "<task-id>",
  "service": "apple.imessage",
  "action": "send_message",
  "auto_execute": false,
  "reason": "John Doe asked a question that warrants a reply"
}
```

The user will be notified to approve the expansion. On approval, the action is
added to the task scope and the expiry is reset.

### Completing a task

When you're done, call `complete_task` with the task ID.

---

## Gateway Requests

Every gateway request must include a `task_id` from an approved task. Call
`gateway_request` with:

```json
{
  "service": "<service_id>",
  "action": "<action_name>",
  "params": { ... },
  "reason": "One sentence explaining why",
  "request_id": "<unique ID you generate>",
  "task_id": "<task-uuid>",
  "context": {"source": "Brief description of what you're working on"}
}
```

### Required fields

| Field | Description |
|---|---|
| `service` | Service identifier (from your catalog) |
| `action` | Action to perform on that service |
| `params` | Action-specific parameters (from your catalog) |
| `reason` | One sentence explaining why. Shown in approvals and audit log. Be specific. |
| `request_id` | A unique ID you generate (e.g. UUID). Must be unique across all your requests. Safe to reuse on retry after `INVALID_REQUEST` errors (parse failures don't consume the ID). |
| `task_id` | The approved task ID this request belongs to. |

### Context field

The `context` field is an object (not a string) with the following shape:

```json
{
  "context": {
    "source": "Brief description of what you're working on"
  }
}
```

Include `context.source` with a description of what you're working on and what
external data influenced the request. This is used for detecting prompt
injection attacks and for security forensics.

When processing external content, mention the source:
- `"source": "Acting on Gmail message msg-abc123"`
- `"source": "Processing content from https://example.com/page"`
- `"source": "Responding to GitHub issue org/repo#42"`

The `context` field is optional. If you omit it, the request will still succeed.

---

## Handling Responses

Every response has a `status` field. Handle each case as follows:

| Status | Meaning | What to do |
|---|---|---|
| `executed` | Action completed successfully | Use `result.summary` and `result.data`. Report to the user. |
| `pending` | Awaiting human approval | Tell the user: "I've requested approval for [action]." Re-send the same `gateway_request` with the same `request_id` until resolved. Do **not** send a new request. |
| `blocked` | A restriction blocks this action | Tell the user: "I wasn't allowed to [action] — [reason]." Do **not** retry or attempt a workaround. |
| `restricted` | Intent verification rejected the request | Your params or reason were inconsistent with the task's approved purpose. Adjust and retry with a new `request_id`. |
| `pending_task_approval` | Task not yet approved | Tell the user and call `get_task` with `wait: true` until approved. |
| `pending_scope_expansion` | Request outside task scope | Call `expand_task` with the new action. |
| `task_expired` | Task has passed its expiry | Expand the task to extend, or create a new task. |
| `error` (`SERVICE_NOT_CONFIGURED`) | Service not yet connected | Tell the user: "[Service] isn't activated yet. Connect it in the Clawvisor dashboard." |
| `error` (`EXECUTION_ERROR`) | Adapter failed | Report the error to the user. Do not silently retry. |
| `error` (`INVALID_REQUEST`) | JSON body could not be parsed | Check that your JSON is well-formed and that field types match the schema (e.g. `context` must be an object, not a string). Fix and retry with the same `request_id` — parse errors do not consume the request ID. |
| `error` (other) | Something went wrong | Report the error message to the user. Do not silently retry. |

---

## Authorization Model Summary

| Condition | Gateway `status` |
|---|---|
| Restriction matches | `blocked` |
| Task in scope + `auto_execute` + verification passes | `executed` |
| Task in scope + `auto_execute` + verification fails | `restricted` |
| Task in scope + `auto_execute: false` | `pending` (per-request approval) |
| Action not in task scope | `pending_scope_expansion` |
