# Swapmeter v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Python CLI that takes a production prompt, runs it across a baseline model and several candidate models, and reports which candidates match the baseline's output, at what cost, with a review loop that turns disagreements into a reusable labeled set.

**Architecture:** A pipeline of small, independently testable modules. `config` parses YAML into typed models. `inputs` produces rows either synthetically or from a file. `runner` executes every (row, model) pair through a `cache` and a `Provider` abstraction that wraps LiteLLM. `outputs` parses raw model text against the declared schema. `scoring` compares candidate output against a reference using pure comparator functions. `judgments` persists user verdicts that override the baseline as the reference. `report` builds a summary and renders it to terminal and HTML. `review` serves an interactive HTML page that writes judgments back.

**Two refinements to the design doc**, both deliberate:
1. The spec put output parsing inside `providers`. This plan splits it into a separate `outputs` module so `Provider` implementations only deal with transport and the parsing logic is pure and heavily testable.
2. The spec described `report` as one module. This plan makes it a package with `summary.py` (builds presentation-shaped data), `terminal.py`, and `html.py`, because building the summary is real logic that deserves its own tests.

**Tech Stack:** Python 3.11+, LiteLLM, Typer, Pydantic v2, Jinja2, rich, PyYAML, SQLite (stdlib), pytest.

---

## Phase Boundaries

The plan is ordered so that **Phase 6 is a shippable tool**. If the work needs to be staged, cut after Task 16: you have a working CLI that answers the core question in the terminal. Phases 7 and 8 add the HTML report and the interactive review UI, which is what makes the demo compelling.

| Phase | Tasks | Outcome |
|---|---|---|
| 0 | 1 | Repo scaffolding, tests run |
| 1 | 2-3 | Output parsing and config |
| 2 | 4-7 | Provider abstraction, fake, cache, LiteLLM |
| 3 | 8-9 | Input sources |
| 4 | 10-12 | Scoring and judgments |
| 5 | 13 | Runner |
| 6 | 14-16 | Terminal report and `init` / `run` **(shippable)** |
| 7 | 17-18 | HTML report and `report` |
| 8 | 19-21 | Review server and `review` |
| 9 | 22-23 | End-to-end test and README |

---

## File Structure

```
swapmeter/
  pyproject.toml
  README.md
  LICENSE
  .gitignore
  src/swapmeter/
    __init__.py
    outputs.py                  # OutputSpec types + parse_output()
    config.py                   # Pydantic models + load_config()
    cache.py                    # SQLite content-addressed response cache
    inputs.py                   # InputRow, render_prompt, FileSource, SyntheticSource
    scoring.py                  # Verdict, Reference, comparators, score_row()
    judgments.py                # Judgments store, canonical(), reference_for()
    runner.py                   # ResultCell, Runner
    review.py                   # Local HTTP server for the review UI
    cli.py                      # Typer app: init, run, report, review
    providers/
      __init__.py
      base.py                   # Completion dataclass, Provider protocol
      fake.py                   # FakeProvider for tests and demos
      litellm_provider.py       # LiteLLMProvider
    report/
      __init__.py
      summary.py                # ModelSummary, Disagreement, RunSummary, build_summary()
      terminal.py               # render_terminal()
      html.py                   # render_html()
      templates/
        report.html.j2          # One template, static and interactive modes
    examples/
      support_router.yaml       # Shipped by `init --example`
  tests/
    test_outputs.py
    test_config.py
    test_cache.py
    test_inputs.py
    test_scoring.py
    test_judgments.py
    test_runner.py
    test_summary.py
    test_terminal.py
    test_html.py
    test_review.py
    test_cli.py
    test_end_to_end.py
    fixtures/
      __init__.py
```

**Responsibility boundaries.** `scoring` imports nothing from other Swapmeter modules, so comparators stay pure functions. `judgments` builds the `Reference` that `scoring` consumes, which is what keeps storage out of the scoring path. Only `providers/litellm_provider.py` imports `litellm`; if it needs replacing, nothing else changes.

---

## Phase 0: Scaffolding

### Task 1: Project skeleton

**Files:**
- Create: `pyproject.toml`
- Create: `.gitignore`
- Create: `LICENSE`
- Create: `src/swapmeter/__init__.py`
- Create: `tests/__init__.py`
- Test: `tests/test_smoke.py`

- [ ] **Step 1: Create `pyproject.toml`**

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "swapmeter"
version = "0.1.0"
description = "Find out whether a cheaper LLM can replace the model you run in production"
readme = "README.md"
requires-python = ">=3.11"
license = { text = "Apache-2.0" }
authors = [{ name = "Tom Tague" }]
keywords = ["llm", "evaluation", "benchmark", "cost", "openai", "anthropic"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: Apache Software License",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]
dependencies = [
    "litellm>=1.55.0",
    "typer>=0.12.0",
    "pydantic>=2.7.0",
    "pyyaml>=6.0",
    "jinja2>=3.1.0",
    "rich>=13.7.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=5.0.0",
    "ruff>=0.6.0",
]

[project.scripts]
swapmeter = "swapmeter.cli:app"

[project.urls]
Homepage = "https://github.com/ttague222/swapmeter"
Issues = "https://github.com/ttague222/swapmeter/issues"

[tool.hatch.build.targets.wheel]
packages = ["src/swapmeter"]

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "live: tests that make real provider API calls (excluded by default)",
]
addopts = "-m 'not live'"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]
```

- [ ] **Step 2: Create `.gitignore`**

```gitignore
__pycache__/
*.py[cod]
.venv/
venv/
dist/
build/
*.egg-info/
.pytest_cache/
.ruff_cache/
.coverage
htmlcov/
.swapmeter/
swapmeter-report.html
.env
.DS_Store
```

- [ ] **Step 3: Download the Apache 2.0 license text**

Run: `curl -sSL -o LICENSE https://www.apache.org/licenses/LICENSE-2.0.txt`
Expected: a `LICENSE` file of roughly 11 KB whose first line is `Apache License`.

Verify: `head -1 LICENSE`

- [ ] **Step 4: Create package init files**

`src/swapmeter/__init__.py`:

```python
"""Swapmeter: find out whether a cheaper LLM can replace the model you run in production."""

__version__ = "0.1.0"
```

`tests/__init__.py`: empty file.

- [ ] **Step 5: Write the smoke test**

`tests/test_smoke.py`:

```python
import swapmeter


def test_package_imports_and_has_version():
    assert swapmeter.__version__ == "0.1.0"
```

- [ ] **Step 6: Install and run the test**

Run:
```bash
pip install -e ".[dev]"
pytest tests/test_smoke.py -v
```
Expected: `1 passed`

- [ ] **Step 7: Commit**

```bash
git add pyproject.toml .gitignore LICENSE src/swapmeter/__init__.py tests/__init__.py tests/test_smoke.py
git commit -m "chore: scaffold swapmeter package"
```

---
