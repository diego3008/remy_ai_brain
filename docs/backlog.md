# Remy AI Backlog

Work we decided to postpone, kept as **vertical slices**: each slice is a small, independently shippable behavior change (test → code → docs), not a layer.

**Priority rule:** get the core flow (Telegram → remy_api → remy_ai) working first, then harden. Items below are hardening unless marked otherwise.

**Status:** `todo` · `doing` · `done` · `shelved`. For test status use the traffic-light key in `docs/webhook-secret/test-plan.md` (⚪ not started, 🔴 failing, 🟡 weak evidence, 🟢 verified).

Run one slice at a time and stop for review after each.

---

## B-1 Webhook secret (security hardening), `shelved`

**Why:** `POST /webhook/telegram` in remy_api accepts any request. Anyone with the URL can impersonate a user, trigger paid LLM runs, and make the bot send messages.
**Decided:** 2026-11-18. Option A: the live webhook is re-registered once by hand. Planned, then shelved to focus on a working flow.
**Repo:** remy_api (code), hippocamp (docs)
**Plan:** `docs/webhook-secret/plan.md` · **Test plan:** `docs/webhook-secret/test-plan.md`

| Slice | Observable behavior when done | Status |
|---|---|---|
| B-1.1 Config | The app refuses to start if `TELEGRAM_WEBHOOK_SECRET` breaks Telegram's rules (1–256 characters, `A-Za-z0-9_-`) | todo |
| B-1.2 Startup | `set_webhook` is called with `secret_token` when the URL changes | todo |
| B-1.3 Request | `POST /webhook/telegram` returns 403 without the right header, and the agent is never called | todo |
| B-1.4 Rollout | The secret is registered live, then enforcement is deployed. M-1 to M-4 are green. `AGENTS.md` gap removed | todo |

**Order matters:** B-1.2 before B-1.3. Never deploy B-1.3 before the secret is registered with Telegram, or every real message gets 403.
**Depends on:** nothing. The assistant-ID fix (B-2) only touches `tests/test_config.py`, which B-1.1 extends.
**Resume with:** "start B-1.1", one slice at a time.

---

## B-2 Assistant ID default, `done`

**Why:** remy_api defaulted `LANGGRAPH_ASSISTANT_ID` to `agent`, but remy_ai registers the graph as `main_graph`. Without an override, no Telegram message got a reply. **This one blocks the working flow**, so it is not hardening.
**Repo:** remy_api, PR #4 merged 2026-11-18 (`f6f3b6b`)

| Slice | Observable behavior when done | Status |
|---|---|---|
| B-2.1 Default | `Settings().LANGGRAPH_ASSISTANT_ID == "main_graph"`. Test added, 7 tests pass | done |
| B-2.2 Ship it | PR merged, and Diego's real `.env` already sets `LANGGRAPH_ASSISTANT_ID=main_graph` (confirmed by Diego 2026-11-18) | done |
| B-2.3 Docs | The "assistant ID mismatch" notes in hippocamp's `AGENTS.md` are updated | done |

---

## Adding items

Copy the B-1 block. Write the **observable behavior** per slice (what a person or a test can see), not the file or layer being changed. If a slice can't be demonstrated on its own, split it differently.
