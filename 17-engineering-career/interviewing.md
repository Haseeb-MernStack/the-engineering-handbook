# Interviewing

> Technical interviews are an imperfect attempt to predict on-the-job performance, and both candidates and interviewers do better when they understand what is being measured and where the process is unreliable.

## Overview

Almost every engineer sits on both sides of the interview table over a career. As a candidate you want to show your real ability under artificial conditions. As an interviewer you want to judge ability fairly with limited time. Understanding both sides improves each.

Be honest about the state of practice. Interviews are noisy. Formats vary widely: algorithm problems at a whiteboard, take-home projects, system design discussions, pair programming on real code, and behavioral conversations. None predicts performance perfectly, and research on selection generally finds that structured approaches, with consistent questions and scoring criteria, do better than unstructured conversation. Companies still disagree about the best mix.

Generated code changes what to test. If a tool can write a standard algorithm on request, the more telling signals are how a candidate frames a problem, reasons about trade-offs, reads and critiques code, debugs and communicates. Fundamentals matter because they let a person judge whether output is correct. Many companies are still adjusting their formats, so check how a specific employer treats AI tools in interviews, and follow its stated rules.

## Core Concepts

### What interviews try to measure

| Signal | Example evidence |
| --- | --- |
| Problem solving | Breaks down an unfamiliar problem; tests ideas; adapts when stuck |
| Technical fundamentals | Data structures, complexity, concurrency, databases, networks |
| Coding ability | Writes clear, working, tested code |
| System design | Handles requirements, scale, trade-offs and failure |
| Communication | Explains reasoning; asks clarifying questions; listens |
| Collaboration | Takes hints; disagrees respectfully; credits others |
| Ownership and judgment | Describes decisions, mistakes and outcomes honestly |
| Motivation and fit | Interest in the work; compatible working style |

"Fit" is risky. Used loosely it becomes a cover for bias toward people similar to the interviewer. Prefer "values alignment" defined by observable behaviors, such as how someone handles disagreement.

### Common formats

- **Recruiter or phone screen.** Basic alignment, motivation and logistics.
- **Coding interview.** Solve one or two problems, often algorithmic, in a shared editor or on a whiteboard.
- **Take-home exercise.** A small project done in your own time. Realistic, but time-consuming and uneven for candidates with caregiving or other commitments.
- **Pair programming or code review.** Work on or read realistic code. Tends to reflect daily work.
- **System design.** Design a service from loose requirements. See [System Design](../05-system-design/README.md).
- **Behavioral interview.** Describe past situations: conflict, failure, leading, ambiguity.
- **Hiring manager or team conversation.** Expectations, growth and mutual fit.

### For candidates: preparation

1. **Fundamentals.** Revisit data structures, algorithms, complexity, and your main language. See [Data Structures and Algorithms](../03-data-structures-algorithms/README.md).
2. **Practice aloud.** Solving silently differs from thinking aloud. Practice with a friend or a recording.
3. **System design.** Learn a repeatable framework: clarify requirements, estimate load, sketch the architecture, deep dive, discuss trade-offs and failure.
4. **Stories.** Prepare four to six real examples from your work: a hard bug, a disagreement, a failure, a project you led, something you learned. Structure each as situation, action, result and reflection.
5. **The company.** Understand the product, the stack and the team's problems. Prepare questions.
6. **Logistics.** Test your setup, and have your environment ready.

### For candidates: during the interview

- **Clarify first.** Ask about inputs, constraints and edge cases before coding.
- **State your approach before writing.** A brief plan shows structure.
- **Start simple.** A correct brute-force solution, then improve it.
- **Think aloud.** Interviewers can only assess what they can hear.
- **Test your own code.** Walk through an example and edge cases.
- **Take hints gracefully.** Using a hint well is a positive signal.
- **Be honest.** Saying "I have not used that, but here is how I would approach it" is better than bluffing.
- **Ask questions at the end** about the team, expectations and how success is measured.

### For interviewers: designing a good interview

1. Decide what the role needs. List four or five signals.
2. Assign each interview to specific signals, so that interviewers do not all test the same thing.
3. Use realistic problems with room for multiple levels of depth.
4. Write down the questions and the scoring criteria in advance.
5. Take notes on evidence, not impressions.
6. Submit your assessment independently before discussing, to avoid anchoring on others' views.
7. Calibrate regularly by reviewing real candidates' outcomes.

### Example rubric

A rubric describes what evidence corresponds to each rating. This one is for a 45-minute coding interview for a mid-level engineer. Adapt it to your context and level.

| Dimension | 1: Weak | 2: Mixed | 3: Meets bar | 4: Strong |
| --- | --- | --- | --- | --- |
| Problem understanding | Starts coding without clarifying; misses requirements | Asks some questions; misses key edge cases | Clarifies inputs, constraints and edge cases before starting | Surfaces non-obvious cases; proposes reasonable assumptions |
| Approach | No clear plan; random attempts | Reaches a working approach with heavy hints | Chooses a sound approach; explains complexity | Compares alternatives and justifies the choice |
| Implementation | Code does not run or is unclear | Works on the main case; messy or buggy | Correct, readable code with sensible naming | Clean, idiomatic, well factored |
| Testing and debugging | Does not test | Tests when prompted; slow to find bugs | Tests with examples and edge cases; finds own bugs | Systematic; anticipates failure cases |
| Communication | Silent or unclear | Explains partially | Explains thinking clearly; responds to hints | Collaborative and precise; makes the interviewer's job easy |

