# Levels and Growth

> A career ladder describes how the scope of your responsibility and the reach of your impact grow over time, and understanding it lets you steer your growth deliberately instead of waiting to be noticed.

## Overview

Most engineering organizations describe roles as a ladder: junior, mid-level, senior, staff and beyond. The ladder exists so that pay, expectations and promotion decisions are not arbitrary. It also gives you a map. If you know what the next level asks for, you can build evidence for it.

Level definitions vary by company. A senior engineer at a ten-person startup may do very different work from a senior engineer at a large company, and titles do not transfer cleanly. Treat the descriptions below as a common pattern, not a standard. Always read your own company's published expectations, and ask your manager how they are applied.

The deepest shift on the ladder is not about writing harder code. It is about scope (how large a problem you can own), autonomy (how much direction you need) and impact (how much of the result depends on your judgment). In the age of AI-assisted coding, this framing matters more. Producing code is getting cheaper. Deciding what to build, checking that it is right and carrying the consequences is the part that scales.

## Core Concepts

### Scope, autonomy and impact

- **Scope** is the size of the problem you own: a task, a feature, a system, a group of teams.
- **Autonomy** is how much guidance you need to make progress, and how well you set your own direction.
- **Impact** is the change your work causes for users, the business or other engineers.
- **Leverage** is impact beyond your own hands: making others faster, setting standards, removing whole classes of problems.

### A common ladder

| Level | Typical scope | Typical autonomy | What impact looks like |
| --- | --- | --- | --- |
| Junior | Well-defined tasks within one component | Needs guidance and review; asks good questions | Delivers correct, tested work; learns the codebase and the process |
| Mid-level | Features within a system | Works independently on clear problems; flags risks | Delivers features reliably; owns what they ship |
| Senior | A system or a project spanning several people | Defines the approach; unblocks others | Reduces ambiguity; raises the quality of the team's decisions and code |
| Staff | An area spanning several teams or a hard, unclear problem | Finds and frames the work that matters | Moves outcomes across a group; sets technical direction and standards |
| Principal and beyond | An organization or a company-wide concern | Shapes strategy as well as execution | Influences what the company builds and how it builds it |

Many companies also offer a management track alongside this ladder. It is a different job, not a promotion. See [Leadership and Mentoring](./leadership-and-mentoring.md).

### What changes at each step

**Junior.** The main task is learning to be effective. You turn small, well-specified tasks into working, tested code. You learn to debug, to read unfamiliar code and to use review feedback. The most valuable habit is asking questions early, after you have tried to answer them yourself.

**Mid-level.** You stop needing someone to break work down. You can take a feature from a rough description to production, including tests, rollout and follow-up. You begin to notice problems no one assigned to you.

**Senior.** You own ambiguous problems. Given "checkout is slow", you find out why, propose options and bring others with you. Your reviews teach. You write [design docs](./design-docs-and-rfcs.md), mentor, and say no to work that is not worth doing. Seniority is largely judgment about risk and trade-offs.

**Staff and beyond.** The work often becomes less visible in code and more visible in alignment. Tanya Reilly's *The Staff Engineer's Path* describes the common themes: big-picture thinking, execution on large efforts and leveling up the people around you. Staff engineers often work across team boundaries, where no one has formal authority, so influence comes from trust, clear writing and sound technical judgment.

### Breadth, depth and T-shaped skills

Growth is not only upward. A useful picture is a T: broad working knowledge across the stack, and deep expertise in one or two areas. Fundamentals such as those in [Engineering Fundamentals](../01-engineering-fundamentals/README.md) and [System Design](../05-system-design/README.md) are the broad bar. Depth makes you the person others consult.

### Evidence, not effort

Promotion decisions rest on evidence of operating at the next level, usually over a sustained period. Hours worked and tasks closed are weak evidence. Strong evidence looks like:

- A project you scoped, led and delivered, with the decisions documented.
- Problems you prevented, not just fixed.
- Other engineers who became more effective because of your work.
- Decisions you made under uncertainty, and how they turned out.

## Trade-offs

