# Swapmeter: Design Document

**Date:** 2026-08-03
**Status:** Approved, ready for implementation planning
**Author:** Tom Tague

---

## 1. Summary

Swapmeter is an open source CLI that answers one question: **can a cheaper model replace the one I am running in production without breaking anything?**

You point it at a prompt you already ship. It runs that prompt across your current model and a set of candidate models, compares the outputs, and reports agreement rate, cost, latency, and structured-output failure rate for each candidate. It then shows you exactly which rows the cheap models got wrong, and lets you judge those rows so the next run is scored against your judgment rather than a guess.

Terminology is deliberately plain. The model you ship today is the **baseline**. The models you are considering are **candidates**. The step where you adjudicate differences is **review**.

---

## 2. Goals and non-goals

### Goals

1. Produce a defensible cost-versus-quality answer for a specific production prompt.
2. Require zero pre-existing labeled data to get a first result.
3. Give the user a path from "directional answer" to "defensible answer" that costs minutes, not days.
4. Run entirely locally. No account, no hosted service, no telemetry.
5. Be trustworthy about what it is measuring, including where the measurement is weak.

### Non-goals for v1

- Open-ended text tasks such as summarization, chat replies, and generated copy. This is planned for v2 and stated on the roadmap.
- Agents, tool use, or multi-turn conversations.
- Fine-tuning, distillation, or model training of any kind.
- A hosted service, dashboard, or team collaboration features.
- Broad academic benchmark coverage. Swapmeter evaluates *your* task, not MMLU.

---

## 3. Positioning

The LLM evaluation space is crowded: lm-evaluation-harness, promptfoo, DeepEval, Ragas, Inspect AI, Braintrust, LangSmith. Swapmeter is differentiated on two axes.

**Cold start.** Every comparable tool begins by asking for a dataset the user does not have. Swapmeter generates a synthetic input set from the prompt itself, so the first run needs nothing but a config file. Reducing time-to-first-result is the primary adoption lever.

**The review loop.** Because the baseline is the incumbent model's own output, the raw metric is *agreement*, not *correctness*. Rather than hiding this, Swapmeter surfaces the disagreements and lets the user adjudicate them. Reviewing a dozen disagreements takes about five minutes and produces a small labeled golden set the user never had to sit down and build. No other tool in this category converts its own weakness into the on-ramp for real data.

Local-first operation is a secondary differentiator against the SaaS options. Data never leaves the machine except to reach the model providers the user is already calling.

---

## 4. User journey

1. **Init.** `swapmeter init` scaffolds a config file. `swapmeter init --example` scaffolds a complete working config for a realistic task, runnable immediately with no edits.

2. **First run, no data.** `swapmeter run` generates a synthetic input set, runs the baseline model, then runs each candidate. A terminal table shows agreement rate, cost per 1,000 calls, p50 latency, and invalid output rate per model. `run` also writes the HTML report and prints its path, so a first-time user reaches the visual output without knowing a second command exists.

3. **Inspect.** The HTML report contains headline numbers and a side-by-side list of every disagreement. `swapmeter report` re-renders it from cached results without re-running anything, which is what you use after editing judgments or when you want a fresh shareable copy.

4. **Review.** `swapmeter review` starts a local web UI that walks through disagreements one at a time. For each, the user marks: baseline was right, candidate was right, or both acceptable. Judgments are written to a judgments file beside the config.

5. **Re-run.** `swapmeter run` scores against saved judgments where they exist, falling back to the baseline output elsewhere. Cached responses mean this costs nothing. Previously judged rows stay judged.

6. **Graduate to real data.** Changing `data.source` from `synthetic` to `file` and supplying a CSV or JSONL path swaps in real traffic. Nothing else in the config changes.

---

## 5. The config file

The config is the primary user interface. It is YAML, validated on load.

### Classification example

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

### Extraction example

Only the output and scoring blocks differ:

```yaml
  output:
    type: json
    schema:
      customer_name: string
      amount: number
      due_date: string

scoring:
  comparator: json_fields
```

### Real data

```yaml
data:
  source: file
  path: ./tickets.csv
```

### Field notes

- `task.prompt` uses Jinja-style `{{ variable }}` placeholders matching `task.inputs[].name`.
- `task.inputs[].description` does double duty. It drives synthetic input generation and documents the task for future readers.
- `output.type` is `enum` or `json`. These two cover classification, routing, tagging, and extraction, which is the full v1 scope.
- `scoring.comparator` is `exact`, `normalized_text`, or `json_fields`.
- `thresholds.min_agreement` causes `run` to exit non-zero when a candidate falls below it. This is the hook for CI usage and costs almost nothing to add now.
- Model identifiers use LiteLLM's `provider/model` convention.

---

## 6. Architecture

Nine modules, each with a single responsibility and testable in isolation.

### `config`

Loads and validates YAML into typed Pydantic models. Owns the config schema. Produces specific, actionable error messages, since a malformed config is the most common first-run failure.

