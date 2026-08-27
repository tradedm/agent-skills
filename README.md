# TradeDM Agent Skills

Open agent skills for the **TradeDM Client API** — the read-only interface a TradeDM
subscriber's own AI assistant uses to fetch the daily trade signals they subscribe to.

**A signal is one model's `UP` or `DOWN` direction call on one ticker for one trading day**,
published before the open, often with a price target and intraday stop prices. Every
performance figure is measured from that day's official open to its official close on a
hypothetical $1,000 notional.

A skill is a `SKILL.md` file your AI assistant reads and follows. It carries the workflow,
the vocabulary, and the guardrails, so an agent behaves consistently instead of improvising
against raw API docs.

TradeDM is a financial publication. These skills **retrieve and explain signals**. They do
not place orders, size positions, or connect to a broker — execution belongs to you, your
agent, and whatever broker tooling you choose.

## Prerequisites

- **A TradeDM subscription** with at least one signal — [tradedm.com](https://tradedm.com)
- **A TradeDM API key** — [Account → API Access](https://tradedm.com/account.php) → Create Key.
  Copy it once; it is never shown again. Store it in an environment variable
  (`TRADEDM_API_KEY`), never in a chat message or a committed file.

## Quickstart

Check that your key works before wiring up any agent:

```bash
curl -H "Authorization: Bearer tdmk_YOUR_KEY_HERE" \
  https://tradedm.com/api/v1/client/me.php
```

```jsonc
// abridged — see reference.md for every field
{"success":true,"data":{"key_label":"My ChatGPT","entitled_signals":12,
 "window":"pre_signal","board_date":"2026-08-27","signals_at_ts":1787837100}}
```

That response means you're connected: the key's label, how many signals it unlocks, and when
the next board publishes. Then `GET /signals.php` for the board itself. Full endpoint
reference: [reference.md](skills/daily-signals/reference.md) or the
[API guide](https://tradedm.com/apidocs.php).

## Install

### Skills CLI

```bash
# Interactive install
npx skills add tradedm/agent-skills

# Preview what's available
npx skills add tradedm/agent-skills --list

# Install one skill
npx skills add tradedm/agent-skills --skill tradedm-daily-signals
```

### Manual install

| Agent | Typical path |
| --- | --- |
| **Claude Code** | Copy the skill directory into `~/.claude/skills/` |
| **Cursor** | Copy or symlink into `.cursor/skills/` (project) or your user skills directory |
| **Other** | Reference the `SKILL.md` path directly in your agent prompt |

```bash
mkdir -p ~/.claude/skills
cp -r skills/daily-signals ~/.claude/skills/tradedm-daily-signals
```

### ChatGPT (Custom GPTs)

ChatGPT does not read `SKILL.md` files — a Custom GPT is configured with an **Action** for API
access and an **Instructions** field for behavior. You get both halves of this skill by putting
each where ChatGPT expects it:

1. **Action** — *Configure → Create new action → Import from URL*:
   `https://tradedm.com/api/v1/client/openapi.php`
   Then *Authentication → API Key → Auth Type **Bearer***, and paste your `tdmk_...` key.
   ChatGPT stores it server-side; the model itself never sees it.
2. **Instructions** — paste the body of
   [`skills/daily-signals/SKILL.md`](skills/daily-signals/SKILL.md) into the GPT's Instructions
   box. Same workflow, vocabulary, and guardrails as the skill file, delivered the way ChatGPT
   accepts them.

Leave the GPT shared with **Only me**. It carries your API key, and anyone you share it with
would be reading your signals on your key and against your rate limit.

## Available skills

| Name | Path | Title |
| --- | --- | --- |
| `tradedm-daily-signals` | [skills/daily-signals/](skills/daily-signals/) | Daily Trade Signals |

## Pairing with execution

This library is the **signal** half. If you want your agent to act on a signal, pair it with
whatever tooling reaches your broker — for example, Alpaca publishes
[open agent skills](https://github.com/alpacahq/alpaca-skills) for its own trading API. Your
agent composes the two: read the signal here, decide with you, execute there. TradeDM has no
part in the second half and no knowledge of it.

## Related resources

- [API guide](https://tradedm.com/apidocs.php) — quickstart, endpoint reference, setup blocks
- [OpenAPI 3.1 spec](https://tradedm.com/api/v1/client/openapi.php)
- [Terms of Service](https://tradedm.com/termsofservice.php) · [Privacy](https://tradedm.com/privacy.php)

## License

Licensed under the Apache License 2.0. See [LICENSE](LICENSE).

**What the license covers.** Apache 2.0 applies to the contents of this repository — the skill
instructions and reference documentation. It grants no rights to the TradeDM Client API itself,
to the signals, statistics, or history the API serves, or to the TradeDM name and marks. Access
to the API and its data requires an active TradeDM subscription and is governed by the
[Terms of Service](https://tradedm.com/termsofservice.php).

## Disclosure

TradeDM is a financial publication. Signals, statistics, and history are standardized,
impersonal model output provided for informational purposes only. Nothing published by
TradeDM or served by its API is investment advice, a recommendation, or an offer to buy or
sell any security. All profit and loss figures are hypothetical results of a uniform $1,000
notional per signal, measured from the official open to the official close, and exclude
commissions, fees, slippage, and taxes. Actual results will differ. Past performance does not
guarantee future results. All trading involves risk, including possible loss of principal.
You are responsible for every order you or your tools place.
