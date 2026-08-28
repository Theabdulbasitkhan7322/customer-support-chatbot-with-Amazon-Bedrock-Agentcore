# Customer Support Chatbot with Amazon Bedrock AgentCore

A customer support chatbot built on Amazon Bedrock AgentCore that handles three types of customer requests through prompt-based routing — no separate classifier, no condition nodes. Routing, bug-report collection, FAQ answering, and human hand-off are all implemented entirely inside a single system prompt.

## What it does

- **Bug reports** — collects a description, steps to reproduce, and the customer's environment across the conversation, then files a ticket via a Lambda tool (through an AgentCore Gateway) into DynamoDB.
- **Platform questions** — answers orders, shipping, returns, and payments questions strictly from an embedded FAQ document.
- **Everything else** — politely hands off to a human support line.

## Architecture

Customer message → AgentCore Harness (Nova Pro + system prompt) → one of three routes:
- Bug Report → Gateway → Lambda → DynamoDB (ticket created)
- Platform Question → answered from embedded FAQ
- Other → hand-off to human support line

There is no separate Flow, classifier node, or condition node — the model reads the entire routing logic from the system prompt on every turn and decides which of the three behaviors applies.

## Testing and evaluation

- `harness-tests.json` — 10 test cases across all three routes, including two edge cases (an ambiguous message and a prompt-injection attempt)
- `generate-eval-dataset.py` runs the harness against every test case and produces `output_eval_dataset.jsonl`
- Bedrock Evaluations (Nova Pro as judge, Correctness metric) scores the dataset — see `OBSERVATIONS.md` for the full run history and score progression (1.00 → 0.83 → 0.90 across three iterations)

## What I learned

- **Prompt-only routing is real routing.** With a clearly defined category list and explicit disambiguation rules, a single system prompt can reliably route a conversation exactly like a dedicated classifier node would — no separate infrastructure needed.
- **A model's own stated reasoning doesn't guarantee its behavior.** More than once, the model's internal thinking correctly identified a missing field or a rule, then acted against that reasoning anyway (e.g., calling the bug-report tool with a placeholder value after noting the value wasn't really there). Explicit pre-flight self-checks in the prompt, plus worked examples of the exact failure, were what actually fixed this — general rules alone weren't enough.
- **Harness defaults matter as much as the prompt.** The most disruptive bug in this project wasn't a prompt issue at all — it was create_harness.py defaulting to persistent memory, which silently let unrelated conversations bleed into each other. This is a good reminder that infrastructure configuration and prompt engineering are both first-class parts of getting an agent to behave correctly.
- **Automated evaluation and manual testing catch different things.** Correctness scores caught routing and answer-quality issues; they didn't catch the memory bleed or the thinking-tag leakage. Both kinds of testing were necessary to find and fix the real problems in this project.
- **Iteration with evidence beats a single lucky run.** Keeping the earlier, lower-scoring evaluation runs (rather than only submitting the best one) made it possible to show a genuine before/after story grounded in specific, reproducible bugs — which is a more convincing (and more honest) result than a single perfect score.

## Files

| File | Purpose |
|---|---|
| system_prompt.txt | The full routing and behavior prompt (with {{FAQ}} placeholder) |
| harness-tests.json | Test cases covering all three routes plus edge cases |
| output_eval_dataset.jsonl | Harness responses scored by Bedrock Evaluations |
| bug-report-transcript.txt | A full chat.py bug-report conversation, including the tool call and ticket ID |
| dynamodb-tickets.txt | DynamoDB scan output showing tickets created by the chatbot |
| OBSERVATIONS.md | Full run history, bugs found, root causes, and fixes |
