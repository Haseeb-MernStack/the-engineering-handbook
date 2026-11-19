# Community and Governance

> Governance is the set of rules and habits that decide who can change a project, how disputes are settled, and what happens when people leave, and it determines whether a community survives growth.

## Overview

Every project has governance, even if nobody wrote it down. In a small project, it is usually "the founder decides." That works until the project grows, the founder tires, or a disagreement arises that the founder cannot settle fairly because they are a party to it.

Community is the people. Governance is the structure that lets those people cooperate without relying on goodwill alone. Good governance is mostly boring. It answers predictable questions in advance: who may merge, how a maintainer is chosen, how a conflict is escalated, and how a decision is recorded.

Engineers meet these questions at work too. Teams that never decide how decisions get made make them badly, slowly, or by whoever shouts loudest. Open source makes the problem visible because nobody can fall back on a manager.

## Core Concepts

### Roles

Many projects use some variation of these roles. Names differ.

| Role | Typical responsibility |
|---|---|
| User | Uses the project; reports problems |
| Contributor | Submits changes, docs, or triage |
| Reviewer or triager | Reviews changes or manages issues, with limited rights |
| Committer or maintainer | Merges changes; shares responsibility for quality and direction |
| Steering group or core team | Sets direction, resolves disputes, manages the project's assets |
| Project lead | Final decision-maker in some models |

Define how a person moves between roles. Vague criteria create resentment. Clear ones, such as "sustained, reviewed contributions over several months, nominated by an existing maintainer and approved by the others," create a path.

### Common governance models

- **Benevolent dictator or founder-led.** One person has final say. Decisions are fast and coherent. The risk is dependence on one person's judgment, health, and availability.
- **Meritocratic or maintainer council.** A group of earned maintainers decides, often by consensus with a fallback vote. Decisions are slower but more resilient.
- **Foundation-hosted.** The project sits under a non-profit foundation that holds trademarks, funds, and legal entities, with technical governance handled by the project. The foundation provides neutrality and administrative support, and it brings its own process and expectations.
- **Company-led.** A company employs the main maintainers and directs the roadmap. Resources are stable. The project's direction depends on the company's interests, and licensing or strategy may change.
- **Consensus with enhancement proposals.** Significant changes go through a written proposal process. Python's PEP process and Rust's RFC process are well-known examples of this pattern. The format is widely imitated. See [Design Docs and RFCs](../17-engineering-career/design-docs-and-rfcs.md).

No model is correct in general. Choose by project size, risk, and the people involved.

### Code of conduct

A code of conduct states which behavior is expected and which is not, and who handles reports. It is useful only if enforced. Include a private reporting channel, a named group that is not the accused, and a description of possible outcomes. Many projects adopt a published template, such as the Contributor Covenant, and adapt it.

### Decision-making and records

- Make decisions in public channels where possible. Private channels create invisible rules.
- Record significant decisions with context and rejected alternatives. A short decision log or architectural decision record helps newcomers understand why things are as they are.
- Define how a stalemate breaks: a vote, a designated tie-breaker, or a deferral with a deadline.

### Communication channels

Match the channel to the purpose.

| Channel | Best for | Weakness |
|---|---|---|
| Issue tracker | Bugs and scoped proposals | Poor for open-ended discussion |
| Discussion forum | Questions, design discussion, announcements | Slower than chat; requires moderation |
| Real-time chat | Quick help, social cohesion | Knowledge disappears; excludes other time zones |
| Mailing list | Long-form, archived discussion | Heavy for newcomers |
| Video calls | Resolving tense or complex topics | Not searchable unless notes are published |

Write down the decision, not only the conversation. Real-time chat is a poor system of record.

### Money, trademarks, and ownership

Projects eventually handle funds, domains, package registry namespaces, and trademark. Decide who owns these. Assets held by one individual are a risk to the project and to that individual. A fiscal host or foundation can hold assets and manage legal exposure.

### Growing a community

- Make the first step obvious: a clear README, a welcoming contributing guide, and labeled starter issues.
- Respond quickly to first-time contributors, even when the answer is no.
- Recognize contributions beyond code.
- Keep documentation current. It is where most newcomers decide whether to stay.
- Say what the project needs, not only what it offers.

## Trade-offs

