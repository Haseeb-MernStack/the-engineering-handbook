# Contributing to The Engineering Handbook

Thank you for helping make this handbook better. The goal is a reference that working engineers can trust, so contributions are held to a deliberately high standard of accuracy and clarity.

## What we welcome

- Corrections to factual, technical, or conceptual errors
- Clearer explanations of existing material
- Missing trade-offs, failure modes, or real-world context
- New chapters that fit the 18-section structure
- Better examples, diagrams, and checklists
- Updated or additional primary references
- Typo, grammar, and formatting fixes

## What we do not accept

- Marketing, product promotion, or affiliate links
- Unreviewed AI-generated text submitted as-is. AI assistance is fine, but the contributor is responsible for verifying every claim
- Content copied from books, courses, or articles without permission
- Tool-specific tutorials with no underlying principle
- Opinions presented as universal facts

## Quality bar

Every contribution should be:

1. **Accurate.** Claims are correct and verifiable. Cite primary sources (specifications, papers, official docs) where possible.
2. **Precise.** Use exact terminology. Avoid vague words such as "fast", "scalable", or "robust" without context or numbers.
3. **Trade-off aware.** Explain what a choice costs, not only what it provides.
4. **Practical.** Include how it behaves in production, not only in theory.
5. **Concise.** Say it once, clearly. Prefer short sentences and concrete examples.
6. **Original.** Written in your own words.

## Chapter template

New topic files should follow this structure:

```markdown
# Topic Title

> One-sentence summary of what this topic is and why it matters.

## Overview
What it is, and where it fits in the larger picture.

## Core Concepts
The ideas the reader must understand, explained precisely.

## Trade-offs
Options, what each costs, and when each is appropriate.

## Failure Modes
How this breaks in production and how to detect or prevent it.

## In Practice
Examples, checklists, or decision frameworks.

## Further Reading
- [Title](url): why it is worth reading
```

## File and naming conventions

- Section folders keep their numeric prefix (`05-system-design`).
- Topic files use lowercase kebab-case (`consistent-hashing.md`).
- Each section folder contains a `README.md` that indexes its topics.
- Images go in an `assets/` folder inside the relevant section.
- Use relative links between files.
- Use fenced code blocks with a language tag.

## Workflow

1. **Open an issue first** for new chapters or large changes, so we can agree on scope.
2. **Fork** the repository and create a branch: `docs/short-description` or `fix/short-description`.
3. **Make your changes** following the quality bar and template above.
4. **Self-review.** Re-read your change as a skeptical reader. Check links, spelling, and claims.
5. **Open a pull request** with a clear description of what changed and why.
6. **Respond to review.** Maintainers may request changes or sources.

## Commit messages

Use short, imperative messages with a type prefix:

```text
docs(system-design): add section on consistent hashing
fix(security): correct description of CSRF token handling
chore: fix broken links in README
```

## Pull request checklist

- [ ] The change is accurate and sourced where needed
- [ ] It follows the chapter template and naming conventions
- [ ] Links and code examples work
- [ ] Spelling and grammar are checked
- [ ] I wrote or fully verified every line, including any AI-assisted text
- [ ] I have the right to contribute this content under the project license

## Code of conduct

Be respectful, assume good intent, and critique ideas rather than people. Harassment, discrimination, or hostile behavior will not be tolerated, and maintainers may remove contributions or contributors that violate this standard.

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
