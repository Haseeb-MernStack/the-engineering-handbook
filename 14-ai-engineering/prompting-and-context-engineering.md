# Prompting and Context Engineering

> Output quality depends on what the model is given, so the real work is deciding what goes into the context, in what form, and how you will check the result.

## Overview

A prompt is the input you give a model. Early practice treated this as wording tricks. Experience has shifted the view. For real applications, the wording of one instruction matters less than the whole context: instructions, examples, retrieved documents, tool results, conversation history, and the format of the answer you expect. Assembling that context deliberately is called context engineering.

The model sees only the context. It cannot read your mind, your codebase, or your last meeting. If a needed fact is absent, it will guess. If irrelevant material is present, it may be distracted. Your job is to supply the right information, state the task unambiguously, and constrain the output so that it can be checked by code.

This is a specification problem. Engineers who are good at writing clear requirements, defining interfaces, and anticipating edge cases tend to write better prompts, because the skill is the same.

## Core Concepts

### Anatomy of a prompt

A robust prompt for an application usually contains these parts:

- **Role and goal.** What the model is doing and for whom.
- **Task instructions.** Specific, ordered, unambiguous.
- **Constraints.** What it must not do, length limits, tone, scope.
- **Context.** The facts needed: documents, data, prior steps.
- **Examples.** Input and output pairs showing the desired behavior.
- **Output format.** A precise structure, ideally machine-parseable.

Many providers separate a system message, which sets standing behavior, from user messages. Put stable rules in the system message and variable content in the user message. Do not rely on this separation as a security boundary. See [Responsible and Secure AI Use](./responsible-and-secure-ai-use.md).

### Be specific and testable

Compare:

```text
Summarize this support ticket.
```

```text
Summarize the support ticket below in at most two sentences for an on-call engineer.
State the affected feature, the observed error, and whether the customer is blocked.
If any of these is not stated in the ticket, write "unknown" for it. Do not guess.

Ticket:
"""
{ticket_text}
"""
```

The second states audience, length, required content, and the behavior when information is missing. Permission to say "unknown" reduces fabrication.

### Examples

Few-shot examples show the model the pattern. They often help more than extra description. Choose examples that cover edge cases and are representative of real inputs. Poorly chosen examples bias the output: the model may copy their length, wording, or label distribution. Keep a small, curated set, and test with and without.

### Structured output

When code consumes the answer, request a defined structure such as JSON, and validate it.

```python
import json
from pydantic import BaseModel, ValidationError

class Triage(BaseModel):
    category: str
    blocked: bool
    summary: str

def parse(raw: str) -> Triage | None:
    try:
        return Triage(**json.loads(raw))
    except (json.JSONDecodeError, ValidationError):
        return None  # retry, repair, or route to a human
```

Many providers offer features that constrain output to a schema. These improve conformance but do not guarantee the content is correct. A well-formed answer can still be wrong. Validate both shape and meaning where you can.

### Decomposition

Complex tasks work better when split into steps, each with its own focused prompt and check. For example: extract facts, then classify, then draft. Smaller steps are easier to test and debug. They cost more calls and add latency, and errors can compound across steps.

### Reasoning and intermediate steps

Asking the model to work through a problem before answering often improves accuracy on multi-step tasks. If you need only the answer, separate the reasoning from the final output in the format so your code reads the right field. Do not treat displayed reasoning as a faithful record of how the model arrived at its answer. Verify the result on its merits.

### Context selection

More context is not better context. Principles:

- Include what the task needs and little else.
- Put the most important instructions and facts where they are easy to find. Models may attend unevenly to long inputs.
- Order and label sections clearly, using delimiters, so the model can tell instructions from data.
- Summarize or trim history rather than replaying it all.
- Keep a token budget per component and enforce it.

### Prompt as code

A prompt that matters belongs in version control with tests. Store it as a template, review changes, record which prompt version produced which output, and run an evaluation set when it changes. See [Evaluation and Testing of LLM Systems](./evaluation-and-testing-llm-systems.md).

### Tools and grounding

