# Leadership and Mentoring

> Leadership in engineering is the practice of making a group of people more effective, and it begins long before anyone gives you a title.

## Overview

Leadership is often confused with management. Management is a role with formal responsibility for people: hiring, performance, compensation, team health. Leadership is behavior. A junior engineer who documents a confusing process and helps a new hire is leading. A manager who hoards decisions is not.

Technical leadership has three common forms. Leading a project means taking responsibility for a result involving several people. Leading by expertise means being the one whose judgment shapes how things are built. Leading through people means mentoring, coaching and growing others. Most engineers do some of each, and the mix changes with level. See [Levels and Growth](./levels-and-growth.md).

Mentoring is the most accessible entry point. It benefits the mentee, and it benefits the mentor, who has to articulate what they know. It is also how a team's knowledge survives turnover.

In an era of AI-assisted coding, this work gains weight. Tools help individuals produce more code, which raises the stakes on judgment: what to build, what to trust, how to review. Passing on that judgment is a human task.

## Core Concepts

### Leadership without authority

Most technical leadership happens without formal authority. Influence is earned through:

- **Competence.** People follow those whose technical judgment has proved sound.
- **Reliability.** You do what you say, and say so when you cannot.
- **Clarity.** You make the situation, the options and the reasoning understandable.
- **Generosity.** You give credit, share context and help others succeed.
- **Care.** People can tell if you are working for the group or for yourself.

### Leading a project

A project lead does not need to write the most code. They need to:

1. Define the goal and what done means.
2. Break the work into pieces that people can own.
3. Surface risks early and decide what to do about them.
4. Keep stakeholders informed without being asked.
5. Remove blockers, including organizational ones.
6. Make or facilitate decisions, and record them.
7. Close the loop: ship, review what happened and learn.

### Delegation

Delegating is giving someone ownership of an outcome, not only a task. Good delegation states:

- The desired result and why it matters.
- Constraints: deadlines, quality bar, interfaces to respect.
- The level of autonomy: decide and inform, or check first.
- How and when you will check in.

Then let go. Rescuing work at the first stumble tells the person you do not trust them.

### Mentoring

Mentoring is a relationship, usually with a less experienced colleague, aimed at their growth. It differs from managing, because the mentor does not evaluate performance.

Mentoring often combines:

- **Teaching.** Explaining concepts and techniques.
- **Coaching.** Asking questions that help the mentee find their own answers.
- **Sponsoring.** Advocating for them, suggesting them for opportunities and speaking to their strengths when they are not present.
- **Modeling.** Showing how you think through a problem, including your mistakes.

### Teaching versus coaching

If someone is stuck on an unfamiliar tool, tell them. If they are stuck on a decision, ask: "What are the options? What would you check first? What would change your mind?" Telling too early removes the learning. Asking too long wastes time and frustrates. Judge by whether the person lacks knowledge or lacks confidence.

### Giving feedback

Good feedback is specific, timely, about behavior and its effect, and focused on the future.

A simple structure:

1. **Situation.** When and where.
2. **Behavior.** What you observed, without judgment.
3. **Impact.** What resulted.
4. **Next.** What to try, or a question.

Example: "In yesterday's design review, you interrupted Priya twice. She stopped contributing afterwards, and we lost her view on the migration. Next time, could you note your point and wait until she finishes?"

Praise deserves the same specificity. "Your rollback plan caught the bad migration on Tuesday" is more useful than "Great job."

### Receiving feedback

Leaders also have to take feedback. Listen fully, ask for examples, thank the person, and decide later what to do. You need not agree with all of it. Patterns across several people deserve attention.

### Code review as mentoring

Review is a daily teaching channel. Explain why, link to resources, praise good decisions, and avoid dictating style on matters that are preferences. Where a junior makes a repeated mistake, a short conversation may beat a dozen comments.

### Psychological safety

Teams perform better when people can ask questions, admit mistakes and disagree without fear. Leaders create this by how they react: thanking people who raise problems, owning their own errors openly and running blameless reviews of incidents. Amy Edmondson's research on psychological safety is the usual reference.

### Setting direction and standards

Leaders influence the bar: what a good pull request, test or design looks like. Standards are best set by example and by shared agreement, then supported with tools such as linters, templates and checklists.

