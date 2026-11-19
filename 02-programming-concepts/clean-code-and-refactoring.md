# Clean Code and Refactoring

> Clean code is code that other people can understand and change safely; refactoring is the practice of improving its structure in small, behavior-preserving steps.

## Overview

Code is read far more often than it is written. Every feature, bug fix and review begins with someone trying to understand existing code. The cost of unclear code is paid again each time, by each reader, usually under time pressure.

"Clean" is not about taste or prettiness. It is a practical property: how quickly a competent teammate can answer "what does this do, and what happens if I change it?" Clear naming, small units with single purposes, honest abstractions and good tests all serve that goal.

Refactoring is how you get there without stopping work. Martin Fowler defines it as changing the internal structure of software without changing its observable behavior. The key constraint is behavior preservation, which you can only claim if you have tests or another way to check.

Developers often experience this topic as style rules. Engineers treat it as risk management: unclear code hides bugs and makes change expensive. The advice here is a set of heuristics, not laws. Where experienced engineers disagree, that is noted.

## Core Concepts

### Naming

Names are the main documentation. A good name says what something is or does in the language of the domain.

```python
# Weak
def proc(d, f):
    return [x for x in d if x[2] > f]

# Clearer
def orders_above_total(orders, minimum_total):
    return [o for o in orders if o.total > minimum_total]
```

Guidelines:

- Prefer specific names over generic ones (`data`, `info`, `manager`, `util`).
- Name booleans as questions (`is_active`, `has_access`).
- Use one word per concept across the codebase.
- Longer names are fine for wider scopes; short names suit tiny scopes.
- Avoid abbreviations that your teammates would need to decode.

### Functions and units

A function should do one thing at one level of abstraction. Signs it does too much: a long body, many parameters, flags that switch behavior, comments dividing it into sections, names containing "and".

Short functions are not an end in themselves. Splitting code into many tiny pieces can scatter the logic and make it harder to follow. The test is whether each piece has a clear name and purpose.

### Comments

Comments should explain why, not restate what. If code needs a comment to explain what it does, try clearer code first. Good comments record intent, constraints, surprising decisions and links to context. Bad comments go stale and lie.

```python
# Retry once: the upstream returns 503 during its nightly rotation (see incident notes).
```

### Duplication and the right abstraction

Duplicated knowledge means a future change must be made in many places and one will be missed. But duplicated text is not always duplicated knowledge. Two pieces of code that look the same today but change for different reasons should stay separate. A wrong abstraction is costlier than a little repetition; Sandi Metz has argued for waiting until the right shape is clear. A common heuristic is to tolerate duplication until you see it a third time.

### Coupling and cohesion

- **Cohesion**: how closely a module's parts belong together. Aim high.
- **Coupling**: how much modules depend on each other's internals. Aim low.

Highly cohesive, loosely coupled modules can be understood and changed independently. See [Architecture Principles](../04-software-architecture/architecture-principles.md) for the same ideas at system scale.

### SOLID and other principles

SOLID is a set of object-oriented design guidelines: single responsibility, open-closed, Liskov substitution, interface segregation and dependency inversion. They are useful lenses, not mandates. Applied mechanically they produce needless interfaces and indirection. Other durable principles: keep it simple, do not build for hypothetical future needs (YAGNI), and make the common case easy.

### Code smells

