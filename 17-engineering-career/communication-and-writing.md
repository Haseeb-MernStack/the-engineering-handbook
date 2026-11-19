# Communication and Writing

> Most engineering work is coordinated through words, so clear writing and honest communication are core technical skills, not soft extras.

## Overview

Software is built by groups of people who cannot read each other's minds. Requirements, designs, incidents, reviews and decisions all pass through messages, documents and meetings. When those are unclear, the result is rework, missed risks and quiet disagreement that surfaces late.

Writing is also thinking. If you cannot explain a design in a page, you probably do not yet understand it. A good document forces you to find the gaps before the code does.

This matters more as code generation gets cheaper. A model can produce an implementation from a good description. It cannot supply the context, the constraints or the reasons. Engineers who can state a problem precisely, specify what "done" means and explain a decision to others are the ones who direct that capability well.

This topic covers day-to-day communication: messages, status updates, code review comments, technical explanations and meetings. For long-form decision documents see [Design Docs and RFCs](./design-docs-and-rfcs.md).

## Core Concepts

### Know the reader and the purpose

Before writing, answer two questions:

1. Who is reading, and what do they already know?
2. What do I want them to do, decide or understand?

A message to your team lead about a delayed project and a message to a new teammate about the same project should differ. The first leads with impact and the decision needed. The second explains context.

### Lead with the conclusion

Busy readers decide within a few lines whether to continue. Put the point first, then the support.

| Weak | Stronger |
| --- | --- |
| "I was looking at the logs and noticed some timeouts, so I checked the pool config and..." | "The checkout timeouts are caused by an undersized connection pool. I propose raising it to 50. Details below." |
| "Any thoughts?" | "I need a decision by Thursday: use the existing queue or add a new one. I recommend the existing queue because..." |

### Make asks explicit

State what you need, from whom and by when. "FYI" and "decision needed" are different messages. Ambiguity about whether action is required is a common cause of dropped work.

### Be precise

- Use exact terms. "The service is slow" is weak. "p95 latency for `/search` rose from 200 ms to 900 ms after Tuesday's deploy" is useful.
- Separate observation from interpretation: "The error rate rose at 14:02" versus "The deploy caused it".
- Say what you know, what you believe and what you have not checked.
- Define acronyms and local jargon for people outside the team.

### Structure for scanning

Use short paragraphs, headings for longer documents, lists for parallel items and tables for comparisons. Avoid walls of text. Bold sparingly.

### Choosing the channel

| Situation | Better channel |
| --- | --- |
| Quick clarifying question | Chat |
| Decision others need to find later | Written doc or ticket |
| Complex disagreement with emotion | Call or conversation, followed by a written summary |
| Urgent production issue | The incident channel, with a single clear owner |
| Praise | Public, specific |
| Critical feedback | Private, direct, timely |

### Asynchronous habits

Distributed teams depend on messages that stand alone. A good async message includes context, the ask, relevant links and a deadline, so the reader does not need a follow-up question to act.

### Writing a useful bug report or question

Include what you expected, what happened, steps to reproduce, the environment, and what you already tried. This also earns goodwill: people help those who have helped themselves.

### Code review communication

- Comment on the code, not the person. "This loop is quadratic for large inputs" rather than "You wrote a slow loop".
- Distinguish blocking issues from preferences. Label the second: "nit:" or "optional:".
- Ask questions when you are unsure: "What happens if this list is empty?"
- Explain the reason behind a request. A reason can be applied again; an order cannot.
- As an author, keep changes small, describe the intent and respond to every comment, even if only to agree.

### Status updates

A good update answers: what is done, what is next, what is blocked, and what has changed in risk or timeline. Report bad news early and with options. Surprises erode trust faster than delays do.

### Explaining technical ideas

- Start from what the listener knows.
- Use one concrete example before the abstraction.
- Draw it. A box-and-arrow diagram answers many questions.
- Check understanding by asking them to restate it, not by asking "does that make sense?"

### Meetings