### Onboarding

A new engineer's first weeks set their trajectory. Provide a documented setup, a small first task, a named buddy and context on the product and the people. Better yet, ask new hires to improve the onboarding docs as they go.

## Trade-offs

**Doing versus enabling.** Doing the task yourself is faster now. Enabling others is slower now and pays back repeatedly. The cost of always doing is a team that cannot function without you.

**Autonomy versus oversight.** Autonomy motivates and develops people but risks drift and mistakes. Oversight prevents errors but breeds dependence. Match it to the person's experience and the cost of failure.

**Consensus versus decisiveness.** Collective decisions build ownership. Slow consensus can be worse than a good decision made quickly. Be explicit about which mode you are in: "I am gathering input, and I will decide on Friday."

**Mentoring time versus personal output.** Time spent mentoring reduces your own output in the short term. Organizations vary in whether they recognize this. Make the work visible, and agree on expectations with your manager.

**Technical track versus management.** Management gives leverage through people and structure, with less hands-on engineering. Individual contributor leadership keeps you close to the technology, with leverage through influence. Neither is better in general. Try what fits, and know you can move.

## Failure Modes

- **The bottleneck lead.** Every decision passes through one person. Prevent it by delegating decisions, documenting principles and trusting others to apply them.
- **The hero.** Solving every crisis personally prevents the team from learning and hides systemic issues.
- **Micromanagement.** Excess control signals distrust and drives away capable people. Define outcomes, not keystrokes.
- **Abdication.** The opposite: letting people flounder without context or support and calling it autonomy. Check in on a schedule.
- **Mentoring as lecturing.** One-way advice that ignores the mentee's goals. Ask what they want, and listen.
- **Unspoken feedback.** Avoiding hard conversations lets problems grow. Deliver it early, privately and kindly.
- **Playing favorites.** Giving visible opportunities to the same few people. Review who gets stretch work.
- **Taking credit, deflecting blame.** The fastest way to lose trust. Do the reverse.
- **Copying a style that does not fit you.** Leaders imitate a charismatic example and come across as false. Develop your own style on shared principles.
- **Passing on shortcuts, not fundamentals.** Teaching juniors to rely on generated answers without understanding leaves a team that cannot debug its own system. Insist on explanations.

## In Practice

### Running a good mentoring relationship

- Agree on purpose, frequency and duration at the start. Monthly or fortnightly meetings are common.
- Let the mentee set the agenda. Ask what they want to get better at.
- Share stories of your own failures, not only your wins.
- Suggest stretch tasks and be available if they need help.
- Review periodically whether it is working, and end it gracefully when it has run its course.

### A one-on-one question bank

- What is going well? What is frustrating?
- What did you learn this week?
- Where do you feel stuck?
- What would you do if you were in charge of this project?
- What would you like to be doing in a year?

### Leading a small project, step by step

1. Write a one-page brief: goal, scope, non-goals, timeline, risks.
2. Hold a short kickoff. Assign owners.
3. Hold brief, regular check-ins. Track decisions in writing.
4. Escalate blockers early with options.
5. After shipping, run a blameless retrospective and record lessons.

### Leading during an incident

Name a single incident commander. Separate roles: someone fixes, someone communicates, someone records. Keep communication calm and factual. After the incident, write a blameless review focused on systems and decisions. See [DevOps, Cloud and Infrastructure](../09-devops-cloud-infrastructure/README.md).

### Developing yourself as a leader

- Ask for feedback on how you lead, not only what you deliver.
- Observe leaders you admire and note specific behaviors.
- Volunteer to coordinate small efforts, onboard a newcomer or run a team meeting.
- Read widely on management and teams, then test ideas on your own team and keep what works.

## Further Reading

- Camille Fournier, *The Manager's Path*. A practical guide to leading from mentor to executive.
- Tanya Reilly, *The Staff Engineer's Path*. Leadership without formal authority.
- Will Larson, *An Elegant Puzzle: Systems of Engineering Management*. Systems thinking about teams.
- Amy Edmondson, *The Fearless Organization*. On psychological safety in teams.
- Michael Lopp, *Managing Humans*. Informal, experience-based essays on managing engineers.
- Lara Hogan, *Resilient Management*. A short book on the basics of supporting a team.
