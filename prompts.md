# Prompts & Bake-off Notes

The exact prompts used in `llm_classifier.ipynb` and the verdict.

> **Where these ran:** the hosted Gemini free tier on this key is capped at **20 requests/day**, and
> the bake-off alone is 24 calls, so per the README's allowance the bake-off (Task 2) and the
> temperature sweep (Task 1) ran against the **local Ollama `llama3.2:3b`** model. The genuine hosted
> calls are concentrated where they teach the most (Task 1 first call + token usage, Task 3 structured
> comparison). All labels below are therefore the *local* model's, at `temperature = 0`.

System message for every classifier call: `You are a precise ticket classifier.`

## Task 2 — Prompt-pattern bake-off

### Zero-shot prompt

```
Classify the support ticket into exactly one of: billing, bug, feature_request, other.
Reply with ONLY the label.

Ticket: "<ticket text>"
```

### Few-shot prompt

```
Classify the support ticket into exactly one of: billing, bug, feature_request, other.

Examples:
Ticket: "Your service charged my card the wrong amount." -> billing
Ticket: "The page goes blank and shows an error when I save." -> bug
Ticket: "Please add a way to schedule reports weekly." -> feature_request

Reply with ONLY the label.

Ticket: "<ticket text>"
```

### Chain-of-thought / decomposition prompt

```
Classify the support ticket into exactly one of: billing, bug, feature_request, other.
First reason in one short sentence, then on a new line write 'Label: <label>'.

Ticket: "<ticket text>"
```

(For CoT the code keeps the reasoning but parses only the text after `Label:`.)

### Results (local llama3.2:3b, temp 0)

| id | ticket (truncated)              | zero-shot       | few-shot        | chain-of-thought |
|----|---------------------------------|-----------------|-----------------|------------------|
| 1  | charged twice / refund          | billing         | billing         | billing          |
| 2  | export button throws 500        | bug             | bug             | bug              |
| 3  | add a dark mode                 | feature_request | feature_request | feature_request  |
| 4  | how do I reset my password?     | bug             | bug             | billing          |
| 5  | app crashes on startup          | bug             | bug             | bug              |
| 6  | send me a copy of my invoice    | billing         | other           | billing          |
| 7  | please add PDF export           | feature_request | feature_request | feature_request  |
| 8  | "the new UI looks clean"        | other           | other           | feature_request  |

### Verdict (which pattern won, where each failed)

On the five unambiguous tickets (1, 2, 3, 5, 7) **all three patterns agreed and were correct**, so the
prompt pattern only mattered on the ambiguous ones. **Zero-shot and few-shot effectively tied for the
win** and were the most stable; few-shot did *not* clearly beat zero-shot here — and its examples even
hurt once, dragging the "send me a copy of my invoice" request (#6) into `other` when `billing` is the
better fit. **Chain-of-thought was the least reliable**: the extra reasoning over-interpreted intent,
turning a plain compliment ("the new UI looks clean", #8) into `feature_request` and the password-reset
question (#4) into `billing`. The shared failure is structural, not about prompting: "how do I reset my
password?" (#4) is really an `other`/how-to question, but with no strong cue every pattern guessed `bug`
or `billing` — a sign the label set lacks a clean home for support how-tos. Net: for short tickets into a
small fixed label set, a tight zero-shot instruction is the best effort-to-reliability trade; few-shot
helps only if the examples are chosen not to bias edge cases, and CoT adds failure modes without a clear
payoff at this scale.

## Task 3 — Structured output notes

Same structured instruction sent to both models, `temperature = 0`, asking for
`{label, confidence, reason}`. Gemini used schema-constrained mode (`response_schema`); Ollama used the
OpenAI-style `{"type": "json_object"}` mode.

**Reliability on this run:** the local `llama3.2:3b` returned **valid JSON for all 8** tickets. The
hosted Gemini calls were throttled by the 20/day cap — only 2 went through (1 passed our validator, 1
was rejected by it), the rest show `quota`. So I can't claim a clean hosted-vs-local reliability gap
from these numbers; the honest takeaways are (a) structured/JSON mode is what makes either model's output
parseable, and (b) **validation is non-negotiable even for hosted output** — `validate_classification`
checks the label is allowed and `confidence ∈ [0,1]`, and on any failure (bad JSON, unknown label,
out-of-range confidence, missing field) the pipeline falls back to a safe `{label: "other",
confidence: 0.0}` record instead of crashing. Four deliberately malformed outputs are run through that
guard in the notebook and all four are caught.