| Choice | What it gives you | What it costs |
|---|---|---|
| Founder control | Speed, vision | Bottleneck; succession problem; contributors may feel they have no stake |
| Shared governance | Resilience, legitimacy | Slower decisions; possible deadlock |
| Formal process (proposals, votes) | Transparency, fairness | Overhead; discourages small changes if applied too widely |
| Informal process | Agility | Hidden rules favor insiders and are hard for newcomers to learn |
| Foundation home | Neutrality, legal support, trademark protection | Fees or requirements, loss of some control, slower change |
| Corporate sponsorship | Funding and full-time maintainers | Dependence; possible roadmap pressure; trust issues if strategy shifts |
| Strict moderation | Safe, productive spaces | Moderators carry emotional load; some people will call it censorship |
| Light moderation | Low effort | Toxic participants drive away the people you most want to keep |
| Open roadmap | Trust and coordination | Public promises become expectations |

## Failure Modes

**Unwritten rules.** Insiders know how decisions are made. Outsiders do not, and drift away. Prevention: write down roles, criteria, and the decision process.

**Founder stagnation.** The founder holds all authority, stops having time, and nobody is empowered to act. The project stalls. Prevention: delegate rights early and plan succession.

**Governance theater.** An elaborate structure exists on paper while real decisions happen privately. Prevention: keep process proportionate to the project's size and make real decisions where the record is.

**Tyranny of the loudest.** Without process, the most persistent voices win. Prevention: use proposals, time-boxed comment periods, and explicit decision criteria.

**Unenforced code of conduct.** A document exists but reports are ignored or applied unevenly. Contributors lose trust and leave. Prevention: appoint responders, follow procedure, and publish enforcement outcomes where appropriate.

**Contributor burnout and entitlement.** Users demand work from volunteers. Maintainers answer until they break. Prevention: state expectations, point to paid support where it exists, and normalize saying no.

**Hostile fork or relicensing shock.** A sponsor changes license or direction, and the community splits. Prevention: understand who holds copyright and trademark, and consider neutral hosting or contributor agreements that reflect the project's values. Forking is the community's ultimate safeguard, and it is expensive for everyone.

**Capture.** One company or person gains disproportionate influence by supplying most of the maintainers. Prevention: cap any one affiliation's share of decision seats if the project needs to remain neutral.

**Succession failure.** A key maintainer leaves with no handoff, and credentials are lost. Prevention: shared access to infrastructure, documented procedures, and an explicit process for retiring maintainers into emeritus status.

## In Practice

### A minimal governance document

A small project can start with one page that states:

1. The project's purpose and scope.
2. The roles and how a person gains or loses them.
3. How decisions are made day to day, and how disputes are escalated.
4. How to propose significant changes.
5. The code of conduct and who enforces it.
6. Who owns the name, domain, repository organization, and registry accounts.
7. How governance itself can be changed.

### A maintainer nomination checklist

- [ ] Sustained contributions over a defined period.
- [ ] Reviews that show good judgment and communication.
- [ ] Understanding of the project's scope and standards.
- [ ] Time and willingness to take on the responsibility.
- [ ] Existing maintainers agree under the stated process.
- [ ] Access is granted with the least privilege needed and documented.

### Handling a conflict

1. Move from public threads to a calmer channel if emotions are high, and record the outcome.
2. Restate each side's position to confirm understanding.
3. Return to the project's written goals and scope.
4. If consensus fails, use the defined tie-breaker.
5. Document the decision and its reasoning.

### Related material

- Maintainer practices: [Maintaining a Project](./maintaining-a-project.md)
- Licensing and contributor agreements: [Licensing Explained](./licensing-explained.md)
- Communication skills that carry across work and open source: [Communication and Writing](../17-engineering-career/communication-and-writing.md)

## Further Reading

- Karl Fogel, *Producing Open Source Software*, chapters on social and political infrastructure.
- Nadia Eghbal, *Working in Public*. Discusses roles and the economics of attention in modern open source.
- Contributor Covenant, contributor-covenant.org. A widely adopted code of conduct template.
- Python Enhancement Proposals (PEP 1 describes the process) and the Rust RFC repository. Both document enhancement-proposal governance in practice.
- Elinor Ostrom, *Governing the Commons*. A foundational study of how communities manage shared resources, often cited in discussions of open source.
