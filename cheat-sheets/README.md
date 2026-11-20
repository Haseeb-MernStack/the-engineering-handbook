# Cheat Sheets and Decision Frameworks

Short pages for decisions engineers make repeatedly. Each page states when the decision applies, gives a way to decide, and says what each option costs. They are meant to be used in design reviews, not read once.

These pages summarize. The linked sections explain the reasoning in depth.

## How each page is built

1. **Use when**: the situation the decision applies to.
2. **Questions first**: what you must know before choosing.
3. **Comparison**: options side by side, with costs.
4. **Decision guide**: a short path from answers to a default.
5. **When the default is wrong**: the conditions that should change your mind.
6. **Review checklist**: what to confirm before you commit.
7. **Go deeper**: links into the handbook.

A default is a starting point under stated assumptions. It is not a rule.

## Index

| Cheat sheet | Decision | Related sections |
| --- | --- | --- |
| [Choosing a Database](./choosing-a-database.md) | Relational, document, key-value, wide-column, search or graph | [06 Database Engineering](../06-database-engineering/README.md) |

More pages are planned for architecture style, API style, consistency level, caching strategy, queue versus stream, deployment strategy, test strategy, and observability signals.

## Contributing a cheat sheet

Follow the structure above. Keep each page to one decision and about one to two screens. State assumptions, label examples as examples, and avoid unsupported numbers. See [CONTRIBUTING.md](../CONTRIBUTING.md).

Back to [The Engineering Handbook](../README.md).
