Incident-Zero : Making Agent Robust to uncertain conditions enabling agent to respond effectively.
 


Assignment 1
Submitted by
Ayesha Shabbir(25I-8037)







DEPARTMENT OF COMPUTING
NATIONAL UNIVERSITY OF COMPUTER & EMERGING SCIENCES
September, 2026

Overview:
In this assignment, we tries to made Incident Zero autonomous and robust incident-response agent  by modifying its various functions to handle failure situations effectively. Our agent observes the simulated environment, made an evidence based plan , calls predefined tools and asks for human approval on risky actions, replans if environment change and rather than failing in changing environment it nicely ends with statuses of Resolved, Escalated, budget_exhausted and aborted.
The core objective is to make agent robust and capable to function effectively in every scenario with no recoverable condition should reach dead end.
Architecture:
Agent State (state.py):  
It is the run time memory of what the agent is currently doing. It is separated from LLM conversation history. It stores current plan, evidence numbers supporting that plan, latest world observation describing environment state, retry calls and its results and terminal end status of agent.
Planner (planner.py):
It produces the initial plan to solve the problem encountered by agent. If initial plan is failed then it trigger it back to replan with trigger causing failure of initial plan and also store compact state summary and preserves already completed action to prevent their execution again and again saving time and resources.
Replan Policy (policies.py)
The result of very tool call is classified into various statuses. Resolved means the problem is solved efficiently  by agent. Escalated means action needs human approval for its proper completion. Re-observe states that world version has changed and need replan. Reject means there is a local error which is fixed by agent in next action or replan only, if current plan fails at any point.
Loop Guard (policies.py)
It is like a finger print of tool recording how many time a specific tool call returns the same failure and stored in form of counter. But if that action of tool call bring some new information like changing environment then reset tool counter to zero. So it prevents the tool from getting stuck in infinite loop.

Risk Policy (policies.py)
It loads the risk policy from config to determine the risk level before execution of action by each tool. If a user added new tool and forget to add in risk policy then set its risk level default to severe and asks for human approval before execution of that action.
Retry policy (recovery.py)
It is helpful in solving permenant and transient error. If the tool action is returning failure again and again then it sets maximum limit to save resources. But if there is a transient error then it assign timeout instead of sending many requests in that time span again and again.
Agent Controller (controller.py)
It is the main orchestrator controlling agent behavior. It validates schema of every tool before calling that tool, asking for human approval for sensitive actions and set wall clock, LLM and budget tool call to prevent agent from getting stuck into infinite loop and end every model call site into degrade or escalate behavior.
Cli.py:
Command line interface to start and initialize controller then creates different files, run agents and returns their responses.
Real world debugging and hardening
Offline testing  went through without a hitch, but running the agent against a real Groq API revealed four unique problems that couldn't be discovered through offline testing, as they rely on the behavior of the provider. All 4 were identified and corrected:
1.	The JSON for the tool call was malformed (tool_use_failed, HTTP 400). The model occasionally generated syntactically invalid JSON for a tool call's arguments. At the API level Groq blocks this prior to any tool call hitting the controller. The original error translated logic was designed to translate this into a permanent error (400 is not in the transient-status set), killing off the entire run if one bad generation occurred. Fix: This pattern which is identified by the provider is now considered transient and it will be passed by the existing bounded-retry path instead of killing the run.
2.	Not reliably detecting rate limiting (HTTP 429). 429 was already listed as a transient status code, but .status_code attribute of the SDK exception was not consistent at runtime, so real 429s were falling through to PermanentModelError and crashing the run. Fix: added a backup that will parse the guaranteed "Error code: NNN" string that Groq always outputs in the error. Also retrieved the wait time suggested by the server (e.g. "try again in 2.775s"), and set a minimum value for the retry delay, as a fixed exponential backoff could be shorter than the actual window of the rate limit and would thus be useless in wasting each attempt.
3.	The 3rd exception is the unhandled KeyError on a malformed structured response, which has a fallback path (triggers a specific Groq constrained-decoding failure) that requests plain JSON with no schema enforcement. Planner.create() and Planner.revise() did not check the shape of what they returned before indexing into it (raw["hypothesis"]), which caused an incompletely-formed response to crash the run on the KeyError. Fix: added explicit shape validation in the model which raises TransientModelError on a malformed payload, this is already handled gracefully by the controller, so it can be retried or escalated.
4.	Uncaught retry exhaustion. RetryPolicy re-raises TransientModelError unchanged when it has used up its budget of attempts (by design, not silently changing a transient failure into a permanent failure). But no more than several call sites (_model_decide, bootstrap plan creation and _replan) were catching that exception, so even a persistent failure had the potential to crash uncaught after fix #1 – #3. Fix: Each call site now fails gracefully: either by providing the model with one corrective turn (where it can) or by escalating the failure with what evidence currently exists that it failed (where there is no plan to fall back to).
Evidence of correctness
All 10 worked use cases in demo_use_cases.py are executing successfully in presence of fake model and completed all tasks successfully. Three public scenarios (public-a, public-b, public-c) are implemented successfully by agent by using groq model.
This submission includes traces/public-a-25I-8037.jsonl, traces/public-b-25I-8037.jsonl, and traces/public-c-25I-8037.jsonl.
The two public-a and public-b handled the incident with verified verify_recovery evidence supporting close_incident, following the complete observe → plan → act → verify → close pipeline with a real model. Not a crash, not a hang, public-c terminated via the wall-clock safety check at 125.09s. The agent had gathered 10 pieces of evidence, but not yet connected to a terminal action, and had progressed through three attempts of a rate-limited turn. By the end of the time, the agent had made real, continuous progress (10 pieces of evidence collected, world_version advanced to 4, three attempts at a rate-limited turn). This is the desired behavior of R12's graceful termination requirement: if a run cannot finish safely within the budget it gracefully terminates with a clear AgentOutcome without crashing or running unbounded.
No unhandled exception or a status="failed" crash was reported in any of the three public scenarios tested on any of the three live runs, or during any of the offline tests, after the fixes.
Scenario	Status	LLM calls	Tool calls	Evidence items	Wall-clock
public-a	resolved	8	10	8	59.2s
public-b	resolved	9	10	9	89.75s
public-c	aborted (wall-clock budget)	11	10	10	125.09s

Design decisions and trade-offs:
Resolved, escalated and budget_exhausted, aborted are all considered as valid "terminal" states, aligning with the assignment's own "close and clean" philosophy: an agent that correctly determines that it cannot safely complete and halts is not failing, but is acting "correctly". There are really only two types of exceptions: unhandled and handled.There are only two types of exceptions: unhandled and handled.
Re-planning is rooted, rather than generic. Planner.revise() always receives the specific trigger event and a summary of previous learning; and is explicitly instructed not to resubmit an unchanged plan, with the idea being that if an agent is told to "try again" he or she will not repeat the same mistake.
Malformed model output is not considered to be a configuration failure, but a stochastic failure. Single bad generation (malformed JSON, incomplete structured response) is retried, not considered fatal, as it is not a sign of broken setup.
Approval is not implied. For the Live CLI runs, this is a real console prompt and NOT an auto-approve stub (only for offline runs).


 
