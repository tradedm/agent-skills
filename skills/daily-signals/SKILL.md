---
name: tradedm-daily-signals
description: >
  Fetch and interpret the daily trade signals a TradeDM subscriber is entitled
  to — the day's board, portfolio groupings, scored history, and model
  statistics — over the read-only TradeDM Client API. Use this skill whenever a
  TradeDM subscriber asks their agent to retrieve, summarize, monitor, or act on
  their signals, including handing them to separate broker tooling for
  execution. TradeDM publishes signals; it never places, sizes, or times orders.
---

# TradeDM Daily Signals

Use this skill when a TradeDM subscriber wants their agent to read the signals they subscribe to.

This skill is written for you, a TradeDM subscriber working with your own account, your own API
key, and your own agent. Your agent's job here is **retrieval and honest interpretation**: fetch
the board, say plainly what the models called, and hand the decision back to you.

TradeDM is a financial publication. It grades signals; it does not manage anyone's money. This
skill is read-only and touches no broker.

---

## 0 - How your AI agent should use this skill

1. **Verify the key first.** Every session starts with `GET /me.php`. A revoked or mistyped key
   fails loudly here instead of quietly returning an empty board later.

2. **Know what it is fetching.** A signal is one model's `UP` / `DOWN` direction call on one
   ticker for **one trading day**. It is not a standing position, not a target price to chase,
   and not advice.

3. **Check the date before treating a board as actionable.** With no `?date=`, `/signals.php`
   returns the most recent *released* board — which before the morning release is **yesterday's**.
   Your agent verifies `data.date` is the trading day you intend and says so out loud.

4. **Respect the release schedule instead of assuming one.** 9:25 AM ET is the platform default
   and the earliest; individual signals may publish later on purpose. Your agent reads
   `signals_at_ts`, `signals_complete_ts`, `next_release_ts`, and per-signal `release_time`
   rather than hard-coding an hour.

5. **Report the whole board, including what is missing.** Signals that have not published yet
   appear in `pending` with their own expected time. "Not published yet" and "you are not
   subscribed" are different facts and your agent never blurs them.

6. **Never invent, infer, or extrapolate a signal.** If a ticker is not in the response, there is
   no signal for it. Your agent does not guess direction from price action, from yesterday's
   call, or from a model's track record.

7. **Keep the sizing and timing decisions with you.** TradeDM publishes no position size and no
   entry/exit schedule. If you ask "how much should I buy?" or "when should I get out?", your
   agent answers that TradeDM does not provide that, and works from *your* stated rules instead.

8. **Hand off execution explicitly.** If you want orders placed, your agent switches to whatever
   broker tooling you use and follows that tool's own confirmation and safety rules. It states
   clearly when it is leaving TradeDM data and entering broker territory.

9. **Carry the disclosure into its output.** Any summary, table, or report your agent produces
   from this data includes the disclosure in §8.

10. **Never put the key in a URL.** It goes in the `Authorization` header. The API rejects
    query-string keys outright, and URLs end up in logs and history.

---

## 1 - Prerequisites

- **A TradeDM subscription** with at least one signal.
- **A TradeDM API key** — Account → API Access → Create Key. Shown once. Stored in an
  environment variable (`TRADEDM_API_KEY`) or your agent's secret store, **never pasted into a
  chat message and never committed**.
- **Base URL**: `https://tradedm.com/api/v1/client`
- **Auth header**: `Authorization: Bearer tdmk_...`
- **Network access** to `tradedm.com`. No SDK required — every endpoint is a plain `GET`
  returning JSON.

---

## 2 - What a signal is

| Field | Meaning |
|---|---|
| `direction` | `UP` = long bias, `DOWN` = short bias, for **that trading day only** |
| `ai_price_target` | The model's predicted closing price, when it publishes one |
| `stop_loss` / `stop_gain` | Intraday stop prices, when the model supports them. Null means the model published none — not "no risk" |
| `release_time` | ET clock time this signal publishes (default 09:25, may be later by design) |
| `alpha_30d`, `prev_pl`, `today_pl` | Grading figures, on the $1,000 basis described below |

**The measurement basis.** Every performance number TradeDM publishes — `today_pl`, `prev_pl`,
`alpha_30d`, everything in `/results.php` and `/performance.php` — is measured from that day's
**official opening price to its official closing price**, on a hypothetical **$1,000 notional per
signal**, excluding commissions, fees, slippage, and taxes.

