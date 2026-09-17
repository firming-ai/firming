# Firming

**Firm prices on AI inference.**

[![CI](https://github.com/firming-ai/firming/actions/workflows/ci.yml/badge.svg)](https://github.com/firming-ai/firming/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

Firming pays you for the slack in your AI bill. Most backend inference — evals, embeddings, backfills, extraction, agent runs — could be served on a venue's discount lane with a rescue behind it, and nobody would notice. Today that option is either built by hand (a retry ladder, a queue, a fallback) or thrown away, and the bill pays list for it.

Firming's quote is a bid for that option: **20 points under list, firm for the month, same models, same speed budget, never worse than list.** You install a sidecar inside your own perimeter, keep your own keys, and settle once a month against your own venue invoice.

**[firming.ai](https://firming.ai/)** · **[Documentation](https://firming-ai.github.io/firming/)** · [Spec](SPEC.md) · [Spread Board](https://github.com/firming-ai/firming/blob/board-data/nightly/BOARD.md)

## The guarantee

*The sidecar is in build — see [Status](#status). This is the contract it implements.*

| | |
| --- | --- |
| **Price** | 20% under the venue's list price on every lane on that month's rate sheet. The sheet is dated and published in advance; the price is firm for the month. |
| **Speed** | Every request carries a budget (60 seconds by default). The sidecar serves it on the venue's discount lane and rescues it at the standard lane if the discount lane does not fill in time. You never wait past the budget. |
| **Floor** | Never worse than list. A rescued request is billed at the guaranteed price; the difference is Firming's risk, not yours. |
| **Perimeter** | The sidecar runs in your infrastructure and talks to the venues with your keys. There is no proxy and no third party in the data path. Nothing about your prompts leaves your network. |
| **Receipts** | Every request writes a receipt — list price, guaranteed price, lane served, latency. Receipts settle into one statement per lane on the 1st of the month, reconciled line by line to your venue invoice, and the difference is charged or credited. |
| **Fail-open** | If the sidecar or the rate feed is unreachable, requests go straight to the standard lane at list. Those windows are excluded from the guarantee and shown on the statement. |

The customer-side change is one environment variable:

```bash
firming serve --port 8787            # pulls the month's rate sheet, prints the eligible lanes
export OPENAI_BASE_URL=http://localhost:8787/v1
```

Requests that must not be deferred, even inside the budget, carry `x-firming: standard` and are passed through at list.

## FILL

Firm prices need a market read behind them. **FILL** is Firming's index: the fill rate of the venues' discount lanes — how often a request placed on the discount lane comes back inside the budget — measured continuously, per venue and model, from more than one vantage. FILL sets the rate sheet, and the rate sheet is the price. The public marks and each month's sheet are published on the [`board-data`](https://github.com/firming-ai/firming/tree/board-data) branch of this repository.

## Status

The sidecar and the rate feed are in build; they ship in the [`firming`](https://pypi.org/project/firming/) package alongside the batch client. Phase one covers OpenAI and Gemini.

What is in this repository today:

- **`src/`** — the batch client, `pip install firming` (0.4.0; renamed from `offpeak`, whose last release is 0.3.0 — same API, new import). It is the first ladder Firming built: place a job on the venue's batch lane at 50% off list, and rescue it at the standard lane before the deadline if the batch is at risk. It stays supported and is documented below.
- **`web/`** — the site and the remote MCP connector.
- **`board-data`** — the daily Spread Board, the price-sheet watch and the FILL output, written by workflows only.
- **[SPEC.md](SPEC.md)** — the deadline and receipt semantics the batch client implements.

## The batch client

[![PyPI](https://img.shields.io/pypi/v/firming)](https://pypi.org/project/firming/)

```bash
pip install "firming[all]"        # OpenAI + Anthropic venues
pip install "firming[anthropic]"  # or one provider
pip install "firming[openai]"
```

```python
import firming

jobs = [firming.job("claude-haiku-4-5", f"Summarize:\n\n{doc}") for doc in docs]

print(firming.quote(jobs, deadline="06:00"))   # list vs batch, per venue, before any call
results = firming.run(jobs, deadline="06:00")  # batch lane, rescued at list if at risk
print(firming.receipt(results))                # list cost, paid cost, captured spread
```

- **Know the price before you spend it.** `quote(jobs, deadline=...)` prices a run against the published sheets with no API calls and no key. Token counts come from the job where it knows them; otherwise the figure is a labeled estimate, and a quote with no output signal is marked a **floor**. Pass `assumed_output_ratio=` or `metadata={"expected_output_tokens": ...}` for a priced estimate.
- **One argument, not a workflow.** `run(jobs, deadline=...)` handles batching, submission, polling, collection and result matching across providers. Results come back in input order, each with a per-job `Receipt`.
- **The deadline is guarded, not hoped for.** When the remaining window reaches the **risk buffer** (default 15% of the window, clamped to 1–10 minutes), unfinished jobs are cancelled and re-run on the standard lane at list. Set `fallback="none"` to report them instead. Deadlines take `"06:00"`, `"4h"`, ISO 8601, `datetime` or `timedelta` — the full semantics are in [SPEC.md](SPEC.md).
- **Every run settles a receipt.** List cost, paid cost, captured spread — arithmetic against public price sheets, not estimates. Unknown models settle with `cost = None` rather than a guess; register your own with `firming.prices.register_price("my-fine-tune", input_per_m=4.0, output_per_m=16.0)`.
- **Your keys, your perimeter.** The client talks directly to the providers with your own API keys (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or a configured client such as `OpenAIBatch(client=my_client)`). No proxy, no third party in the data path.
- **Zero-dependency core.** Provider SDKs load only via extras.

**When the process can't wait.** `run()` blocks for as long as the batch takes — right for a Temporal activity or an Airflow task, wrong for a laptop that sleeps, a CI step with a timeout, or a serverless function. Keep the run's state as a value instead:

```python
ticket = firming.submit(jobs, deadline="06:00")   # returns immediately
ticket.save("run.json")                            # disk, a DB row, S3 — anywhere

# later, in another process
ticket = firming.Ticket.load("run.json")
results = firming.collect(ticket)                  # same deadline, same fallback
print(firming.receipt(results))
```

`collect(ticket, wait=False)` is one non-blocking sweep; `status(ticket)` peeks. `run()` is exactly `collect(submit(...))`. The ticket carries handles, never keys.

**More venues.** Five further venues are in the tree and opt-in — each wants its own key and extra, and is passed to `run()` explicitly:

| Venue | Models | Extra | Key | Lane |
| --- | --- | --- | --- | --- |
| `groq:batch` | `openai/gpt-oss-*`, `groq/*` | `groq` | `GROQ_API_KEY` | batch, 24h–7d window |
| `mistral:batch` | `mistral-*`, `codestral`, … | `mistral` | `MISTRAL_API_KEY` | batch, window in hours |
| `gemini:batch` | `gemini-*` | `gemini` | `GEMINI_API_KEY` | batch, 24h |
| `qwen:batch` | `qwen*` (Model Studio spelling) | `qwen` | `DASHSCOPE_API_KEY` | batch, 24h–336h window, per-region |
| `deepseek:clock` | `deepseek-*` | `deepseek` | `DEEPSEEK_API_KEY` | **clock** — no batch API; priced by the hour, held until the cheap window |

```python
from firming.venues import DeepSeekClock, QwenBatch

results = firming.run(jobs, deadline="06:00", venues=[DeepSeekClock(), QwenBatch(region="intl")])
```

DeepSeek is the one venue that prices by the clock rather than by lane: weekday peak is 01:00–04:00 and 06:00–10:00 UTC, everything else is half the peak rate, and the driver holds a job until the rate is cheap rather than uploading it anywhere. Neither DeepSeek nor Qwen has a live receipt yet.

**Quote from the CLI.**

```bash
python -m firming quote --model gpt-5.6-luna --input-tokens 800 --output-tokens 200 --jobs 5000
```

Prices are a bundled snapshot of public list sheets; providers change them, so verify and override at runtime via `firming.prices`. The sheet also carries what venues charge for *urgency* — `get_fast_price()`, `urgency_spread()` — and flags promotional list prices with the date and price they decay to.

## Contributing

Issues and PRs welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). Spec changes start as issues against [SPEC.md](SPEC.md). `main` is protected: branch, PR, four green checks; tests are network-free.

## License

Apache-2.0 © Firming
