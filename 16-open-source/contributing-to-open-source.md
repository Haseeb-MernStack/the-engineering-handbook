# Contributing to Open Source

> Contributing to open source is the fastest way to practice engineering in a codebase you did not write, under review by people who owe you nothing, and that is exactly why it builds real skill.

## Overview

Most professional work happens inside code that already exists. You read it, infer the intent behind it, change it without breaking it, and convince someone else the change is right. Open source lets you practice all of that in public, on real systems, with real maintainers.

Contribution is not only code. Documentation fixes, bug reproductions, triage, translations, and review all count. Maintainers are usually short of attention, not of ideas. The most valuable contributors reduce the maintainer's workload instead of adding to it.

This matters more in the age of AI. A tool can generate a patch in seconds. It cannot tell you whether the project wants that patch, whether it matches the conventions of the codebase, or whether the maintainers have capacity to review it. Those judgments are the engineering part. A contribution you cannot explain and defend is a liability, however it was produced.

## Core Concepts

### Read before you write

Every healthy project publishes its rules. Before you touch code, find and read:

- `README` for what the project is and is not.
- `CONTRIBUTING` for process, branching, and review expectations.
- `CODE_OF_CONDUCT` for behavior expected in all project spaces.
- The issue tracker, for existing discussion of your idea.
- Recent merged pull requests, to see what a typical accepted change looks like.
- The license, to know the terms your contribution will carry.

Some projects require a Developer Certificate of Origin sign-off (a `Signed-off-by` line in each commit) or a Contributor License Agreement. These are legal statements about your right to contribute the code. Read them and do not sign what you do not understand.

### Choose the right first contribution

A good first contribution is small, scoped, and clearly wanted. Reasonable starting points:

| Starting point | Why it works |
|---|---|
| Fix a typo or unclear passage in docs | Teaches the workflow with almost no risk |
| Reproduce and clarify a bug report | Maintainers value a minimal reproduction highly |
| Issues labeled for newcomers | The maintainers have already agreed the work is wanted |
| Add a missing test for existing behavior | Teaches the code and improves safety |
| Fix a bug you hit yourself in real use | You understand the problem and have motivation to finish |

Avoid starting with a large refactor, a new feature nobody requested, or a style overhaul. These cost the maintainer far more than they give.

### The contribution loop

1. Find or open an issue. For anything beyond a trivial fix, discuss first.
2. Say you intend to work on it, so effort is not duplicated.
3. Fork, branch, and build the project locally. Run the existing tests before you change anything.
4. Make the smallest change that solves the problem.
5. Add or update tests and documentation.
6. Open a pull request that explains what, why, and how you verified it.
7. Respond to review promptly and without defensiveness.
8. Accept that the change may be reshaped, delayed, or declined.

### Anatomy of a good pull request

A reviewer should understand your change within a minute. Include:

- A title that states the change, not the activity.
- A description of the problem and why this approach was chosen.
- A link to the issue it addresses.
- How you tested it, including commands.
- Anything you are unsure about, stated openly.

Keep each pull request to one logical change. A reviewer can approve a 40-line change in one sitting. A 1,500-line change tends to wait, or to be skimmed.

### A contribution checklist

- [ ] I read the contributing guide and code of conduct.
- [ ] An issue exists, and a maintainer has signaled the work is welcome.
- [ ] The project builds and its tests pass on my machine before my change.
- [ ] My change follows the existing style, naming, and structure.
- [ ] I added tests that fail without my change and pass with it.
- [ ] I updated docs and changelog entries if the project uses them.
- [ ] My commits are clean and follow the project's message convention.
- [ ] I can explain every line of the diff.

The last item is not optional. If you used an AI assistant to draft code, you remain the author of record. You must understand it, verify it, and be able to defend it in review.

### Giving and receiving review

Review is a conversation about the code, not about you. When you receive feedback, ask a clarifying question if you disagree, and offer evidence instead of opinion. When a maintainer says no, the decision is theirs. Projects belong to the people who carry their long-term cost.

When you review others, be specific, explain the reason behind a request, and separate required changes from suggestions.

## Trade-offs

