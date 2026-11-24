# Telegram Webhook Secret: Test Plan (traffic lights)

Companion to `plan.md`. Update the **Status** column as each task is executed.

## How the lights work

| Light | Meaning |
|---|---|
| ⚪ | Not started: the test doesn't exist yet. |
| 🔴 | Failing, or a blocker. Expected right after writing a test (TDD red); not allowed at a task's gate. |
| 🟡 | Passing, but the evidence is limited (fake bot only, or a manual check not yet done). Allowed at a task's gate only for cases that name the manual check backing them. |
| 🟢 | Passing with the strongest evidence available for that case. |

**Task gate:** a task is ready for review when none of its cases are 🔴 and every 🟡 names the case that will turn it 🟢.
**Release gate (before merging the PR):** all automated cases 🟢, `uv run pytest` fully green.
**Deploy gate:** M-1 to M-4 🟢. M-5 🟢 or 🟡 with a written reason.

## Automated cases (remy_api, `uv run pytest`)

| ID | Task | Case | Expected | Evidence ceiling | Status |
|---|---|---|---|---|---|
| T-CFG-1 | 1 | Valid secret `abc_DEF-123` | Accepted unchanged | 🟢 | ⚪ |
| T-CFG-2 | 1 | Invalid secret: empty, with space, non-ASCII, 257 chars (parametrized) | `ValidationError` | 🟢 | ⚪ |
| T-REG-1 | 2 | Registered URL differs | `set_webhook` awaited with `url`, `secret_token`, `allowed_updates=ALL_TYPES`, `drop_pending_updates=True`; returns `True` | 🟡 until M-2 | ⚪ |
| T-REG-2 | 2 | Registered URL unchanged | `set_webhook` not called; returns `False` | 🟢 | ⚪ |
| T-REG-3 | 2 | `RetryAfter` from Telegram | Swallowed; returns `False` | 🟢 | ⚪ |
| T-WH-1 | 3 | No secret header | 403; agent and bot not called | 🟢 | ⚪ |
| T-WH-2 | 3 | Wrong secret header | 403 | 🟢 | ⚪ |
| T-WH-3 | 3 | Empty secret header | 403 | 🟢 | ⚪ |
| T-WH-4 | 3 | Non-ASCII header bytes | 403, not 500 | 🟢 | ⚪ |
| T-WH-5 | 3 | Rejected request | Neither received nor configured secret in logs | 🟢 | ⚪ |
| T-WH-6 | 3 | Correct header, text update | 200; agent awaited with `chat_id="42"`, `content="hola"`; reply sent | 🟡 until M-5 | ⚪ |
| T-WH-7 | 3 | Correct header, agent raises | Still 200 (existing behavior) | 🟢 | ⚪ |
| T-WH-8 | 3 | Correct header, non-text update | 200; agent not called (existing behavior) | 🟢 | ⚪ |
| T-ALL | each | Full suite incl. the 6 existing CRUD tests | All pass | 🟡 (SQLite only; Postgres untested) | ⚪ |

## Manual cases (human-run, real credentials)

| ID | Task | Check | Pass criteria | Status |
|---|---|---|---|---|
| M-1 | 2, 4 | App starts with the real `.env` | No `ValidationError`; log shows `Webhook registered` or `Webhook already registered` | ⚪ |
| M-2 | 4 | One-off `setWebhook` with the secret | Telegram returns `"ok": true` | ⚪ |
| M-3 | 4 | POST to the live URL without the header | HTTP 403 | ⚪ |
| M-4 | 4 | POST `{"update_id":1}` with the correct header | HTTP 200 (no agent run) | ⚪ |
| M-5 | 4 | Real message to the bot | Log shows the update processed (200). A reply is expected; if none, check `LANGGRAPH_ASSISTANT_ID` (separate known issue) | ⚪ |

## Known risks outside this plan

| Light | Risk | Note |
|---|---|---|
| 🔴 | `LANGGRAPH_ASSISTANT_ID` defaults to `agent`, but the graph is `main_graph` | Can make M-5 show no reply even when the secret works. Not fixed here. |
| 🟡 | Tests run on SQLite only | Webhook tests don't touch the DB, so the risk is low for this change. |
| 🟡 | A leaked secret needs rotation by hand | Documented in Task 4; no automation (option A). |

## Execution log

Add one line per run: date, task, which cases changed light, and the command output summary.

| Date | Task | Changes | Notes |
|---|---|---|---|
| | | | |