Overall recommendation: strong no, no, yes, strong yes, each with a written justification citing specific evidence. A rating without evidence should not count.

### Evaluating behavioral answers

Look for specifics, not slogans. Strong answers describe the candidate's own actions (not "we" throughout), the reasoning at the time, the outcome and what they learned. Probe with follow-ups: "What did you consider and reject?" "What would you do differently?"

### Offers and negotiation

Offers involve compensation, level, scope and growth. Do not invent market numbers. Research current ranges from reliable, recent sources and from people in similar roles, and treat any single figure as approximate. Be polite, clear and honest. Ask about level and expectations, not only salary. Compare total compensation, including equity terms, and consider the team and manager as much as the number.

## Trade-offs

**Algorithm puzzles versus realistic tasks.** Puzzles are standardized and easy to compare but only loosely related to daily work and favor those who have drilled them. Realistic tasks reflect the job better but are harder to standardize and take longer.

**Take-homes versus live sessions.** Take-homes reduce performance anxiety and allow deeper work, at the cost of candidate time and the risk of unauthorized help. Live sessions are quick and observable but stressful and artificial.

**Structure versus flexibility.** Structured interviews are fairer and more consistent. They can feel rigid and miss unusual strengths. Allow follow-ups within the structure.

**Speed versus thoroughness.** More rounds give more evidence but exhaust candidates and risk losing them. Fewer rounds are respectful, but each decision carries more risk.

**Pedigree versus evidence.** Credentials are a cheap filter that excludes capable people without them. Evidence takes effort to gather. Strong programs look at demonstrated work, such as the projects in [Real-World Projects](../18-real-world-projects/README.md).

## Failure Modes

- **The solitary grinder.** A candidate memorizes hundreds of solutions and falters on any variation. Prevent it by understanding patterns and practicing explanation.
- **Silent solving.** Interviewers cannot give credit for unspoken thought. Narrate.
- **Bluffing.** Claiming knowledge you lack is easily probed and costs credibility.
- **Ignoring the question behind the question.** Offering code before understanding the need. Ask.
- **Interviewer bias.** Affinity, first impressions and a candidate's confidence can dominate the decision. Counter with structured criteria, evidence-based notes and independent scoring.
- **Trivia and gotchas.** Questions with a single trick answer measure luck. Prefer problems with multiple paths.
- **The unprepared interviewer.** Improvised questions with no criteria yield noise and a poor candidate experience.
- **Ghosting candidates.** Failing to give timely decisions harms the employer's reputation. Communicate promptly and kindly.
- **Hiring a clone.** Selecting for similarity narrows a team's perspectives. Define the bar by capability and add diverse viewpoints to the panel.
- **Overfitting to AI-assisted answers.** Fluent, generated-looking answers may hide shallow understanding. Ask follow-ups that require reasoning about the code: change a requirement, ask for failure cases, ask why.
- **Treating rejection as a verdict.** Interviews are noisy. A rejection reflects one process on one day. Ask for feedback, and learn what you can.

## In Practice

### A candidate preparation plan (four weeks)

| Week | Focus |
| --- | --- |
| 1 | Fundamentals review; solve two or three problems a day, explaining aloud |
| 2 | System design practice; sketch designs and write down trade-offs |
| 3 | Behavioral stories; mock interviews with a peer |
| 4 | Company research; targeted practice on weak areas; rest |

Adjust to your own situation. Quality of practice matters more than volume.

### A system design discussion outline

1. Clarify functional and non-functional requirements.
2. Estimate scale: users, requests, storage.
3. Sketch the high-level architecture.
4. Dive into one or two critical components.
5. Discuss data model, caching, consistency and failure.
6. Close with trade-offs and what you would do next.

### Behavioral story template

```
Situation: Context and stakes, in two sentences.
Task: Your responsibility.
Action: What you did and why, including alternatives considered.
Result: The outcome, with evidence where you have it.
Reflection: What you learned and would change.
```

### Interviewer checklist

- Read the candidate's materials beforehand.
- Open with a brief, friendly introduction and explain the format.
- Give the problem clearly, and offer hints in stages.
- Take notes on evidence as you go.
- Leave time for the candidate's questions.
- Write your feedback within hours, citing specifics.
- Review your own decisions against later outcomes when you can.

### After the interview

Write down the questions you were asked and how you handled them while it is fresh. Send a brief thanks. Reflect on gaps and fix them before the next one.

## Further Reading

- Gayle Laakmann McDowell, *Cracking the Coding Interview*. A widely used preparation guide, centered on algorithmic interviews.
- Alex Xu, *System Design Interview*. A common preparation text for design rounds.
- Laszlo Bock, *Work Rules!* Describes Google's approach to structured hiring; read it as one company's account.
- Frank Schmidt and John Hunter, "The Validity and Utility of Selection Methods in Personnel Psychology" (1998), a well-known review of selection methods. Later work has revised some of its estimates.
- Camille Fournier, *The Manager's Path*, chapters on hiring.