Good meetings have an owner, a purpose and an outcome. If the purpose is to share information, consider writing it. If it is to decide, send the options beforehand. End by restating decisions and owners, and send notes.

### Listening

Communication is two-way. Summarize what you heard before replying. Ask what concerns the other person has. Many technical disagreements are really about differing constraints or priorities that neither side stated.

## Trade-offs

**Brevity versus completeness.** Short messages respect time but may omit context that causes a follow-up. Long ones are complete but may go unread. Solve it by structure: a short summary on top, detail below.

**Written versus spoken.** Writing is durable, searchable and scales to many readers, but it is slow and can feel cold. Talking is fast and builds rapport but leaves no record and excludes people not in the room. Combine them: talk, then write down the outcome.

**Directness versus diplomacy.** Directness saves time and avoids misunderstanding. Excessive bluntness damages relationships and makes people hide problems. The aim is clarity about the issue and care for the person.

**Transparency versus noise.** Sharing everything keeps people informed but trains them to ignore messages. Share what the audience needs to act or to avoid being surprised.

**Documenting versus doing.** Documentation has a cost, and stale docs are worse than none. Document decisions and non-obvious reasons, and keep ephemeral detail out.

## Failure Modes

- **Burying the lead.** A key risk sits in the seventh paragraph and no one reads it. Prevent it by stating the conclusion and the ask first.
- **Assumed context.** You wrote for yourself. Readers lack the background and guess. Test by asking whether a new teammate could follow it.
- **Vague asks.** "Can someone look at this?" gets no response. Name a person, a task and a deadline.
- **Silent disagreement.** People nod in the meeting and then do something else. Invite objections explicitly, and ask "what would make this a bad idea?"
- **Tone lost in text.** Short, blunt messages read as hostile. Add a sentence of warmth or context, particularly in review and across cultures.
- **Over-communicating during incidents or under-communicating during projects.** Both create confusion. Match frequency to stakes and agree a cadence.
- **Hiding bad news.** Delay turns small problems into crises and damages trust. Report early with what you know and your plan.
- **Overclaiming certainty.** Presenting guesses as facts leads to decisions built on sand. Label confidence.
- **Pasting generated text unread.** AI-written messages can be fluent and wrong, or generic and padded. You remain responsible for every claim you send. Edit and verify.

## In Practice

### A message template for asking for help

```
Context: What I am trying to do, in one sentence.
Problem: What I expected versus what happened.
Tried: What I have already checked or ruled out.
Ask: What I need (a pointer, a review, a decision) and by when.
```

### A status update template

```
Done: Shipped X; Y is in review.
Next: Z, expected by Friday.
Risks: The vendor API change may delay Z. Mitigation: stub it and swap later.
Need: A decision from product on the retry behavior by Wednesday.
```

### Reviewing your own writing

1. Read it as the recipient, with no context.
2. Cut the first paragraph if it only warms up.
3. Replace vague words (several, soon, some, slow) with numbers or dates.
4. Check that every request is explicit.
5. Read it aloud. Awkward phrases show up.

### Habits that compound

- Write things down: decisions, rationales, runbook steps.
- Keep a personal glossary of project terms and share it.
- Present your work regularly in demos or short write-ups.
- Ask for feedback on your writing from a trusted colleague.
- Read well-written engineering documents, such as published postmortems and design docs, and note why they work.

### Using AI tools for writing

Tools can help with drafting and editing. Use them to tighten structure and catch errors, not to replace your thinking. Give them accurate context, then verify the output against the facts. Never send what you have not read.

## Further Reading

- Strunk and White, *The Elements of Style*. Short and still useful on concision.
- William Zinsser, *On Writing Well*. A guide to clear nonfiction writing.
- Google, *Developer Documentation Style Guide*, developers.google.com/style. A public reference for technical writing.
- Camille Fournier, *The Manager's Path*. Includes practical guidance on communication and feedback.
- Kim Scott, *Radical Candor*. On giving direct feedback with care, though some of its framing is debated.
- Tanya Reilly, *The Staff Engineer's Path*, chapters on communicating and building alignment.
