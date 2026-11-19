# LLM Fundamentals

> A large language model is a probabilistic function that predicts text one token at a time; knowing what that means, and what it does not, is the basis for using one as an engineering component.

## Overview

A large language model (LLM) is a neural network trained on very large amounts of text to predict what comes next. Given a sequence of tokens, it produces a probability distribution over the next token. A system samples from that distribution, appends the result, and repeats. Everything else, including answering questions, writing code, and summarizing documents, emerges from this loop plus additional training that shapes behavior.

You do not need to train models to use them well. You do need an accurate mental model, because most mistakes with LLMs come from a wrong one. People treat them as databases, as search engines, as compilers, or as colleagues. They are none of these. They are statistical text generators that are often very useful and sometimes confidently wrong.

This topic opens the section. It is deliberately balanced. Models are capable enough to change how software is built. They are unreliable enough that responsible use requires verification. Both statements are true at once. The rest of this section builds on them.

## Core Concepts

### Tokens

Models do not read characters or words. They read tokens, which are chunks of text produced by a tokenizer. A token may be a whole common word, part of a rare word, or a single symbol. Consequences:

- Limits, cost, and latency are measured in tokens, not words.
- Different languages and code styles use different numbers of tokens for the same meaning.
- Tasks that depend on character-level detail, such as counting letters or exact string manipulation, are harder than they look.

### Context window

The context window is the maximum number of tokens the model can consider at once, covering your instructions, supplied documents, conversation history, and the output. Nothing outside it exists for the model. It has no memory between calls unless your application supplies one. "Memory" in a chat product is the application re-sending earlier text or stored summaries.

A larger window does not guarantee equally good use of everything inside it. Models tend to attend unevenly to long inputs, and relevant details can be missed among irrelevant ones. Treat the window as a scarce resource to be curated, not a bin to fill.

### Training stages

At a high level, building a model has several stages:

1. **Pretraining** on a large corpus to predict text. This gives broad knowledge and language ability.
2. **Instruction tuning and preference training** so that the model follows requests and produces helpful, safer responses. Human or automated feedback shapes it.
3. **Optional specialization**, such as fine-tuning on a narrower domain or format.

Knowledge comes mostly from pretraining and is frozen at a cutoff. The model does not know about events after that, and it does not know your private data unless you provide it.

### Sampling and nondeterminism

At each step the model produces probabilities. Sampling parameters control how the next token is chosen:

- **Temperature** scales the distribution. Lower values favor likely tokens and make output more uniform. Higher values add variety.
- **Top-k and top-p** restrict which candidates are considered.

Even at the lowest temperature, many deployments do not guarantee identical output across calls, because of infrastructure details and model updates. Never write code that assumes the same prompt gives the same answer. Design for variance, and test for it.

### Hallucination

A hallucination is output that is fluent and plausible but false or unsupported: an invented citation, a function that does not exist in a library, a confident wrong explanation. It is not a rare glitch. It follows from the mechanism. The model optimizes for plausible continuations, not for truth, and it has no built-in way to know what it does not know.

Hallucination risk is higher for obscure facts, precise details, recent events, and questions that presuppose something false. It is lower when the answer is well represented in training data or is supplied in the prompt. It can be reduced. It cannot be assumed away.

### What models do well

- Transforming and restructuring text: summarizing, rewriting, translating, classifying, extracting fields.
- Producing first drafts of code, tests, documentation, and queries in well-known patterns.
- Explaining unfamiliar code or concepts, with the explanation then checked.
- Brainstorming alternatives and surfacing considerations you had not listed.
- Acting as a flexible natural-language interface to tools and data.

### What models do badly

- Guaranteeing factual accuracy without grounding in a source.
- Exact, multi-step calculation without tools.
- Knowing the current state of your system, repository, or world unless told.
- Staying consistent across very long or complex tasks without structure.
- Recognizing their own errors reliably. A model asked to check its own answer can repeat the same mistake.
- Judgments that depend on accountability, context they were never given, or values specific to your organization.

### Reasoning

Models can produce step-by-step text that improves results on many problems. Whether this constitutes reasoning in a human sense is debated. For engineering, the practical point is narrower. Intermediate steps often help, they can also be plausible and wrong, and the final answer must be verified independently when it matters.

### Capabilities change, fundamentals do not

Specific models, sizes, prices, and benchmark scores change frequently. Build on durable properties instead: outputs are probabilistic, knowledge is frozen and incomplete, input is bounded, behavior can change when a provider updates a model, and cost and latency scale with tokens.