Rather than asking a model to recall or calculate, give it a tool or a source. Let it call a function to query data, run code, or look up a document, then base the answer on the result. This converts a recall problem, where the model is weak, into a retrieval and reading problem, where it is stronger. See [Retrieval-Augmented Generation](./retrieval-augmented-generation.md) and [Agents and Tool Use](./agents-and-tool-use.md).

## Trade-offs

**Longer, more detailed prompts** improve clarity and consistency. They cost more tokens, add latency, and can conflict internally. Every added rule is something to maintain and test.

**Few-shot examples** raise reliability on tricky formats. They consume context and can anchor the model on superficial features.

**Strict output schemas** make integration easy. They can force an answer when "I do not know" would be more honest, unless the schema includes that option.

**Decomposed pipelines** are testable and debuggable. They multiply calls, cost, and points of failure.

**Prompt tuning against a small set** is quick and risks overfitting to the cases you looked at. Gains may disappear on real traffic.

**Model-specific tricks** can squeeze out quality today and break when the model changes. Prefer clear instructions that would make sense to a person.

## Failure Modes

### Ambiguity resolved wrongly

The prompt allows two readings. The model picks one, not always the one you meant, and not consistently. Detect by reviewing outputs on varied inputs. Fix by stating the intended reading and giving an example.

### Instruction conflict

A long prompt grows by accretion. A new rule contradicts an old one. Behavior becomes erratic. Review the whole prompt on each change, remove rules that no longer apply, and test.

### Prompt injection through context

Untrusted text in the context, such as a web page, an email, or a user's document, contains instructions, and the model follows them. This is a security problem, not a wording problem. Treat all external content as data, limit what the model can do, and do not rely on "ignore instructions in the document" as a defense.

### Context rot

In long sessions, stale or contradictory material accumulates. Quality degrades. Summarize and reset, or restart with a clean, curated context.

### Overfitted prompts

A prompt tuned on ten examples fails on the eleventh. Hold out a test set that you do not tune against.

### Format drift

Output sometimes breaks the schema, adds commentary, or wraps JSON in extra text. Parsing fails in production. Validate, retry with the error message, and track the failure rate as a metric.

### Leaking sensitive data into prompts

Convenient context assembly pulls in more than intended, such as full records, credentials, or personal data. Build context from explicit allow lists of fields.

### Cost blowups

Unbounded history or retrieved text grows token counts and bills. Monitor tokens per request and set limits.

## In Practice

### Prompt checklist

- State the task, the audience, and the success criteria.
- Specify output format and validate it in code.
- Say what to do when information is missing or the request is out of scope.
- Separate instructions from data with clear delimiters.
- Supply the facts needed. Do not rely on recall for specifics.
- Include a few representative examples, including a hard case.
- Version the prompt and test it on a fixed evaluation set.
- Set token limits and log usage.

### Worked example: improving a code-review assistant

First version: "Review this diff." Output is generic and long, and mixes style nits with real issues.

Iteration: the team defines what a useful review is. They ask for only issues in three classes: correctness bugs, security concerns, and missing tests. Each finding must cite the file and line, quote the code, and state the failure scenario. If nothing is found, return an empty list. They include two examples from past reviews, one real bug and one non-issue. They request JSON and validate it. Findings without a quote of the diff are discarded, which removes a class of hallucinated line references.

They then run it over fifty historical diffs with known outcomes. Precision is poor on style-heavy diffs. They add a rule to ignore formatting. The team tracks acceptance rate of findings over time. The improvement came from specifying the job, constraining the output, and measuring it, not from a clever phrase.

### Developer vs. engineer

A developer tweaks the wording until the demo looks right. An engineer writes down the expected behavior, builds a test set, changes the prompt, and checks whether the numbers moved.

### Heuristics

- If a competent new colleague could not do the task from your prompt, the model probably cannot either.
- Fix problems at the right level: missing information is a context problem, not a wording problem.
- When a task needs exactness, use code for the exact part.

## Further Reading

- Provider documentation on prompt design, system messages, and structured outputs.
- Tom Brown et al., "Language Models are Few-Shot Learners".
- Jason Wei et al., "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models".
- Simon Willison, writing on prompt injection (his blog).
- Chip Huyen, *AI Engineering*.