| Choice | What it gives you | What it costs |
|---|---|---|
| Discuss before coding | Avoids wasted work and rejected designs | Slower start; some maintainers respond slowly or not at all |
| Small, narrow pull requests | Fast review, low risk | More overhead per change; progress feels slow |
| Large, ambitious pull request | Can deliver real value in one step | Heavy review burden; high chance of rejection or abandonment |
| Contributing to a big, famous project | Visibility, strong review culture | Intense competition for easy tasks; slow reviews; strict standards |
| Contributing to a small project | Fast feedback, real influence | Fewer mentors, thinner documentation, risk the project goes dormant |
| Fixing what you personally use | High motivation, deep context | Your needs may not match the project's direction |
| Using AI to draft changes | Speed in exploring and boilerplate | Risk of plausible but wrong or unwanted code; some projects restrict or ban it |

Time is the underlying cost. A first merged change in an unfamiliar codebase often takes longer than the code itself suggests, because most of the effort is learning the build, the conventions, and the people.

## Failure Modes

**The drive-by feature.** A contributor opens a large pull request implementing something nobody asked for. The maintainer must either reject hours of work or accept code they now own forever. Prevention: open an issue first and wait for agreement.

**The unreviewable pull request.** Unrelated changes, reformatting, and renames are bundled together. Review stalls. Prevention: one logical change per pull request, with formatting changes separated.

**Low-effort volume.** Submitting many trivial or machine-generated changes to collect contribution credit burdens maintainers and damages trust. Several projects have publicly complained about this pattern and tightened their policies. Prevention: contribute because the change helps the project, and expect to be judged on quality, not count.

**Unverified generated code.** A patch compiles but misunderstands the codebase, duplicates an existing helper, or ignores a convention. Prevention: run it, read it, and test it as if a stranger wrote it.

**Defensiveness in review.** The contributor argues taste or takes feedback personally. The pull request dies in a thread. Prevention: treat each comment as information about what the next reader will misunderstand.

**Abandonment.** The contributor disappears after the first review round. The pull request rots and the issue stays claimed. Prevention: claim only what you will finish, and say so if your plans change.

**Licensing mistakes.** Code is copied from another project with an incompatible license, or from a source you cannot identify. Prevention: write your own code, and see [Licensing Explained](./licensing-explained.md).

**Ignoring project norms.** Pinging maintainers repeatedly, demanding timelines, or posting in the wrong channel. Prevention: remember that most maintainers are volunteers or have day jobs.

## In Practice

### A realistic first month

- Week 1: Pick one project you already use. Build it. Run its tests. Read the last twenty merged pull requests.
- Week 2: Find a small issue or documentation gap. Comment to confirm it is wanted.
- Week 3: Submit the change with tests and a clear description.
- Week 4: Respond to review, revise, and learn from the outcome whether merged or not.

### Writing a good bug report

A bug report is a contribution. Include the version, the environment, exact steps, expected behavior, actual behavior, and a minimal reproduction. A report that a maintainer can reproduce in five minutes is often worth more than a speculative fix.

### Building a track record

A history of small, correct, well-explained changes earns trust. Over time that trust turns into review rights, commit access, or maintainer status in some projects. It is also visible evidence to employers of how you work with others. For how to present this, see [Project Selection and Method](../18-real-world-projects/project-selection-and-method.md).

### Related material

- Understanding unfamiliar code: [Engineering Fundamentals](../01-engineering-fundamentals/README.md)
- Tooling used in contribution workflows: [Developer Tools](../15-developer-tools/README.md)
- Writing tests that maintainers will accept: [Testing and Quality](../11-testing-quality/README.md)
- The maintainer's side of the same conversation: [Maintaining a Project](./maintaining-a-project.md)

## Further Reading

- Karl Fogel, *Producing Open Source Software*. A practical guide to how open source projects work, including contributor dynamics.
- Nadia Eghbal, *Working in Public: The Making and Maintenance of Open Source Software*. An account of the economics and labor of maintainership.
- GitHub Docs, "Contributing to a project" and "About pull requests". Official documentation of the fork-and-pull-request workflow.
- Open Source Guides (opensource.guide). A set of practical guides published by GitHub on contributing and maintaining.
- Developer Certificate of Origin, published by the Linux Foundation at developercertificate.org.
