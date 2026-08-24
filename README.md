# Swapmeter

**Can a cheaper model replace the one you run in production without breaking anything?**

Swapmeter is an open source CLI that answers that question for a specific prompt you already ship. Point it at your production prompt, and it runs the prompt across your current model and a set of candidates, then reports agreement rate, cost per 1,000 calls, latency, and structured-output failure rate for each. It shows you exactly which rows the cheap models got wrong, and lets you judge those rows so the next run is scored against your judgment instead of a guess.

> **Status: design complete, v1 implementation beginning.** There is nothing to install yet. The full design is in [docs/superpowers/specs/2026-08-03-swapmeter-design.md](docs/superpowers/specs/2026-08-03-swapmeter-design.md) and the v1 implementation plan is beside it. Watch the repo if you want to follow along.

## What v1 will look like

No dataset required. You describe the task, and Swapmeter generates a synthetic input set from the prompt itself:

```yaml
version: 1

task:
  name: support-ticket-router
  prompt: |
    Classify the support ticket into exactly one category.
    Categories: billing, technical, account, other

    Ticket: {{ ticket_text }}
  inputs:
    - name: ticket_text
      description: Body of a customer support ticket, 1-3 sentences
  output:
    type: enum
    values: [billing, technical, account, other]

baseline:
  model: openai/gpt-4o

candidates:
  - anthropic/claude-haiku-4-5
  - google/gemini-2.5-flash
  - groq/llama-3.3-70b

data:
  source: synthetic
  count: 50

scoring:
  comparator: exact

thresholds:
  min_agreement: 0.95
```

Then:

```
swapmeter init --example   # scaffold a complete working config
swapmeter run              # run baseline + candidates, print the table, write the HTML report
swapmeter review           # judge the disagreements in a local web UI
swapmeter run              # re-score against your judgments, free (responses are cached)
```

When you are ready to use real traffic, change `data.source` to `file` and point at a CSV or JSONL. Nothing else changes.

## The honest caveat

On a first run, the reference answer for every row is the baseline model's own output. That means the headline number is **agreement with your current model, not correctness**. Swapmeter never calls it accuracy until you have judged the rows yourself.

This is also the on-ramp: the review loop walks you through the disagreements one at a time, and adjudicating a dozen of them takes about five minutes. That produces a small labeled golden set you never had to sit down and build, judgments are keyed to rows rather than candidates so they are reused across every model and every future run, and the report can then show you something most teams have never seen: their incumbent model's own accuracy against their own labels.

## Principles

- **Zero labeled data to get a first result.** Synthetic inputs are generated from the prompt itself.
- **Local first.** No account, no hosted service, no telemetry. Data leaves your machine only to reach the model providers you are already calling.
- **Cost honesty.** Cost preview before a run, confirmation above a threshold, and aggressive caching so re-runs are free. A tool about saving money must not surprise you with a bill.
- **Invalid output is data.** Output that fails the declared schema is reported as its own rate, not folded into mismatches. Cheap models fail structured output more often than frontier models, and that rate is one of the most decision-relevant numbers in the report.
- **One implementation.** Python 3.11+, LiteLLM wrapped behind a single provider interface, Typer, Pydantic, SQLite cache. No parallel ports. The interface is a config file and a CLI, so any stack can use it.

## Scope

v1 covers structured outputs: classification, routing, tagging, and extraction (`enum` and `json` output types).

## Roadmap

- **v2: open-ended text tasks.** Summarization, chat replies, generated copy. Requires an LLM judge, and judge calibration becomes the credibility question, so it comes after real users have shaped the requirements.
- **v2: CI integration.** A GitHub Action gating on `thresholds`. The non-zero exit code on threshold failure ships in v1 for exactly this.
- **Later:** JSON result artifact for programmatic consumption; redaction helpers for exporting real production data.

## License

Apache 2.0. See [LICENSE](LICENSE).
