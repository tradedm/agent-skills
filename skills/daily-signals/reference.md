# Daily Signals Reference

Companion to [SKILL.md](SKILL.md). Read the workflow and guardrails there first.

## Authoritative sources

- [API guide](https://tradedm.com/apidocs.php)
- [OpenAPI 3.1 spec](https://tradedm.com/api/v1/client/openapi.php) — read this rather than
  trusting a cached field list; the API adds fields.

## Base URL and auth

```
https://tradedm.com/api/v1/client
Authorization: Bearer tdmk_...
```

That base URL is a prefix, not a page — opening it in a browser returns `403`, which is
correct. Append an endpoint from the table below.

Keys are created at [Account → API Access](https://tradedm.com/account.php) and shown once.
Up to 3 active keys per account. Revoking takes effect on the next request. Keys in query
strings are rejected (`401 key_in_query_not_allowed`).

## Endpoints

All `GET`. All read-only. All scoped to the key's own account.

| Endpoint | Parameters | Returns |
|---|---|---|
| `/me.php` | — | Key + account confirmation and the polling contract |
| `/signals.php` | `date` (YYYY-MM-DD), `portfolio_id` | The signal board for one trading day |
| `/portfolios.php` | — | The subscriber's portfolios and their member signals |
| `/results.php` | `model_id`, `days` (7–3650, default 30) | Scored day-by-day history |
| `/performance.php` | `model_id`, `days` | 30-day snapshot for all, or depth for one |
| `/openapi.php` | — | The spec itself (no key required) |

Envelope: `{"success": true, "data": {...}}` · errors: `{"success": false, "error": "code"}`.

## `/me.php` fields

| Field | Meaning |
|---|---|
| `key_label` | The label given when the key was created |
| `entitled_signals` | How many signals this account subscribes to |
| `server_ts` | TradeDM's unix clock — compare all `*_ts` against this |
| `window` | `pre_signal` · `active` · `post_market` · `closed` |
| `market_phase` | Inside `active` only: `pre_open` · `open` · `post_close`; else null |
| `board_date` | Trading day the next/current board belongs to |
| `signals_at_ts` | When the EARLIEST signal on this account publishes on `board_date` |
| `signals_complete_ts` | When the LAST expected signal publishes on `board_date` |
| `next_trading_day` | Next NYSE session (holiday-aware) |

## `/signals.php` context fields

| Field | Meaning |
|---|---|
| `date` | The trading day these signals belong to. **Verify before acting** |
| `portfolio_id` | Echoes the applied scope; `null` = the full subscribed board |
| `window`, `market_phase`, `board_date` | As above |
| `scored` | `true` once the board day has closed and `today_pl` is final |
| `server_ts` | TradeDM's unix clock |
| `signals_at_ts`, `signals_complete_ts` | First and last expected publish on `board_date` |
| `next_release_ts` | Next signal still outstanding; `null` = none. Earlier than `server_ts` = overdue, not absent |
| `signals[]` | Published signals |
| `pending[]` | Entitled signals not yet published today (active window), each with `release_time` and `expected_at_ts` |

## Signal fields

| Field | Meaning |
|---|---|
| `release_time` | ET clock time this signal publishes (HH:MM). 09:25 default and earliest |
| `model_id` | Handle for `/results.php` and `/performance.php` |
| `ticker`, `full_name` | The security |
| `version` | The model, e.g. `A0.08`. A Signal is Model × Symbol |
| `model_type` | `ai` or `classic` |
| `direction` | `UP` (long bias) or `DOWN` (short bias), for that day only |
| `ai_price_target` | Model's predicted closing price, or null |
| `stop_loss`, `stop_gain` | Intraday stop prices, or null when the model publishes none |
| `group_price_avg`, `votes_up`, `votes_down` | Classical-group inputs, where applicable |
| `prev_close`, `prev_day_pct`, `prev_direction`, `prev_correct`, `prev_pl` | Prior session |
| `alpha_30d` | 30-day net minus close-to-close Buy & Hold, $1,000 basis |
| `today_pl`, `today_correct` | This board day's scored result; null until `scored` is true |

## Release schedule

`09:25` ET is the platform default and the general earliest. A model may publish later on purpose
— for example to take in more pre-market or at-open data — and that time appears as the signal's
own `release_time`.

The schedule is **informational**. What may actually be read is governed solely by whether a
signal has been released; nothing is ever served early, on any endpoint, with any parameters.
`release_time` tells an agent when to come back, so it can wait for one outstanding signal
instead of polling the whole board.

## Formulas

All figures use a hypothetical **$1,000 notional per signal**, official open to official close,
before commissions, fees, slippage, and taxes.

```text
direction = UP    →  pl = (close - open) / open * 1000
direction = DOWN  →  pl = (open - close) / open * 1000

gain_loss_pct     = pl / 10          # percent of the $1,000
accuracy          = wins / scored_days
alpha             = signal_net - buy_and_hold_close_to_close
```

Entry and exit timing is the subscriber's decision; fills away from the official open and close
will differ from these figures.

## Error codes

| HTTP | `error` | Meaning |
|---|---|---|
| 401 | `unauthorized` | Bad, revoked, or inactive key |
| 401 | `key_in_query_not_allowed` | Key was passed in the URL |
| 403 | `not_permitted` | Not entitled to that model or portfolio |
| 403 | `api_disabled` | Client API is off for this environment |
| 405 | `method_not_allowed` | Only `GET` is supported |
| 422 | `validation_failed` | Bad parameter; `fields[]` names it |
| 429 | `rate_limited` | Throttled; `Retry-After` header and `retry_after` field |
| 500 | `internal_error` | Server-side fault |

## Rate limits

60 requests/minute and 5,000 per ET day, per key.

| Header | Meaning |
|---|---|
| `X-RateLimit-Limit` / `-Remaining` / `-Reset` | Per-minute budget; Reset is a unix timestamp |
| `X-RateLimit-Limit-Day` / `-Remaining-Day` / `-Reset-Day` | Per-day budget |
| `Retry-After` | Seconds to wait; sent on `429` |

Throttle from `Remaining` rather than a hard-coded ceiling.
