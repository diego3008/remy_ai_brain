# Telegram Webhook Secret Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
> **Execution rule for this plan:** run ONE task at a time and stop for the human's review after each task. Do not start the next task unprompted.

**Goal:** Make `POST /webhook/telegram` in remy_api accept only requests that carry Telegram's secret-token header, and register that secret with Telegram.

**Architecture:** Telegram sends the secret in the `X-Telegram-Bot-Api-Secret-Token` header on every webhook call; remy_api registers it via `set_webhook(secret_token=...)` and rejects mismatches with 403 using a FastAPI dependency (so the rejection happens outside the handler's catch-all `try/except`, which always returns 200). remy_ai is not involved. The live webhook is re-registered once by hand (option A), because `getWebhookInfo` never returns the secret and the lifespan only re-registers when the URL changes.

**Tech Stack:** FastAPI, python-telegram-bot v22, pydantic-settings, pytest + pytest-asyncio (`asyncio_mode=auto`), httpx `AsyncClient` + `ASGITransport`.

**Spec:** No separate spec file. Requirements were decided in the 2026-11-18 session (webhook-secret discussion, option A chosen) and are restated under Global Constraints. Companion file: `docs/webhook-secret/test-plan.md` (traffic-light test plan, case IDs referenced below).

## Where the work happens

| What | Repo / path |
|---|---|
| Code + tests | remy_api, worktree `~/orca/workspaces/remy_api/silverside` |
| This plan, test plan, `AGENTS.md` update | hippocamp (`~/orca/workspaces/remy_brain/hippocamp`) |

Branch in remy_api: `diego3008/webhook-secret`, created from `origin/main` (local `main` may be behind). Never commit to `main`; open a PR.

## Global Constraints

- Header name: `X-Telegram-Bot-Api-Secret-Token`. Telegram's secret rules: 1–256 characters, only `A-Z a-z 0-9 _ -`.
- Rejection status is exactly **403**. A rejected request must never reach `run_agent` or `bot.send_message`.
- Valid requests keep the existing behavior: always return 200, errors are logged and swallowed.
- The received header value and the configured secret must never appear in logs.
- Compare with `secrets.compare_digest` on **bytes** (str with non-ASCII raises `TypeError` → 500).
- Tests never hit Telegram, OpenAI or LangGraph. Fake the bot and patch `run_agent` where `src.api.telegram` uses it.
- The agent never touches `TELEGRAM_BOT_TOKEN` or the real secret. Live steps in Task 4 are run by the human.
- **Never deploy the enforcement (Task 3) before the secret is registered with Telegram (Task 4, step "register").** Merging is fine; deploying is the danger.
- Commit messages in English; end each with `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`.
- Python 3.14, run tests with `uv run pytest` from the remy_api worktree root.

## Review Focus

Inputs and conditions the spec implies but that are easy to miss, most likely first. Each has a test in the owning task.

1. Non-ASCII header value → must be 403, not a 500 from `compare_digest` (Task 3, T-WH-4).
2. Header present but empty → 403 (Task 3, T-WH-3).
3. A rejected request must not leak the received secret into logs (Task 3, T-WH-5).
4. A secret in `.env` that breaks Telegram's charset/length rules → app must fail fast at startup, not at `set_webhook` (Task 1, T-CFG-2).
5. Existing registration has no secret → after deploy every real message would get 403 (Task 4, ordering + M-2/M-4).

## File Structure

| File | Repo | Responsibility |
|---|---|---|
| `src/config.py` (modify) | remy_api | Validate `TELEGRAM_WEBHOOK_SECRET` format |
| `src/api/webhook_registration.py` (create) | remy_api | `ensure_webhook_registered(...)`: the register-if-URL-changed logic moved out of `main.py`, now passing the secret |
| `main.py` (modify) | remy_api | Lifespan calls `ensure_webhook_registered` |
| `src/api/telegram.py` (modify) | remy_api | `verify_webhook_secret` dependency on the webhook route |
| `tests/test_config.py` (extend: created by the assistant-ID fix, commit `c80054f`), `tests/test_webhook_registration.py`, `tests/test_webhook.py` (create) | remy_api | One test file per slice |
| `docs/webhook-fix-notes.md` (modify) | remy_api | Secret section + one-off registration command |
| `AGENTS.md` (modify) | hippocamp | Remove the "secret never checked" gap, note the header check |

---

### Task 1: Fail fast on an invalid secret (config slice)

**Files:**
- Modify: `src/config.py:12`
- Test: `tests/test_config.py` (already exists once the assistant-ID fix `c80054f` is merged; add to it and reuse its `REQUIRED` dict instead of a new `VALID`)

**Interfaces:**
- Consumes: nothing.
- Produces: `Settings.TELEGRAM_WEBHOOK_SECRET: str`, guaranteed 1–256 chars of `[A-Za-z0-9_-]`. Tasks 2 and 3 rely on that guarantee.

- [ ] **Step 1: Create the branch** in the silverside worktree: `git status` must be clean, then `git fetch origin && git switch -c diego3008/webhook-secret origin/main`.
- [ ] **Step 2: Write the failing tests** in `tests/test_config.py`. Build `Settings(_env_file=None, **{**REQUIRED, "TELEGRAM_WEBHOOK_SECRET": secret})`, where `REQUIRED` (already in `tests/test_config.py`) supplies every required field. Tests:
  - `test_valid_secret_accepted`: `"abc_DEF-123"` is stored unchanged.
  - `test_invalid_secret_rejected` parametrized over `""`, `"has space"`, `"ñandú"`, `"a" * 257`: each raises `pydantic.ValidationError`.
- [ ] **Step 3: Run** `uv run pytest tests/test_config.py -v`. Expected: `test_invalid_secret_rejected` cases FAIL (no validation yet).
- [ ] **Step 4: Implement** the constraint on `TELEGRAM_WEBHOOK_SECRET` in `src/config.py` using `pydantic.Field(min_length=1, max_length=256, pattern=r"^[A-Za-z0-9_-]+$")`.
- [ ] **Step 5: Run** `uv run pytest -v`. Expected: all pass (6 CRUD + new config tests).
- [ ] **Step 6: Commit** `git add src/config.py tests/test_config.py && git commit -m "Validate TELEGRAM_WEBHOOK_SECRET format at startup"`.

---

### Task 2: Register the webhook with the secret (startup slice)

**Files:**
- Create: `src/api/webhook_registration.py`
- Modify: `main.py:31-50` (lifespan registration block)
- Test: `tests/test_webhook_registration.py` (create)

**Interfaces:**
- Consumes: validated `TELEGRAM_WEBHOOK_SECRET` from Task 1.
- Produces: `async def ensure_webhook_registered(bot: telegram.Bot, webhook_url: str, secret_token: str) -> bool`. Returns `True` if `set_webhook` succeeded, `False` if skipped (URL unchanged) or swallowed `RetryAfter`.

This extracts existing logic (see `docs/webhook-fix-notes.md`) so it can be tested without importing `main.py`, which reads settings at import time. Behavior other than `secret_token` must not change.

- [ ] **Step 1: Write the failing tests.** Use a fake bot (`AsyncMock`s for `get_webhook_info` and `set_webhook`; `get_webhook_info` returns an object with a `.url`):
  - `test_registers_with_secret_when_url_differs`: `set_webhook` awaited once with `url=<webhook_url>`, `secret_token=<secret>`, `allowed_updates=Update.ALL_TYPES`, `drop_pending_updates=True`; returns `True`.
  - `test_skips_when_url_unchanged`: `set_webhook` not awaited; returns `False`.
  - `test_retry_after_is_swallowed`: `set_webhook` raises `telegram.error.RetryAfter(5)`; no exception escapes; returns `False`.
- [ ] **Step 2: Run** `uv run pytest tests/test_webhook_registration.py -v`. Expected: FAIL (`ModuleNotFoundError`).
- [ ] **Step 3: Implement** `ensure_webhook_registered` in `src/api/webhook_registration.py` by moving the `get_webhook_info` / `set_webhook` / `RetryAfter` / logging logic out of `main.py` and adding `secret_token`. Add a comment: an unchanged URL is skipped, so an already-registered webhook keeps its old secret until re-registered manually (see Task 4).
- [ ] **Step 4: Wire it in** `main.py`: the lifespan calls `await ensure_webhook_registered(ptb.bot, webhook_url, SETTINGS.TELEGRAM_WEBHOOK_SECRET)` in place of the inline block. Keep `async with ptb:` / start / stop / `get_engine().dispose()` untouched.
- [ ] **Step 5: Run** `uv run pytest -v`. Expected: all pass.
- [ ] **Step 6: Smoke check (manual M-1)**: `uv run fastapi dev main.py` starts with the real `.env` and logs `Webhook already registered` or `Webhook registered`. Stop it after. Record the result in `test-plan.md`.
- [ ] **Step 7: Commit** `git add src/api/webhook_registration.py main.py tests/test_webhook_registration.py && git commit -m "Register Telegram webhook with secret_token"`.

---

### Task 3: Reject webhook calls without the secret (request slice)

**Files:**
- Modify: `src/api/telegram.py` (add dependency, attach to the route)
- Test: `tests/test_webhook.py` (create)

**Interfaces:**
- Consumes: `Settings.TELEGRAM_WEBHOOK_SECRET` via `get_settings` (from `src.config`).
- Produces: `async def verify_webhook_secret(x_telegram_bot_api_secret_token: Annotated[str | None, Header()] = None, settings: Annotated[Settings, Depends(get_settings)]) -> None` in `src/api/telegram.py`. Raises `HTTPException(403, "Invalid webhook secret")`.

- [ ] **Step 1: Write the test harness** in `tests/test_webhook.py`: a fixture that builds a `FastAPI()` with only `telegram_router`, sets `app.state.ptb` to an object whose `.bot` is a fake with `send_message = AsyncMock()` (if `Update.de_json` rejects the fake, use a real `telegram.Bot("123:abc")` with `send_message` patched), overrides `get_settings` via `app.dependency_overrides` to return an object with `TELEGRAM_WEBHOOK_SECRET = "s3cret_-ok"`, and monkeypatches `src.api.telegram.run_agent` to an `AsyncMock(return_value="hola de vuelta")`. Include a `TEXT_UPDATE` payload (`update_id`, `message` with `message_id`, `date`, `chat={"id": 42, "type": "private"}`, `from={"id": 7, "is_bot": False, "first_name": "T"}`, `text="hola"`).
- [ ] **Step 2: Write the failing tests** (post `TEXT_UPDATE` to `/webhook/telegram`):
  - `test_missing_header_returns_403`: status 403, `run_agent` not awaited, `send_message` not awaited. (T-WH-1)
  - `test_wrong_header_returns_403`. (T-WH-2)
  - `test_empty_header_returns_403`. (T-WH-3)
  - `test_non_ascii_header_returns_403`: send the header as bytes (`"é".encode()`); status 403, not 500. (T-WH-4)
  - `test_rejected_request_does_not_log_secrets` using `caplog`: neither the received value nor `"s3cret_-ok"` appears in any record. (T-WH-5)
  - `test_correct_header_runs_agent_and_replies`: 200; `run_agent` awaited once with an `IncomingMessage` whose `chat_id == "42"` and `content == "hola"`; `send_message` awaited with `chat_id="42"` and `text="hola de vuelta"`. (T-WH-6)
  - `test_correct_header_agent_failure_still_returns_200`: `run_agent` raises; status 200. (T-WH-7)
  - `test_correct_header_non_text_update_is_ignored`: payload without `text`; 200, `run_agent` not awaited. (T-WH-8)
- [ ] **Step 3: Run** `uv run pytest tests/test_webhook.py -v`. Expected: the four 403 tests and the log test FAIL (everything currently returns 200); the three "correct header" tests pass already, which proves the harness works.
- [ ] **Step 4: Implement** `verify_webhook_secret` in `src/api/telegram.py`. A missing or empty header raises 403; otherwise compare `received.encode()` with `settings.TELEGRAM_WEBHOOK_SECRET.encode()` using `secrets.compare_digest`. On rejection `logger.warning` a fixed message that includes neither value. Attach with `@telegram_router.post("/telegram", dependencies=[Depends(verify_webhook_secret)])`. Do not move anything into the existing `try/except`.
- [ ] **Step 5: Run** `uv run pytest -v`. Expected: all pass.
- [ ] **Step 6: Commit** `git add src/api/telegram.py tests/test_webhook.py && git commit -m "Reject webhook calls without the Telegram secret token"`.

---

### Task 4: Roll out safely and update the docs (ops slice)

**Files:**
- Modify (remy_api): `docs/webhook-fix-notes.md`
- Modify (hippocamp): `AGENTS.md` (Known gaps list and the Telegram webhook bullet under remy_api architecture)

**Interfaces:**
- Consumes: Tasks 1–3 merged on `diego3008/webhook-secret`.
- Produces: a runbook for a one-off `setWebhook` call and the verified live state.

Steps marked **(human)** need real credentials; the agent prepares text and the human runs it.

- [ ] **Step 1: Document** in `docs/webhook-fix-notes.md` a "Webhook secret" section: what the header is, why `getWebhookInfo` can't confirm the secret, and the rotation procedure (change `.env`, re-run the command below). Include this command, which reads values from the environment so nothing is pasted into chat:
  ```bash
  curl -sS "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook" \
    --data-urlencode "url=${WEBHOOK_URL}/webhook/telegram" \
    --data-urlencode "secret_token=${TELEGRAM_WEBHOOK_SECRET}"
  ```
  Omitting `allowed_updates` keeps the previously registered list. Call it once: Telegram rate-limits `setWebhook`.
- [ ] **Step 2: Update `AGENTS.md`** in hippocamp: remove the "`TELEGRAM_WEBHOOK_SECRET` is loaded but never passed…" gap bullet; state in the webhook bullet that the header is verified (403 on mismatch).
- [ ] **Step 3 (human): Pre-flight.** Confirm the secret in the real `.env` passes Task 1's rule (start the app once or run `uv run python -c "from src.config import get_settings; get_settings()"`). Fixes M-1.
- [ ] **Step 4 (human): Register BEFORE deploying enforcement.** Run the curl above. Expect `{"ok":true,"result":true,...}` (M-2). Old code ignores the header, so there is no downtime.
- [ ] **Step 5 (human): Deploy / restart** the branch, then verify against the real URL (tunnel or host):
  - M-3: `curl -i -X POST "$WEBHOOK_URL/webhook/telegram" -H 'Content-Type: application/json' -d '{"update_id":1}'` → `403`.
  - M-4: same request plus `-H "X-Telegram-Bot-Api-Secret-Token: $TELEGRAM_WEBHOOK_SECRET"` → `200` (a payload without a message is ignored, so no agent run or cost).
  - M-5: send the bot a real message and check the server log shows the update was processed. If there is no reply but the log shows 200, check `LANGGRAPH_ASSISTANT_ID` (known mismatch with `main_graph`); that is a separate problem from the secret.
- [ ] **Step 6: Rollback note** (already in the docs from Step 1): reverting the deploy is enough; the registered secret is harmless to old code.
- [ ] **Step 7: Commit** remy_api docs (`git commit -m "Document Telegram webhook secret and one-off registration"`) and the hippocamp `AGENTS.md` change separately in its own repo.

---

## Self-review (done)

- **Spec coverage:** header check (T3), registration with secret (T2), format validation (T1), option A manual registration + docs + `AGENTS.md` update (T4). Every Review Focus line has a case ID and an owning task.
- **Type consistency:** `ensure_webhook_registered(bot, webhook_url, secret_token) -> bool` and `verify_webhook_secret` are named identically wherever they appear. Test names match the case IDs in `test-plan.md`.
- **Proportion:** four tasks, each a vertical slice (config boundary, startup, request, rollout) that can be reviewed and merged on its own. The only deliberate refactor is the extraction in Task 2, done for testability.