**The basis is not a schedule.** It is how the track record was computed, not an instruction about
when to trade. TradeDM does not tell you when to enter or exit, what order types to use, how close
to the bell to work, or whether to use the published stops. Those choices are yours, your agent's,
and your broker's. Fills at any other moment will differ from the open-to-close basis, and that
difference belongs to you.

Your agent states this basis whenever it quotes a P/L figure, and never presents an open-to-close
number as the result you would have achieved.

---

## 3 - Source-of-truth references

| Source | URL | Used for |
|---|---|---|
| API guide | https://tradedm.com/apidocs.php | Quickstart, endpoint reference, polling etiquette |
| OpenAPI 3.1 spec | https://tradedm.com/api/v1/client/openapi.php | Machine-readable contract; import target for GPT Actions |
| Terms of Service | https://tradedm.com/termsofservice.php | Publication terms |
| `reference.md` | alongside this file | Endpoint table, field glossary, error codes, formulas |

Read the live spec rather than trusting a cached copy of the field list — the API adds fields.

---

## 4 - Workflow

### Phase 1: Verify the key and read the clock

`GET /me.php` → confirm `success: true`, then note:

- `key_label`, `entitled_signals` — the right key on the right account
- `window` — `pre_signal` (today's board not out yet) · `active` · `post_market` · `closed`
- `board_date` — the trading day the next/current board belongs to
- `signals_at_ts` / `signals_complete_ts` — when the first and last of your signals publish
- `server_ts` — TradeDM's clock; compare timestamps against this, never the local clock

If this call fails with `401`, stop. Do not retry with the same key; the key is revoked,
mistyped, or the account is inactive.

### Phase 2: Fetch the board

`GET /signals.php` (add `?portfolio_id=N` to scope to one of your portfolios).

Then, **before presenting anything as actionable**:

1. **Verify `data.date`** is the trading day you intend. If `window` is `pre_signal`, the board
   is yesterday's — say so and wait for `signals_at_ts`.
2. **Read `pending`.** Each entry has `release_time` and `expected_at_ts`. Report how many
   signals are still expected and when.
3. **Note `next_release_ts`.** Null means nothing is outstanding. A value earlier than
   `server_ts` means a signal is *overdue*, not absent — TradeDM releases late arrivals when
   they land.

### Phase 3: Present the board

Report every signal with ticker, model version, direction, and any target and stop prices.
Separate published signals from pending ones. Do not reorder by "best" or filter to "the good
ones" unless asked — the subscriber chose these signals deliberately.

### Phase 4: Context on request

- `GET /results.php?model_id=N&days=90` — scored day-by-day history
- `GET /performance.php` — 30-day snapshot for every signal; `?model_id=N` for depth
- `GET /portfolios.php` — how the subscriber grouped their signals

Use these when asked for context. Do not dump history into every response.

### Phase 5: Hand off execution (optional)

If the subscriber wants orders placed, your agent:

1. States plainly that it is leaving TradeDM data and moving to broker tooling.
2. Restates the subscriber's own rules — sizing, order types, timing, risk limits — and confirms
   them. These come from the subscriber, never from TradeDM.
3. Passes the signal facts through unchanged: ticker, direction, and the published stops if the
   subscriber wants them used.
4. Follows the broker tool's own preview and confirmation rules. This skill does not override
   another tool's safety gates, and it never asks for them to be skipped.

### Phase 6: Review

After the close, `scored: true` on a board means `today_pl` is final for that day. Report results
on the stated basis, plainly, without characterizing a day as good, bad, or predictive.

---

## 5 - Execution rules

### Authentication

- Key travels in `Authorization: Bearer tdmk_...`. Never in a query string — the API returns
  `401 key_in_query_not_allowed`.
- Read the key from the environment. Never echo it, log it, or include it in a summary.
- A `401` is terminal for that key. Do not retry it in a loop.

### Freshness

- Signals are **single-day**. A board older than the trading day you intend is stale and must not
  be acted on.
- Verify `data.date`; treat `window: "pre_signal"` as "not out yet."
- Asking for today is not evidence. Only the returned `date` is.

### Polling

- Poll after the moment you are waiting on — `signals_at_ts`, `next_release_ts`, or a specific
  signal's `expected_at_ts` — **plus 0–60 seconds of random jitter**.
- Do not poll on a fixed short interval all morning. Wait for the specific outstanding signal.
- Compare against `server_ts`, not the local clock.

### Rate limits

- 60 requests/minute and 5,000/day per key.
- Every response carries `X-RateLimit-Remaining` and `X-RateLimit-Reset` (plus `-Day` variants).
  Slow down as Remaining approaches zero rather than waiting to be throttled.
- On `429`, honor `Retry-After`. Back off; do not hammer.

### Error handling

| Code | Meaning | Action |
|---|---|---|
| `401 unauthorized` | Bad, revoked, or inactive key | Stop. Tell the subscriber to check Account → API Access |
| `401 key_in_query_not_allowed` | Key was in the URL | Move it to the header |
| `403 not_permitted` | Not entitled to that model or portfolio | Do not probe other ids. Report it plainly |
| `403 api_disabled` | API is off for this environment | Stop and report |
| `422 validation_failed` | Bad `date` (malformed or future) | Fix the parameter |
| `429 rate_limited` | Throttled | Honor `Retry-After` |
| `500 internal_error` | Server-side fault | Report it; do not retry in a loop |

An empty `signals` array is a valid answer, not an error. It means nothing has published yet for
that board.

---

## 6 - Output contract

When reporting a board, your agent includes:

- The **board date** and whether it is today's or an earlier day
- Each signal: ticker, model version, direction, target and stop prices when present
- **Pending** signals with their expected times, counted separately
- The disclosure from §8

When reporting performance, your agent additionally states the basis in words:
*"hypothetical $1,000 notional per signal, official open to official close, before costs."*

---

## 7 - Anti-patterns

- **NEVER** present a signal as investment advice, a recommendation, or a prediction of profit.
- **NEVER** state or imply that TradeDM recommends a position size, an entry time, an exit time,
  or an order type. It publishes none of those.
- **NEVER** invent a signal for a ticker absent from the response, or infer one from price action,
  a prior day, or a track record.
- **NEVER** act on a board without checking `data.date` — signals are single-day calls.
- **NEVER** put the API key in a URL, a log line, a commit, or a chat message.
- **NEVER** present an open-to-close figure as the result the subscriber would have obtained.
- **NEVER** characterize a signal or model as safe, reliable, low-risk, or due for a win.
- **NEVER** probe model or portfolio ids the subscriber is not entitled to; a `403` is an answer,
  not an invitation to enumerate.
- **NEVER** place, modify, or cancel an order from this skill. It is read-only.
- **NEVER** skip or talk a subscriber out of a broker tool's confirmation step.
- **NEVER** poll in a tight loop; wait for the published release times.

---

## 8 - Disclosures, safety, and data handling

### Required disclosure

> **Important disclosure:** TradeDM is a financial publication. Signals and statistics are
> standardized, impersonal model output provided for informational purposes only, and are not
> investment advice, a recommendation, or an offer to buy or sell any security. All profit and
> loss figures are hypothetical results of a uniform $1,000 notional per signal, measured from
> the official open to the official close, excluding commissions, fees, slippage, and taxes;
> actual results will differ. Past performance does not guarantee future results. All trading
> involves risk, including possible loss of principal. You are responsible for every order you
> or your tools place.

Your agent includes this in every board summary, performance report, and review it produces.

### Credentials and data handling

- The key is read from the environment or a secret store, never from chat.
- Your agent refuses a key offered in plain text and asks for it to be set in the environment.
- Signal data is the subscriber's own subscription content. Your agent does not republish it,
  post it, or send it to third parties.
- The API is read-only: nothing your agent does here can change the subscriber's account,
  subscriptions, or portfolios.

---

## 9 - Related files

| File | Description |
|---|---|
| `reference.md` | Endpoint table, response fields, error codes, formulas |

### Companion tooling

Execution is deliberately outside this skill. Pair it with whatever reaches your broker — for
example, Alpaca publishes [open agent skills](https://github.com/alpacahq/alpaca-skills) for its
own trading API, with their own preview and confirmation gates. Your agent composes the two:
read here, decide with the subscriber, act there.
