# IncidentZero

An autonomous SRE incident-response agent built on the Groq API. The agent
observes a simulated production incident, forms an explicit plan grounded in
real evidence, calls tools through a validated registry, requires human
approval for risky actions, dynamically re-plans when the world changes or an
action fails, and always terminates in a well-defined, traceable state
(`resolved`, `escalated`, `budget_exhausted`, or `aborted`) instead of
crashing or looping indefinitely.

**Student ID:** 25I-8037
**Default model:** `openai/gpt-oss-20b` (Groq API)

## Contents

- [Architecture](#architecture)
- [Setup](#setup)
- [Running the agent](#running-the-agent)
- [Running the tests](#running-the-tests)
- [Repository structure](#repository-structure)
- [Traces](#traces)
- [Configuration](#configuration)

## Architecture

| Module | Responsibility |
|---|---|
| `incidentzero/agent/state.py` | Explicit runtime state (plan, evidence, world version, retry/replan logs) |
| `incidentzero/agent/planner.py` | Creates and revises the agent's plan, grounded in real observations |
| `incidentzero/agent/policies.py` | Re-planning classification (`ReplanPolicy`), loop detection (`LoopGuard`), risk levels (`RiskPolicy`) |
| `incidentzero/agent/recovery.py` | Bounded, backoff retry for transient model errors (`RetryPolicy`) |
| `incidentzero/agent/controller.py` | Main orchestration loop: validation → approval → execution → classification |
| `incidentzero/agent/prompts.py` | System prompt for the controller loop |
| `incidentzero/model/groq_client.py` | Groq API adapter, with transient/permanent error classification |
| `incidentzero/cli.py` | Real-model run entry point |
| `demo_use_cases.py` | 10 offline worked examples using a scripted model client |
| `tests/student/` | Student-written test suite |

The agent's core control-flow guarantee: **every tool result, model response,
and control-flow event is explicitly classified and routed to a handler.** No
recoverable condition is allowed to reach an unhandled exception.

## Setup

Requires Python 3.11+.

```bash
# from the repo root
python -m venv env
# Windows
.\env\Scripts\activate
# macOS/Linux
source env/bin/activate

pip install -r requirements.txt
```

Set your Groq API key, either as an environment variable or in a `.env` file
in the repo root (never commit this file):

```
GROQ_API_KEY=your_key_here
```

Get a key at [console.groq.com](https://console.groq.com) → API Keys.

## Running the agent

Run a single scenario against the real Groq model:

```bash
python -m incidentzero.cli run --student-id YOUR_ID --scenario public-a
```

```
usage: incidentzero.cli run [-h] --student-id STUDENT_ID --scenario SCENARIO
                             [--model MODEL] [--env-file ENV_FILE]
                             [--trace-dir TRACE_DIR] [--trace-name TRACE_NAME]
                             [--risk-policy-path RISK_POLICY_PATH]
                             [--limits-path LIMITS_PATH]
                             [--approval {console,auto}] [--print-seed]
```

- `--approval console` (default) prompts you interactively for every
  high/critical-risk action — this is the human-in-the-loop approval gate.
  `--approval auto` approves everything unattended, for convenience trace
  generation.
- Each run writes a JSONL trace to `traces/<scenario>-<student-id>.jsonl`,
  containing every decision point: `plan_created`, `plan_revised`,
  `approval_request`/`approval_result`, `retry_event`, `loop_detected`, and
  `terminal_result`.

Run the offline worked examples (no Groq quota consumed):

```python
from demo_use_cases import verify_registration_scenarios
verify_registration_scenarios()
```

## Running the tests

```bash
pytest -q tests/
```

All tests run against an offline `ScriptedModelClient`, so they're
deterministic and require no API key.

## Repository structure

```
incidentzero_starter/
├── incidentzero/
│   ├── agent/
│   │   ├── state.py
│   │   ├── planner.py
│   │   ├── policies.py
│   │   ├── recovery.py
│   │   ├── controller.py
│   │   └── prompts.py
│   ├── model/
│   │   ├── groq_client.py
│   │   ├── base.py
│   │   └── errors.py
│   ├── tools/
│   │   └── registry.py
│   ├── environment/
│   │   ├── engine.py
│   │   └── scenario_factory.py
│   ├── approval/
│   │   └── gateway.py
│   ├── telemetry/
│   │   ├── budget.py
│   │   └── trace.py
│   ├── domain/
│   │   └── models.py
│   └── cli.py
├── configs/
│   ├── risk_policy.json
│   └── limits.json
├── tests/
│   ├── public/
│   └── student/
├── traces/
├── demo_use_cases.py
├── requirements.txt
├── REPORT.md
├── AI_USAGE.md
└── README.md
```

## Traces

Three required public-scenario traces (live Groq runs) are included under
`traces/`:

| Scenario | Status | LLM calls | Tool calls |
|---|---|---|---|
| public-a | resolved | 8 | 10 |
| public-b | resolved | 9 | 10 |
| public-c | aborted (wall-clock budget, safe termination) | 11 | 10 |

See `REPORT.md` for full details on the run results and the debugging
process that led to the current implementation.

## Configuration

- `configs/risk_policy.json` — maps each tool to a risk level
  (`low`/`medium`/`high`/`critical`). Unknown tools default to `critical`
  (fail-closed).
- `configs/limits.json` — resource budgets: max LLM requests, max tool calls,
  max consecutive retries, max repeated actions, max wall-clock runtime
  (seconds).

Do not commit `.env` or any file containing `GROQ_API_KEY`.