**Depends on:** nothing internal.

### `providers`

Defines the `Provider` interface:

```
complete(model: str, messages: list, output_schema: Schema | None) -> Completion
```

A `Completion` carries parsed output, raw text, prompt and completion token counts, cost, latency, and an optional error. `LiteLLMProvider` is the sole v1 implementation. No other module imports LiteLLM, so it can be replaced without touching anything else.

**Depends on:** nothing internal.

### `inputs`

Produces input rows. Two implementations behind one interface:

- `SyntheticSource` generates rows using an LLM, driven by the prompt and the `inputs[].description` fields.
- `FileSource` reads CSV or JSONL, mapping columns to input variable names.

**Depends on:** `providers` (synthetic only), `config`.

### `cache`

Content-addressed store keyed by model identifier, prompt hash, and input hash. Backed by SQLite in a local `.swapmeter/` directory.

This module is load-bearing rather than an optimization. The review loop depends on re-running being free. If step 5 of the user journey re-bills the user, the journey collapses. It also makes tests deterministic and demos instant.

**Depends on:** nothing internal.

### `runner`

Orchestrates baseline and candidate execution. Owns concurrency limits, retry with backoff, rate-limit handling, and partial-failure recording. Reads through `cache` before calling `providers`. Emits a flat list of result records.

**Depends on:** `providers`, `cache`, `config`.

### `scoring`

Compares a candidate output against a reference output. Three pure-function comparators:

- `exact`: strict equality after type normalization.
- `normalized_text`: equality after whitespace and case normalization.
- `json_fields`: per-field comparison producing partial credit and a structured diff identifying which fields drifted.

Returns a verdict plus a diff. Being pure functions, these carry the densest test coverage in the project.

**Depends on:** nothing internal.

### `judgments`

Reads and writes user verdicts, keyed by input hash. Each entry holds an accepted-output set and a rejected-output set for that row. Owns one important rule: **where a judgment exists for a row, it overrides the baseline output as the scoring reference for every model.** Stored as JSON beside the config so it can be committed to the user's repo. Full semantics are in section 7.

**Depends on:** nothing internal.

### `report`

One Jinja template rendered two ways: a self-contained HTML file with inlined CSS and JS for sharing, and a terminal summary via `rich`.

**Depends on:** `scoring`, `judgments`.

### `cli`

Typer-based. Commands: `init`, `run`, `review`, `report`. Thin dispatch with no business logic.

**Depends on:** all of the above.

### Review server

`review` starts a local HTTP server that serves the report template in interactive mode and accepts judgment submissions via POST, writing them through `judgments`. A static HTML file cannot persist to disk, so the interactive path needs a process behind it. `report` remains a static shareable artifact.

This is the largest single chunk of v1 work. It is included deliberately because clicking through disagreements and watching the score change is the demo that communicates what the tool does.

### Data flow

```
config -> inputs -> runner (baseline + candidates, through cache)
       -> results -> scoring (judgments override baseline) -> report
```

---

## 7. Scoring semantics

Understanding what the number means is central to the product's credibility.

**Default reference.** For a given input row, the reference output is the baseline model's output for that row.

**Judgment override.** A judgment is stored per input row, not per candidate. It records the set of outputs the user has accepted as correct for that row, plus the set explicitly rejected. The review UI presents each disagreement as a choice, and that choice updates those sets:

- *Baseline was right*: the baseline's output joins the accepted set, the candidate's output joins the rejected set.
- *Candidate was right*: the candidate's output joins the accepted set, the baseline's output joins the rejected set.
- *Both acceptable*: both outputs join the accepted set.

Once a row has any judgment, scoring for that row uses the accepted set for **every** model, including the baseline. Any model producing an output in the accepted set is a match. Any model producing an output in the rejected set is a mismatch. An output in neither set is unjudged and surfaces as a new disagreement for review.

Keying judgments to the row rather than to a candidate means the work of judging is reused across every model, and across future runs that add new candidates. It also lets the report show the baseline model's own accuracy, which is often the most surprising number in the run.

**Naming discipline.** The metric is called **agreement**, never accuracy, anywhere in the UI, the report, or the docs, until judgments exist for the rows in question. Once a run's rows are fully judged, the report may present it as accuracy against the user's own labels. Blurring this line is the fastest way to lose technical credibility.

**Invalid output.** Output that fails to parse against the declared schema, or an enum value outside the declared set, is recorded as `invalid` rather than as a mismatch. It gets its own reported rate. Cheap models fail structured output more often than frontier models, and this failure rate is one of the most decision-relevant numbers the tool produces.

---

## 8. Error handling

**Fail fast before spending.** On `run`, before any billable call: validate the config, confirm every referenced model resolves, and verify required API keys are present. Discovering a missing key partway through a paid run is an infuriating first experience.

**Cost preview.** `run` estimates total cost from token counts and the model list, prints it, and prompts for confirmation above a configurable threshold. A tool about saving money must not surprise the user with a bill.

