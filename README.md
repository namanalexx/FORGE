# FORGE — Feedback-Oriented Refinement & Generation Engine

A multi-agent system (CrewAI + Groq) that tests whether an autonomous generate → test →
review → retry loop produces more reliable code than single-shot LLM generation — on two
separate axes: **functional correctness** and **production readiness**.

## The question this answers

Any LLM can generate code once. The interesting question isn't "can it write code" — it's
**does wrapping generation in autonomous verification measurably improve reliability**,
holding the base model constant?

FORGE tests a sharper version of that: **can code pass its own tests and still be
unshippable?** A function can satisfy every test case and still crash on malformed input,
swallow exceptions silently, or hardcode a value that should be a parameter. Single-shot
generation has no mechanism to catch that. FORGE does.

## Architecture

Five CrewAI agents, each with a distinct role:

| Agent | Job |
|---|---|
| **Planner** | Turns a raw task description into a functional spec + explicit business constraints |
| **Generator** | Writes the implementation from the spec (and from prior-attempt feedback on retries) |
| **Test Engineer** | Writes pytest tests **from the spec only**, never seeing the implementation — so tests can't be unconsciously fitted to whatever the Generator produced |
| **Correctness Reviewer** | Runs the generated code against the tests in a sandboxed subprocess, interprets pass/fail |
| **Production-Readiness Auditor** | Checks the code against the task's business constraints — independent of whether tests pass — using both static checks (bare `except`, `eval`/`exec`, hardcoded secrets, missing docstrings) and an LLM review of task-specific constraints |

An attempt is approved only when **both** reviewers pass it. If either rejects, their
combined feedback is passed back to the Generator for another attempt (bounded retry loop).

**Baseline control:** for a fair comparison, a single-shot generation (no loop, no
feedback) is evaluated against the *exact same test code* FORGE uses — isolating the effect
of the agentic loop itself rather than differences in what's being tested for.

## How it works — input, pipeline, output

**Input:** a task bank of real-world business-utility functions (not algorithmic puzzles)
— e.g. "compute monthly revenue from transaction records, must not crash on malformed
rows" — each paired with explicit business constraints a reviewer would actually care about.

**Pipeline, per task:**
1. Planner expands the task into a spec + constraints
2. Test Engineer writes tests from that spec alone
3. Generator writes an implementation
4. Sandbox harness runs the tests via subprocess + pytest (hard timeout, no network)
5. Correctness Reviewer and Readiness Auditor each independently judge the result
6. If either fails, structured feedback goes back to the Generator; repeat up to a retry cap
7. Baseline: the same spec is sent to the Generator once, no loop, evaluated against the same tests

**Output:** for every task, a pass/fail on both axes for both baseline and FORGE, the
number of attempts FORGE needed, and the final code. Aggregated into a comparison table and
a bar chart: baseline vs. FORGE, functional correctness vs. production readiness.

## Evaluation methodology

Controlled, not anecdotal: same base model, same tasks, same test code, only the presence
of the agentic loop differs between baseline and FORGE. This isolates what the loop itself
contributes, rather than attributing differences to the model being "smarter."

## Results (sample run, 6 tasks, `openai/gpt-oss-20b` via Groq free tier)

| Metric | Baseline (single-shot) | FORGE (agentic loop) |
|---|---|---|
| Functional correctness | 0.0% | 16.7% |
| Production readiness | 16.7% | **100%** |
| Avg. attempts to resolution | — | 1.83 |

**Readiness is the headline finding:** single-shot generation satisfied the business-
constraints rubric on only 1 of 6 tasks; FORGE's review-and-retry loop caught and fixed
every readiness gap it encountered, reaching 100%.

**Correctness was a harder result, reported honestly rather than inflated:** both baseline
and FORGE struggled functionally. This sample used a small free-tier model, a bounded
2-attempt retry budget (to stay within rate limits), and tests written blind from spec —
three factors that plausibly suppress correctness independent of whether the *architecture*
works. The readiness result demonstrates the architecture works; correctness is a capability
ceiling of this specific model/budget, not evidence against the approach.

## Engineering notes

A few non-obvious things this project surfaces, in case they're useful to anyone else
building on CrewAI + Groq:

- CrewAI's synchronous `kickoff()` fails inside Colab's already-running event loop —
  `kickoff_async()` is required there.
- CrewAI tags messages with a `cache_breakpoint` key meant for Anthropic's prompt caching,
  which leaks into non-Anthropic providers routed through LiteLLM and gets rejected by
  Groq's API (open upstream issue). Stripped via a small `litellm.completion` patch.
- `gpt-oss` models are reasoning models — under a tight token budget they can spend the
  whole budget on hidden reasoning and return empty final content. Fixed with
  `reasoning_effort="low"` and a generous `max_tokens`.
- Groq's free tier enforces both per-minute (TPM) and per-day (TPD) token caps, per model.
  The retry layer parses Groq's own wait-time hint from the error message and treats daily-
  cap errors differently from per-minute ones (fail fast with a clear message rather than
  retrying for hours).

## Setup

1. Open `FORGE.ipynb` in Google Colab
2. Get a free API key at [console.groq.com](https://console.groq.com/keys)
3. Paste it into the setup cell
4. Run top to bottom — `USE_MOCK = True` first to dry-run the pipeline with zero API calls, then `USE_MOCK = False` for a real run

## Limitations

- Single base model tested per run — no cross-model comparison yet
- Task sample size (6–12) is small; a larger, more diverse bank would strengthen the result
- Retry budget capped low to respect free-tier rate limits
- Readiness rubric is a fixed checklist, not an exhaustive production-readiness standard

Developer: Naman Alex Xavier (namanalex@gmail.com)
