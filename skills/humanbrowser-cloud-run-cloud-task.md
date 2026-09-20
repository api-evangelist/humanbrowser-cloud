---
name: humanbrowser-cloud-run-cloud-task
description: Get a Human Browser token, check the balance, run one natural-language browser goal over A2A, poll it to a terminal state, hand the live viewer URL to a person, and know what can and cannot be undone.
api: openapi/humanbrowser-cloud-openapi.json
operations:
  - claimTrial
  - getAccount
  - getPlans
  - topUp
  - runA2ATask
method: generated
generated: '2026-09-19'
grounding: >-
  All five operationIds exist verbatim in openapi/humanbrowser-cloud-openapi.json. JSON-RPC method names,
  metadata keys, states and limits are quoted from the provider's agent card and /a2a page as captured in
  a2a/, conventions/, errors/ and rate-limits/ in this repo. The provider's own, much longer skill is saved
  verbatim next to this one (humanbrowser-cloud-human-browser.md); this file is the short, contract-grounded
  version.
---

# Run one cloud browser task with Human Browser

Two hosts: `https://humanbrowser.cloud` for the account operations, `https://agent.humanbrowser.cloud` for the work. One token (`hb_live_…`) for both, sent as `Authorization: Bearer` — never in a query string.

## 0. Know what it costs before you start

- `GET /api/plans` (`getPlans`, no auth) returns the rate sheet. The site and Terms price it as prepaid pay-as-you-go: **$0.10 per browser-minute, $4/GB residential bandwidth, $0.005 per solved CAPTCHA, AI from $0.005 per 1K tokens**. Note that `/api/plans` itself describes monthly subscriptions — the Terms are the binding document.
- A typical task is **$0.13–0.25** the first time and **$0.03–0.05** on a warm profile; hostile sites run **$0.30–0.50**. Balance only decreases while a session is live.

## 1. Get a token

- **Trial:** `POST /api/trial-balance` (`claimTrial`) with `{"email": "…"}` → `{token, balance_usd}`. One per email for life (**429** on repeat); the trial balance is revoked 14 days after issuance unless a paid top-up is made. No card.
- **Paid:** `POST /api/topup` (`topUp`, Bearer) with `{"amount_usd": 20, "method": "stripe"}` → `{url}`, a Stripe Checkout the user opens in a real browser. Any amount **$1–$2,000**; `method: crypto` routes to 0xProcessing and is **final-sale**.
- `GET /api/account` (`getAccount`) is the dashboard route and wants the `hb_session` cookie, not the bearer token; without either it answers **400 bad-token**.

## 2. Send the goal

`POST https://agent.humanbrowser.cloud/a2a` (`runA2ATask`), JSON-RPC 2.0:

```json
{"jsonrpc":"2.0","id":1,"method":"message/send",
 "params":{"message":{"role":"user","messageId":"msg-1",
   "parts":[{"kind":"text","text":"Open ifconfig.me and report the IP address"}]},
   "metadata":{"country":"gb","profile":"my-profile"}}}
```

- Put credentials in a **DataPart with `metadata.sensitive: true`** — they are injected at runtime and never echoed in artifacts or logs (screenshots are the stated exception).
- Attach files as a **FilePart** (`file.uri` public URL, or `file.bytes` base64; ≤ 8 files). Never a local path.
- Text that must land character-for-character goes inside `<verbatim>…</verbatim>` markers in the goal.
- Do **not** pass engine / model / mode knobs; the server routes from the goal. Use `metadata.callback_url` if you want the terminal envelope pushed instead of polled.
- The response is a `Task` within ~1 s with `metadata.viewer_url` (`https://humanbrowser.cloud/a/s_<id>?k=…`). **Relay that URL to the user immediately, on its own line** — it is where a human watches and takes over.

## 3. Poll to a terminal state

`{"method":"tasks/get","params":{"id":"<task id>","wait_seconds":30}}`

States: `working | submitted | input-required | completed | failed | canceled` — only the last four are terminal. **While `working`, the task is running: do not report failure.** First-run authenticated tasks on hostile sites take 3–8 minutes; cached runs 20–90 seconds. Quote `status.message` verbatim as progress.

- `input-required` (`final: true`): the agent needs an OTP / approval. Resume with `message/send` carrying `message.taskId` + `message.contextId` and the answer as a TextPart (or `{decline:true, reason}`). A human can also answer in the viewer; first writer wins; server auto-declines after `timeout_s` (default 300 s).
- `failed`: read `metadata.postmortem` (`root_cause_category`, `observed_blockers`, `retry_recommendation`) and `metadata.blocks[]` before deciding to retry. `submit_guard` and `account_lockdown` are `retryable: false`.
- Prefer `message/stream` (`Accept: text/event-stream`) for live `task` / `status-update` / `artifact-update` frames.

## 4. Handle the failures the contract declares

- **401** — missing or wrong bearer (on `/a2a` it arrives as JSON-RPC error `-32001`, which is *not* "task not found" here).
- **402 `quota_exceeded`** — balance is zero; top up. Running sessions finish; nothing is deleted.
- **503 with `retry_after_seconds`** — more than 5 concurrent sessions on the token. Reuse the live session (`force_new: false`, tasks queue up to 20 and answer **202 queued**) instead of spawning.
- **429** — only on `claimTrial`; a retry will not help.

## Rules that are not in the status codes

- **No idempotency key anywhere.** A retried `message/send` after a timeout starts or queues a second billed task on the same profile; a retried `topUp` opens a second checkout. Poll `tasks/get` instead of resending.
- **Reversibility:** `tasks/cancel` stops a non-terminal task and tears the session down — it does not undo anything already done on the target site, and consumed minutes are not refunded. Money: an **untouched** top-up is refundable within **7 days** by email; used balance, crypto and trial credit are not.
- Profiles persist cookies per token, canonicalised from the goal's first domain; pass `force_new: true` for an isolated browser.
- Acceptable use (Terms 5, docs/mcp): user-authorized tasks only — no scraping behind others' auth walls, no mass account creation, no circumventing a site's ToS.
