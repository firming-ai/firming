# Firming

**Firm prices on AI inference.** 20% under list, same models, same speed,
never worse than list — a sidecar inside your perimeter, on your keys, settled
monthly against your own invoice. The sidecar is in build; see the
[README](https://github.com/firming-ai/firming#the-guarantee) for the contract
it implements and the [Roadmap](roadmap.md) for what exists today.

These docs cover the shipped part: the **batch client**, `pip install firming`.
It places a job on a venue's batch lane at 50% off list and rescues it at the
standard lane before the deadline if the batch is at risk — the first ladder
Firming built, and the one the sidecar generalises.

```python
import firming

jobs = [firming.job("claude-haiku-4-5", f"Summarize:\n\n{d}") for d in docs]

print(firming.quote(jobs, deadline="06:00"))   # list vs batch, per venue, before any call
results = firming.run(jobs, deadline="06:00")  # batch lane, rescued at list if at risk
print(firming.receipt(results))                # list cost, paid cost, captured spread
```

- **[Quickstart](quickstart.md)** — install, quote, run, read the receipt.
- **[The Spread Board](spread-board.md)** — the token spreads, marked daily against open grid data.
- **[Spec](spec.md)** — deadline semantics, statuses, receipts.
- **[API reference](reference.md)** — every public symbol.
- **[Roadmap](roadmap.md)** — what exists, what does not, and what is being built.

## What the client guarantees

**One `Result` per job, always.** Provider failures at submit, poll, cancel or
sync are captured, not raised. Affected jobs take the sync fallback where the
deadline still allows it, and otherwise return failed with the provider's
message attached. Exceptions are reserved for programming errors — a deadline
in the past, or a model no venue supports.

**Your keys, your perimeter.** `firming` talks straight to the providers with
your own credentials. There is no proxy and no third party in the data path.

**Receipts are arithmetic, not estimates.** Every figure traces to a published
price sheet, and a model that is not on one settles as `None` rather than a
guess.

!!! note "Renamed from `offpeak`"
    Releases up to 0.3.0 were published as `offpeak`. 0.4.0 is the same client
    under the `firming` name: `pip install firming`, `import firming`. The
    `offpeak` package receives no further releases.