**Chasing the next title versus building skill.** Titles are a lagging signal. Aiming only at them can lead you to pick visible projects over valuable ones. The cost of ignoring them completely is that your growth may go unrecognized, and compensation follows recognition.

**Depth versus breadth.** Deep specialists solve problems few others can, but their value is tied to one domain staying relevant. Generalists adapt well and connect teams, but may lack the depth needed for the hardest problems. Most careers alternate.

**Staying on the technical track versus moving into management.** Management offers a different kind of leverage and different satisfactions, with less hands-on technical work. Moving costs you some technical currency. Moving back is possible but not free. Many engineers try management for a period precisely to find out.

**Switching companies versus growing in place.** A new company can offer a faster level or pay change and a fresh start. It also resets your reputation, context and relationships, and levels may be calibrated differently. Staying offers accumulated trust and deep context, and may offer fewer opportunities if the company is not growing.

**Hard problems versus safe wins.** Ambitious work builds skill faster but fails more often. Safe wins build a record but can stall learning. A mix is healthier than either extreme.

## Failure Modes

- **Waiting to be noticed.** Good work is not self-advertising. Managers have many reports. If you do not make your impact legible through writing, demos and regular updates, it may not register.
- **Optimizing for the promotion packet.** Engineers who pick projects for how they will look, rather than for what matters, often produce thin work and lose the trust of peers. Prevent it by choosing work on its value and documenting it as you go.
- **Doing the next level's tasks without the next level's judgment.** Taking on bigger tasks is not the same as handling ambiguity well. Seniority shows when things are unclear, not when they are well specified.
- **Becoming a hero.** Always being the one who saves the day signals a fragile team. Staff-level work reduces the need for heroics.
- **Stagnating comfortably.** After three years in the same role on the same code, you may have one year of experience repeated three times. Prevent it by deliberately seeking unfamiliar problems.
- **Misreading the ladder.** Assuming your company's levels match another's leads to bad negotiations and bad expectations. Ask for the written criteria.
- **Neglecting fundamentals.** Over-reliance on tools that generate code, without understanding what it does, caps your growth. You cannot review, debug or design beyond your understanding.

## In Practice

### Make the criteria concrete

1. Find your company's written level expectations. If none exist, ask your manager what separates your level from the next.
2. Compare yourself honestly against each criterion. Mark where you have evidence and where you do not.
3. Agree on one or two gaps to work on over the next quarter.

### Keep a work log

Write short notes weekly: what you shipped, decisions you made, problems you found and who you helped. This is the raw material for performance reviews, promotion cases and your own reflection. Memory is unreliable and recency-biased.

### Use one-on-ones well

- Bring a short agenda: progress, blockers, feedback and growth.
- Ask for specific feedback: "What is one thing that would have made that design review stronger?"
- Ask what the next level looks like in this team, not in general.

### Seek scope deliberately

Volunteer for work slightly beyond your comfort: an on-call improvement, a migration, a cross-team integration, a design that needs an owner. Pair it with a sponsor who can see and speak to the result.

### Build a feedback loop

Collect feedback from peers, not only from your manager. Ask after significant pieces of work. Look for patterns, not single comments.

### Plan for changes in the tools you use

AI assistants change how code is produced. They raise the value of skills they do not replace: framing problems, reading code critically, testing assumptions and making accountable decisions. Treat them as a multiplier on your fundamentals. See [AI Engineering](../14-ai-engineering/README.md).

### Reasonable review cadence

Every six months, ask: Am I learning? Am I trusted with harder problems than a year ago? Do I understand why my company makes money and where my work fits? If the answers are consistently no, change something: team, scope or company.

## Further Reading

- Tanya Reilly, *The Staff Engineer's Path*. A practical guide to scope, influence and leadership beyond senior.
- Will Larson, *Staff Engineer: Leadership Beyond the Management Track*. Profiles and archetypes of staff-level roles.
- Camille Fournier, *The Manager's Path*. Describes the ladder from tech lead to executive, useful for understanding the management side.
- Dave Farley, *Modern Software Engineering*. On the habits that define engineering as a discipline.
- Public career ladders published by companies. Compare several, and remember that each reflects one organization's context.
