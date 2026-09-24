# AI Usage Declaration

AI assistance (Claude) was used during this assignment for the following purposes:

1. **Coding** — drafting and refining implementations of the agent modules under
   `incidentzero/agent/` (policies.py, recovery.py, planner.py, controller.py,
   prompts.py) and the CLI entry point (`incidentzero/cli.py`), based on the
   requirements in `assignment_a.docx`.

2. **Error resolving** — diagnosing and fixing issues encountered while running
   the agent against the real Groq API, including:
   - A malformed tool-call JSON error (`tool_use_failed`) being misclassified as
     a permanent/fatal error instead of a retryable one.
   - A rate-limit (`429`) error not being correctly detected as transient.
   - An unhandled `KeyError` when a model response was missing expected fields.
   - Environment/path issues (virtual environment selection, missing packages,
     duplicate project folders).

3. **Refinement** — hardening the controller and model-client code so that
   recoverable failures (stale world state, denied approval, malformed model
   output, rate limits) are retried or re-planned instead of crashing the run,
   and adding a top-level safety net so no failure mode can surface as an
   unhandled exception.

All final code was reviewed, tested (offline via `ScriptedModelClient` and
against the real Groq API), and understood before submission. No content was
submitted without verification.