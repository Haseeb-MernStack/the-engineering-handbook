# State Management

> State management is the discipline of deciding where each piece of data lives, who may change it, and how the UI stays consistent with it.

## Overview

A user interface is a function of data over time. Most frontend bugs are state bugs: a list that does not update, two components that disagree, a spinner that never stops, a form that loses input. The tools differ, but the underlying questions are the same. What is the source of truth? Who can change it? How do changes propagate?

Libraries make state easier to manipulate but do not decide the design. Teams that reach for a global store by default end up with everything coupled to everything. Teams that never plan end up with data duplicated across ten components. The skill is choosing the simplest placement that is still correct.

## Core Concepts

### Kinds of state

Different state has different needs. Mixing them is the root of much complexity.

| Kind | Example | Where it usually belongs |
|---|---|---|
| Local UI state | Is this dropdown open? | Inside the component |
| Shared UI state | Selected tab used by two siblings | Nearest common parent |
| Server state | A list of orders fetched from the API | A data-fetching and caching layer |
| URL state | Current page, filters, search query | The URL |
| Form state | Field values, validation errors, dirty flags | A form library or local state |
| Persistent client state | Theme, saved preferences | Browser storage, synced to state |
| Global app state | Current user, feature flags | A small global store or context |

### Server state is not client state

Data fetched from a server is a cache of something you do not own. It can be stale, loading, failed or being refetched. Treating it as ordinary local state means hand-writing loading flags, error handling, deduplication, refetching and invalidation over and over.

Dedicated server-state tools handle this: caching by key, background refetching, request deduplication, retries and invalidation after mutations. Using one removes a large category of manual state.

### Single source of truth and derived state

Store the minimum. Compute everything else.

```ts
// Duplicated state: can drift out of sync.
const [items, setItems] = useState<Item[]>([]);
const [count, setCount] = useState(0);

// Derived: always consistent.
const [items, setItems] = useState<Item[]>([]);
const count = items.length;
```

When two pieces of state must change together, ask if one is derivable from the other. If so, store only one.

### Unidirectional data flow

A common pattern: state flows down to components, events flow up as actions, and state changes in one place.

```
 user event --> action --> update state --> render --> UI
      ^                                                 |
      +-------------------------------------------------+
```

This makes changes traceable. You can answer "what changed this value?" by finding the one place that updates it.

### Immutability

Many UI libraries detect changes by comparing references. Mutating an object in place can leave the UI stale because the reference did not change. Create new objects for updates.

```ts
// Mutation: may not trigger a re-render.
state.todos[0].done = true;

// Immutable update.
setState({ ...state, todos: state.todos.map((t, i) => i === 0 ? { ...t, done: true } : t) });
```

### State machines

When state has distinct modes with rules about transitions, model them explicitly. Instead of several booleans (`isLoading`, `isError`, `isSuccess`) that allow impossible combinations, use one status.

```ts
type Fetch<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "error"; error: string }
  | { status: "success"; data: T };
```

Making illegal states unrepresentable removes whole bug classes.

### Optimistic updates

To feel instant, the UI can apply a change before the server confirms it, then roll back on failure. It requires keeping the previous value, handling the error path, and reconciling with the server's response. It suits low-conflict actions like toggling a like and is risky for operations with real consequences like payments.

### URL as state

Filters, pagination and selected items that users might share or bookmark belong in the URL. This gives free deep links, back-button behavior and refresh safety.

### Forms

Forms are state with validation. Decide whether inputs are controlled (React-style, value in state) or uncontrolled (DOM holds the value). Validate on the client for fast feedback, and always validate again on the server. See [API Design](../07-backend-engineering/api-design.md).

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Global store | One place to look, easy cross-component sharing | Coupling, boilerplate, everything re-evaluates on change if poorly scoped |
| Local state | Simple, isolated | Prop drilling or lifting when sharing is needed |
| Context for state | Built in, no library | Consumers may re-render broadly when the value changes |
| Server-state library | Less code, correct caching behavior | Another concept to learn, cache-key discipline |
| Normalized store | No duplicate entities, consistent updates | Indirection, more code to read and write |
| Optimistic updates | Responsive feel | Rollback complexity, momentary lies to the user |
| URL state | Shareable, restorable | Serialization, length limits, privacy of what is put in URLs |

## Failure Modes

- **Stale data.** The UI shows old values after a mutation because related cache entries were not invalidated. Define what each mutation invalidates.
- **Duplicated truth.** The same entity is stored in two places and they diverge. Normalize or derive.
- **Race conditions.** A slow earlier request returns after a later one and overwrites newer data. Cancel superseded requests or ignore out-of-order responses.
- **Impossible states.** Loading and error shown together. Use a status union.
- **Over-global state.** Everything lives in the store, so any change touches many components and tests become hard.
- **Unnecessary re-renders.** Frequently changing state in a high-level context re-renders a whole subtree. Split state, scope subscriptions.
- **Lost input.** A refetch or re-mount resets a form. Keep form state stable and key components carefully.
- **Secrets or personal data in persisted state.** Local storage retains them after logout. Clear on sign-out.
- **Optimistic update never rolled back.** The user believes an action succeeded when it failed.

Detect these by testing state transitions, throttling the network, and simulating failure responses. See [Frontend Testing and Security](./frontend-testing-and-security.md).

## In Practice

### Placement checklist

For each piece of state ask:

1. Who needs to read it? Who needs to change it?
2. Is it derivable from something else?
3. Is it really server data? If so, use the data layer.
4. Should it survive a refresh or be shareable? Then URL or storage.
5. What are its valid states and transitions?
6. What happens on failure, and on a slow or out-of-order response?

### Heuristic

Start with state as local as possible. Lift it only when something else needs it. Reach for a global store when real cross-cutting data, such as the current user, justifies it.

### Worked example

A product list with filters. Filters go in the URL so the page is shareable. The list is server state, keyed by the filters, so changing a filter fetches or reads from cache. The "quick view" panel's open flag is local UI state. Selected rows for bulk action are shared UI state in the list's parent. No global store is needed.

### Developer vs. engineer

The developer adds a global variable when two components need to share something. The engineer asks what kind of state it is, where it should live, and what happens when it is stale.

### Verifying AI-generated code

Generated code often copies server data into local state and then edits the copy, uses many booleans for status, stores derived values, forgets request cancellation, and puts everything in one big store. Check for each.

## Further Reading

- Official documentation of your framework on state, effects and data fetching.
- Kent C. Dodds, "Application State Management with React."
- Dan Abramov, "You Might Not Need Redux."
- David Khourshid, writing on statecharts and XState; and David Harel, "Statecharts: A Visual Formalism for Complex Systems" (1987).
- Official documentation of TanStack Query or a comparable server-state library.