**Per-cell failure isolation.** A timeout or rate limit on one input-model pair retries with backoff. If it still fails, that cell is recorded as an error and the run continues. The report shows coverage per model, for example "gemini-2.5-flash: 48/50 rows completed." A single flaky provider must never destroy a paid run.

**Malformed output is data, not an exception.** See section 7.

**Resumability.** Interrupting a run and re-running resumes from the cache. No separate checkpoint machinery is needed.

---

## 9. Testing strategy

The bar: **the full default suite runs offline, in seconds, with no API keys.** This is what makes outside contribution possible.

- **`FakeProvider`** implements the `Provider` interface with scripted responses, simulated failures, controllable latency, and synthetic token counts. The large majority of tests use it and never touch the network.
- **Comparator unit tests** are the densest in the project. Coverage includes whitespace, casing, key ordering, nulls, missing fields, extra fields, and type coercion. A silent bug here produces wrong recommendations, which is the worst failure mode this tool has.
- **Runner tests** use `FakeProvider` to cover concurrency, retry behavior, partial failure, and cache hit paths.
- **Judgment resolution tests** verify the override rules in section 7 exhaustively.
- **Recorded provider fixtures** guard the LiteLLM wrapper against upstream changes. Checked in, marked as a separate suite, excluded from the default run.
- **Report golden-file tests** render from a fixed result set and diff against committed snapshots.
- **One end-to-end test** covers config, fake run, report, judgment, re-run, asserting the score changes as expected. This exercises the entire product in a single test.

---

## 10. Technology choices

| Concern | Choice | Reasoning |
|---|---|---|
| Language | Python 3.11+ | Target users' LLM pipelines are overwhelmingly Python. The eval ecosystem and its contributors live here. |
| Distribution | `uvx swapmeter` / `pip install swapmeter` | Near-zero friction first run. |
| Provider layer | LiteLLM, wrapped | Breadth across 100+ providers plus maintained pricing metadata, which is tedious and error-prone to maintain by hand. Wrapped so it is replaceable. |
| CLI | Typer | Standard, good help output, minimal boilerplate. |
| Config validation | Pydantic | Typed models with useful validation errors. |
| Terminal output | rich | Tables and progress display. |
| Templating | Jinja2 | One template serves both HTML render modes. |
| Cache | SQLite | Stdlib, single file, no service to run. |
| Review UI | Stdlib HTTP server plus vanilla JS | Avoids a frontend build step. The repo stays pip-installable with no Node toolchain. |

**A single implementation, deliberately.** There will be no parallel TypeScript port. Two implementations means two test suites, two release processes, inevitable drift, and split attention from a solo maintainer. Because the interface is a config file plus a CLI, a TypeScript shop uses Swapmeter without writing Python. If demand for a native Node client appears later, a thin wrapper over the same config format is a small amount of work.

**Known risk:** LiteLLM is a large dependency with active release churn, and its structured-output handling is not uniform across providers. Edge cases are expected. The `Provider` wrapper is what makes them survivable and contained.

---

## 11. Repository and launch

- **Name:** `swapmeter`. Verified on 2026-08-03: no PyPI package by that name, and zero GitHub repositories matching it. The namespace is empty on both.
- **License:** Apache 2.0. The patent grant removes a common enterprise legal objection at no cost.
- **Visibility:** public, under `ttague222`.

An earlier working name, `understudy`, was rejected because PyPI `understudy` is an active package for trace-based evaluation of agentic systems, which is close enough to this project's space to cause real confusion. `downshift` and `standin` were rejected for heavy collisions with a popular React library and a .NET security toolkit respectively.

**README structure**, in order:

1. One sentence stating the question the tool answers.
2. A terminal screenshot showing a real cost saving. The table, not a logo.
3. A three-line quickstart that runs with no dataset.
4. The honest caveat about agreement versus correctness, stated plainly, followed by how the review loop resolves it.
5. Roadmap, naming open-ended text evaluation as the next release.

**`init --example` is the highest-leverage adoption feature in the project.** A curious visitor gets a complete run in under a minute without writing anything.

**Launch framing:** lead with a finding, not the tool. "I ran my production classification prompt against six models. Here is what matched and what it would save." The repository is the footnote. This framing is also honest, since building the tool is how that finding gets produced.

**Channels:** Hacker News, r/LocalLLaMA, and AI engineering communities on Twitter and LinkedIn.

---

## 12. Roadmap beyond v1

Stated publicly in the README so the project reads as directed rather than abandoned.

- **v2: open-ended text tasks.** Summarization, chat replies, generated copy. Requires an LLM judge, and the judge's calibration becomes the credibility question. By then, real users will have shaped the requirements rather than guesswork.
- **v2: CI integration.** A GitHub Action gating on `thresholds`. The `thresholds` config and non-zero exit code already exist in v1 for this purpose.
- **Later: JSON result artifact** for programmatic consumption.
- **Later: redaction helpers** to make exporting real production data an easier internal conversation.
