# OBSERVATIONS.md

## Overview

This document records what was tested, what was found, and what changed across
multiple iterations of the system prompt and harness configuration for the
customer support chatbot (bug reports, FAQ platform questions, and
out-of-scope hand-offs).

Four Bedrock Evaluation runs were performed over the course of development,
each with the Nova Pro evaluator model and the Correctness metric. Manual
testing via `chat.py`, DynamoDB scans, and CloudWatch review were used
alongside automated evaluation to catch issues the single-turn eval prompts
alone would not surface.

## Evaluation Run History

| Run | Test count | Correctness (avg) | Notes |
|-----|-----------|--------------------|-------|
| Run 1 (`bugevaluator`) | 3 | 1.00 | Original 3-route smoke test. Passed, but manual review found `<thinking>` tags leaking into customer-facing output — not caught by this metric. |
| Run 2 (`bugevaluator2`) | 3 | 0.83 | Same 3 tests, after a field-ordering fix. One bug-report test scored 0.5 — the assistant asked to re-confirm the description (already given) while claiming steps/environment were already provided (they were not). |
| Run 4 (`support-chatbot-eval-run-4`) | 10 | **0.90** | Expanded test suite covering all three routes plus two edge cases (an ambiguous message and a prompt-injection attempt), after: (a) disabling harness memory, (b) adding placeholder-value guards for the bug-report tool call, (c) strengthening the no-thinking-tags output format rule, (d) adding explicit ambiguous-message handling, and (e) requiring the exact hand-off sentence for all route-C and hand-off cases. 9 of 10 prompts scored in the 0.9 range; 1 prompt scored 0 (see below). |

*(Run 3, an intermediate CLI-created job with the same 10 tests before the ambiguous-message and hand-off wording fixes, was superseded by Run 4 and is not included as separate evidence.)*

## Bugs Found, Root-Caused, and Fixed

### 1. Cross-session memory bleed (harness-level, not a prompt bug)

**Symptom:** In manual `chat.py` testing, a conversation would sometimes reference
details, or even resurrect and act on, an entirely separate customer request
from a previous session — including fabricating a bug ticket from an
unrelated FAQ exchange, and once quoting several real, exact ticket IDs from
earlier in the DynamoDB table inside a fresh conversation that had never
mentioned them.

**Investigation:** Ruled out `chat.py` and `generate-eval-dataset.py` as the
cause — both generate a fresh, random `runtimeSessionId` (via `uuid.uuid4()`)
on every run, and `agentcore_config.json` stores no session identifier.

**Root cause:** `create_harness.py` does not pass a `memory` parameter when
creating or updating the harness, so AgentCore defaults the harness to
**managed memory enabled**, which persists context beyond a single
`runtimeSessionId`.

**Fix:** Added `memory={"disabled": {}}` to both the `create_harness(...)`
and `update_harness(...)` calls in `create_harness.py`. Verified via:

get_harness(...)['harness']['memory'] -> {"disabled": {}}

(previously returned `{"managedMemoryConfiguration": {"arn": "..."}}`).
Confirmed the fix by rebuilding the harness from scratch and re-running the
same manual conversation that had previously triggered the bleed — no
recurrence.

**Disclosure:** This is a modification to a provided script (`create_harness.py`),
not just the system prompt, and is disclosed here as required.

### 2. Tool called with placeholder values instead of asking for missing fields

**Symptom:** For the message "my wishlist is not loading" (which only
contains a description), the assistant's own `<thinking>` reasoning
correctly identified that `stepsToReproduce` and `environment` were missing —
but it then called `create_bug_report` anyway, writing the literal string
`"not specified"` into both fields. Confirmed directly in DynamoDB:

{
"description": "my whishlist is not loading",
"stepsToReproduce": "not specified",
"environment": "not specified"
}

This is a direct violation of the prompt's own stated rule ("never call the
tool with an empty or invented value").

**Fix:** Added an explicit pre-flight self-check instruction before the
tool-calling rule ("BEFORE calling create_bug_report, check each of the three
values you are about to send individually..."), added a matching validity
gate for `stepsToReproduce` (previously only `environment` had one), and
added a second worked example using this exact failure case. Re-tested the
same message after the fix — the assistant correctly asked for the missing
field instead of calling the tool. No further placeholder-value tickets
appeared in DynamoDB after this fix.

### 3. `<thinking>` tags leaking into customer-facing responses

**Symptom:** Across every evaluation run and several manual conversations,
the model's internal reasoning (wrapped in `<thinking>...</thinking>`) was
included in the visible response, rather than being stripped before the
reply reached the customer.

**Fix attempts:** A single-line instruction ("never include thinking tags")
was added first but was not consistently followed. It was replaced with a
dedicated, first-position "OUTPUT FORMAT" section in the prompt with more
explicit, repeated constraints on what the visible output may and may not
contain.

### 4. Ambiguous messages routed straight to hand-off instead of asking a clarifying question

**Symptom:** The test prompt "My order is wrong" — which is genuinely
ambiguous between a bug report (a technical fault) and a platform question
(a fulfillment issue) — was handed off to the human support line rather than
prompting a short clarifying question, scoring 0 against the reference
response. This is the one prompt that still scored 0 in the final (Run 4)
evaluation.

**Fix:** Added an explicit "genuinely ambiguous messages" rule with a
concrete worked example and a sample clarifying question, directly targeting
this case. This fix was applied for Run 4; the remaining 0 score suggests
the model still defaults to hand-off under some ambiguity rather than
asking — noted as a residual limitation below.

### 5. "Other request" responses deviated from the exact hand-off wording

**Symptom:** For out-of-scope requests (e.g. "Can you help me write a Python
program?", "I want to speak to a manager about a complaint"), the assistant
sometimes offered alternative resources or a softened, reworded version of
the hand-off message instead of the exact required wording, scoring 0.5 in
two of the ten test cases.

**Fix:** Rewrote the "OTHER BEHAVIOUR" and "HAND-OFF" sections to require the
reply to be, word-for-word, the exact hand-off sentence with nothing added,
removed, or reworded — including for the prompt-injection test case, which
previously received a different, softer refusal instead of the standard
hand-off. This fix, combined with fix #3, is reflected in the jump from 0.83
(Run 2) to 0.90 (Run 4).

## Manual Testing Summary

- **Bug report path:** Verified end-to-end via `chat.py` — the assistant
  asked for missing fields one at a time in the correct order, did not
  re-ask for fields already given, called `create_bug_report` only once all
  three fields were real values, and returned a ticket ID that matches a
  corresponding record in the `bug-report-tool-stack-bug-reports` DynamoDB
  table (see `bug-report-transcript.txt` and `dynamodb-tickets.txt`).
- **Platform question path:** Verified that FAQ-covered questions (delivery
  time, return policy) are answered using only FAQ content, and that a
  question the FAQ does not cover (student discounts) is handed off to the
  human support line rather than answered with an invented policy.
- **Other request path:** Verified that off-topic requests and escalation
  requests are redirected to the human support line using the exact required
  wording.

## Known Limitations / Open Items

- One test case (the ambiguous "My order is wrong" prompt) still scores 0 in
  the final evaluation run — the model handed off rather than asking a
  clarifying question, despite an explicit rule addressing this case. This
  is noted as a residual limitation rather than a fixed issue.
- Automated eval test cases each run in a single, fresh turn, so multi-turn
  bug-report collection behavior is verified manually via `chat.py`
  transcripts rather than through the automated eval dataset.
- The DynamoDB table contains one earlier ticket with placeholder values
  (`"not specified"`) from before Bug #2 was fixed; it has been left in
  place as direct evidence of the bug rather than deleted.