## Trade-offs

| Choice | Gain | Cost |
|--------|------|------|
| Larger or more capable model | Better quality on hard tasks | Higher cost and latency |
| Smaller or faster model | Cheaper, quicker, easier to host | Weaker on complex or subtle tasks |
| Hosted API | No infrastructure, rapid start | Data leaves your boundary, vendor dependency, behavior may change |
| Self-hosted model | Control over data and versions | Operational burden, hardware cost, usually lower peak capability |
| Low temperature | More consistent output | Less variety, still not deterministic |
| Long context | Fewer retrieval steps | More cost, slower, attention dilution |
| Fine-tuning | Consistent style or format, shorter prompts | Data preparation, maintenance, risk of overfitting and staleness |

Cost, latency, and quality pull against each other. Pick the cheapest approach that meets a measured quality bar, not the strongest one by default.

## Failure Modes

### Treating fluency as correctness

Polished prose creates false confidence. A wrong answer in a confident tone passes casual review. Mitigation: require evidence, such as a source, a test, or a run, for claims that matter.

### Invented APIs and facts

Generated code calls a method that does not exist, or uses a deprecated signature. Detect by compiling, type checking, running tests, and checking documentation for anything unfamiliar. Package names that do not exist are also a security concern. See [Responsible and Secure AI Use](./responsible-and-secure-ai-use.md).

### Silent behavior changes

A provider updates a model and your prompts begin to behave differently. Mitigate by pinning versions where possible, running an evaluation suite on every change, and monitoring production quality. See [Evaluation and Testing of LLM Systems](./evaluation-and-testing-llm-systems.md).

### Context overflow and truncation

Inputs grow until the system silently drops the start of a conversation or document. Behavior degrades with no error. Count tokens, set budgets per component, and test with realistic maximum inputs.

### Variance mistaken for improvement

You change a prompt, try it twice, see better output, and ship. The difference was noise. Compare changes over a sample of cases, not anecdotes.

### Over-reliance and skill atrophy

Engineers who accept output without understanding it lose the ability to judge it. The risk grows as the tool improves, because errors become rarer and harder to notice. See [AI-Assisted Development](./ai-assisted-development.md).

### Anthropomorphism

Phrases like "it understood" or "it decided" lead to wrong expectations. A model may be unable to do something that looks trivial to a person, and do something else that looks hard. Judge by measured behavior on your task.

## In Practice

### Questions to ask before using an LLM

- Is the task one where occasional errors are tolerable, or can they be caught cheaply?
- Is there a deterministic way to do this? If so, prefer it. Regular expressions, SQL, and rules are testable and cheap.
- Does the model need information it does not have? How will you supply it?
- How will you know it is working, and how will you know it has stopped?
- What are the cost and latency per call, at the volumes you expect?
- What data are you sending, and may it leave your boundary?

### Worked example: ticket triage

A team wants to route support tickets to the correct queue.

A developer pastes a few tickets into a chat tool, gets sensible categories, and wires up an API call. It works in the demo.

An engineer asks different questions. What are the categories, exactly, and who owns them? What is the cost of a misroute? What is the baseline accuracy of the current manual process? The engineer collects a labeled sample of real tickets, measures how often the model agrees with human labels, examines the disagreements, and finds that two categories are ambiguous even to humans. They fix the category definitions, add a "needs human review" path for low confidence, log every decision with its input, and set up a recurring evaluation. The model is the same. The difference is that the system now has a measurable quality bar and a way to notice when it slips.

### Developer vs. engineer

A developer judges an LLM by whether the demo looks good. An engineer judges it by a measured error rate on representative inputs, knows what it costs when wrong, and has designed what happens next.

### Heuristics

- Treat model output as untrusted input until verified.
- Prefer giving the model the facts over hoping it remembers them.
- Use the model for what is hard to specify and easy to check.
- Keep a deterministic fallback for anything critical.

## Further Reading

- Ashish Vaswani et al., "Attention Is All You Need" (the transformer architecture).
- Dan Jurafsky and James H. Martin, *Speech and Language Processing*.
- Ian Goodfellow, Yoshua Bengio, and Aaron Courville, *Deep Learning*.
- Emily Bender et al., "On the Dangers of Stochastic Parrots" (a critical perspective on language models).
- Documentation from your model provider on tokens, context limits, and sampling parameters.