A smell is a surface sign of a possible deeper problem. Examples: long function, large class, long parameter list, feature envy (a method uses another object's data more than its own), shotgun surgery (one change requires edits in many places), primitive obsession, deeply nested conditionals, dead code, magic numbers.

### Refactoring in small steps

Common behavior-preserving moves:

- Rename a variable, function or class.
- Extract a function from a block.
- Inline a needless function.
- Replace a magic number with a named constant.
- Replace nested conditionals with guard clauses.
- Move behavior to the class that owns the data.
- Introduce a parameter object.

Guard clauses example:

```python
# Before
def ship(order):
    if order is not None:
        if order.paid:
            if order.items:
                return dispatch(order)
    return None

# After
def ship(order):
    if order is None or not order.paid or not order.items:
        return None
    return dispatch(order)
```

### Tests as the safety net

Refactoring without tests is editing and hoping. Before changing messy code, add characterization tests that capture what it currently does, even if the behavior seems wrong. Then refactor while the tests stay green. See [Testing and Quality](../11-testing-quality/README.md).

## Trade-offs

| Practice | Benefit | Cost |
| --- | --- | --- |
| Extensive refactoring | Lower long-term cost of change | Time now, risk of regressions without tests |
| Small functions | Named, testable pieces | Indirection, jumping between files |
| Abstraction | Flexibility, reuse | Harder to follow; wrong abstractions are expensive to undo |
| Following a style guide | Consistency, less debate | Rules can be applied without thought |
| Comments | Capture intent | Maintenance burden, staleness |
| Leaving working ugly code | Zero risk now | Interest accrues as technical debt |

Technical debt is a useful metaphor with limits. Taking on debt deliberately to meet a date can be correct if you record it and plan to repay it. Unrecorded debt becomes a hidden tax.

"Clean" is also contextual. A throwaway script, a prototype to test a market idea and a payment system have different needs. Over-polishing the first is waste; under-polishing the last is negligence.

## Failure Modes

- **The big-bang rewrite.** A team halts feature work to rewrite a system "properly". The rewrite takes longer than planned, misses undocumented behavior the old system had, and the old system keeps changing underneath. Prefer incremental replacement: wrap the old code, move one slice at a time and keep shipping. Joel Spolsky's essay on rewrites is a well-known critique; the pattern commonly called the strangler fig describes the alternative.
- **Refactoring without tests.** Behavior changes unnoticed and bugs appear in production. Add tests first or use very small, mechanical, tool-assisted steps.
- **Mixing refactoring with behavior change.** One large change that both restructures and alters logic cannot be reviewed or bisected. Keep them in separate commits or pull requests.
- **Premature abstraction.** A generic framework built for one use case, then bent for every new case. Detect it by parameters and flags that exist for one caller only. Inline and simplify.
- **Dogmatic rule following.** Splitting every function to five lines, or adding an interface for each class, because a rule said so. Ask what problem the rule solves here.
- **Stale comments and names.** A function renamed in behavior but not in name. Review names and comments when code changes.
- **Dead code and flag graveyards.** Unused branches and old feature flags that nobody dares remove. Delete them; version control remembers.
- **Cleanup as a never-ending project.** Refactoring without a goal. Tie it to upcoming work: tidy the area you are about to change.
- **Style fights.** Review time spent on formatting. Automate with a formatter and linter in the pipeline.

## In Practice

### The boy scout approach

Leave code slightly better than you found it. When you touch a function to fix a bug or add a feature, make it clearer first (a rename, an extraction), then make the change. Kent Beck's phrase for this is to make the change easy, then make the easy change.

### Refactoring workflow

1. Confirm there are tests covering the behavior; add characterization tests if not.
2. Make one small structural change.
3. Run the tests.
4. Commit.
5. Repeat. If tests fail, revert the last step instead of debugging a large change.

### Code review heuristics

- Can I explain what this does from the names alone?
- What would surprise the next reader?
- Is the abstraction justified by present needs?
- Is there duplicated knowledge?
- Are error cases and edge cases visible in the code?
- Is the diff one thing, or several?

### Worked example: extract and name

```python
# Before
def invoice_total(items, customer):
    t = 0
    for i in items:
        t += i.price * i.qty
    if customer.tier == "gold":
        t = t * 0.9
    t = t + t * 0.2
    return t

# After
TAX_RATE = 0.20
GOLD_DISCOUNT = 0.10

def subtotal(items):
    return sum(i.price * i.qty for i in items)

def apply_discount(amount, customer):
    return amount * (1 - GOLD_DISCOUNT) if customer.tier == "gold" else amount

def invoice_total(items, customer):
    discounted = apply_discount(subtotal(items), customer)
    return discounted * (1 + TAX_RATE)
```

The behavior is identical, and each rule now has a name and a single place to change. A real system would also avoid floating-point for money; see [Type Systems](./type-systems.md).

### Developer vs. engineer

A developer makes it work and moves on. An engineer makes it work, makes it understandable, and leaves evidence (tests, names, notes) so the next person can change it with confidence.

### Verifying AI-generated code

Generated code is often fluent and plausible, which makes unclear logic look clean. Check that names reflect domain meaning rather than generic filler, that it matches existing conventions, that it has not duplicated helpers that already exist in your codebase, and that it does not introduce abstractions you did not ask for. Read it as you would a colleague's pull request, and insist on tests before accepting a refactor it proposes.

## Further Reading

- Martin Fowler, *Refactoring: Improving the Design of Existing Code*.
- Robert C. Martin, *Clean Code: A Handbook of Agile Software Craftsmanship* (influential; some guidance is debated, so apply it with judgment).
- John Ousterhout, *A Philosophy of Software Design*.
- Michael Feathers, *Working Effectively with Legacy Code*.
- Kent Beck, *Tidy First?* and *Test-Driven Development: By Example*.
- Steve McConnell, *Code Complete*.

Previous: [Error Handling](./error-handling.md). Back to the [section overview](./README.md).
