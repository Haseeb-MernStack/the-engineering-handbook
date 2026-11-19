# Browser Fundamentals

> The browser is a sandboxed runtime with its own networking, security and execution model, and frontend work is only reliable when you understand how it behaves.

## Overview

Frameworks come and go. The browser underneath them changes slowly. Every framework compiles down to the same things: HTML parsed into a DOM, CSS applied to produce a render tree, and JavaScript run on a single main thread with an event loop. When a page is slow, janky, insecure or broken, the cause is almost always in that layer.

Engineers who understand the browser can debug problems the framework documentation does not mention. Developers who know only the framework are stuck when the abstraction leaks. In an age of generated UI code, this knowledge is what lets you judge whether the output is correct, fast and safe.

## Core Concepts

### From URL to pixels

When you navigate, roughly this happens:

1. Resolve the hostname (DNS), open a connection (TCP, then TLS, or QUIC), send an HTTP request.
2. Receive HTML and parse it incrementally into the DOM.
3. Discover subresources (CSS, scripts, images, fonts) and fetch them.
4. Build the CSSOM from stylesheets and combine with the DOM into a render tree.
5. Compute layout (sizes and positions), paint, and composite layers on screen.

Some resources block rendering. CSS blocks rendering because the browser will not paint unstyled content. Synchronous scripts block HTML parsing because they may change the document. `defer` and `async` change this: `defer` runs scripts after parsing in order, `async` runs them as soon as they arrive in any order.

### The event loop and the main thread

JavaScript runs on one main thread, shared with layout, painting and input handling. The event loop takes tasks from a queue, runs each to completion, and between tasks the browser may render. Microtasks (promise callbacks) run after the current task and before rendering.

Consequence: any long task blocks everything. A 300 ms loop means 300 ms of ignored clicks and frozen scrolling. Asynchrony (`await`, timers, fetch) lets the thread do other work while waiting for I/O, but it does not make CPU work parallel. Use Web Workers for heavy computation.

```js
// Blocks the main thread: UI frozen until done.
for (const item of hugeArray) expensive(item);

// Yields between chunks so input and rendering can proceed.
async function processInChunks(items, size = 200) {
  for (let i = 0; i < items.length; i += size) {
    items.slice(i, i + size).forEach(expensive);
    await new Promise((r) => setTimeout(r, 0));
  }
}
```

### The DOM and its cost

The DOM is a tree of live objects. Reading layout properties (such as `offsetHeight`) after writing styles can force a synchronous layout. Alternating reads and writes in a loop causes repeated forced layouts. Batch reads together, then writes.

Large DOMs make style calculation, layout and memory usage worse. Event delegation (one listener on a parent) scales better than a listener per element.

### HTTP caching

The browser cache is the cheapest performance tool you have, and one of the easiest to misconfigure. Key headers:

| Header | Purpose |
|---|---|
| `Cache-Control: max-age=N` | Fresh for N seconds without revalidating |
| `Cache-Control: no-cache` | May store, but must revalidate before use |
| `Cache-Control: no-store` | Do not store (sensitive responses) |
| `ETag` / `Last-Modified` | Validators enabling conditional requests (304) |
| `Vary` | Which request headers affect the cached response |

A common strategy: give static assets content-hashed filenames and a long max-age (they never change under the same name), and serve the HTML document with short or revalidated caching so new deploys are picked up. See [Caching in Services](../07-backend-engineering/caching-in-services.md) for the server side.

### Storage options

| Mechanism | Capacity and nature | Notes |
|---|---|---|
| Cookies | Small, sent with every matching request | Server-readable. Use `HttpOnly`, `Secure`, `SameSite` for session cookies |
| `localStorage` | Synchronous, string key-value, persists | Blocks main thread, readable by any script on the origin |
| `sessionStorage` | Same API, per tab | Cleared when the tab closes |
| IndexedDB | Asynchronous structured storage | Larger and more complex |
| Cache API | Request/response pairs | Used with service workers |

Anything stored in script-accessible storage can be stolen by an XSS bug. Do not put long-lived secrets there.

### Same-origin policy, CORS and CSP

An origin is scheme, host and port. The same-origin policy stops scripts from one origin reading data from another. It is the browser's central security boundary.

CORS (Cross-Origin Resource Sharing) is a controlled relaxation. The server declares which origins may read its responses through headers such as `Access-Control-Allow-Origin`. Cross-origin requests with custom headers or non-simple methods trigger a preflight `OPTIONS` request. CORS is enforced by the browser to protect users. It does not protect your server from non-browser clients, and it is not authentication.

Content Security Policy (CSP) is a response header that restricts where scripts, styles and other resources may load from, limiting the damage of injected content.

### Security-relevant behavior of cookies

`SameSite` controls whether cookies are sent on cross-site requests, which mitigates cross-site request forgery. `HttpOnly` hides the cookie from JavaScript. `Secure` limits it to HTTPS. See [Authentication and Authorization](../07-backend-engineering/authentication-and-authorization.md).

### Service workers

A service worker is a script that sits between the page and the network, enabling offline support, caching strategies and background features. It has its own lifecycle and caching rules, and a buggy one can serve stale content to users until it is replaced.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Long cache lifetimes | Fewer requests, faster repeat visits | Stale content unless filenames change with content |
| `localStorage` for state | Simple, persistent | Synchronous, XSS-readable, no expiry |
| Service worker | Offline, control of caching | Lifecycle complexity, hard-to-debug stale state |
| Strict CSP | Strong XSS mitigation | Breaks inline scripts and third-party tags, needs maintenance |
| Many third-party scripts | Features without building them | Performance, privacy and security exposure |
| Relying on newest browser APIs | Capability | Compatibility gaps, need for fallbacks |

## Failure Modes

- **Stale assets after deploy.** HTML cached too long references removed files, or an old bundle talks to a new API. Use hashed filenames, keep old assets briefly, and watch for 404s on assets.
- **Main-thread blocking.** A long task freezes the page. Detect with performance profiling and long-task monitoring.
- **CORS misconfiguration.** Reflecting any `Origin` with credentials allowed lets other sites read authenticated responses. Maintain an explicit allowlist.
- **Token in `localStorage` stolen via XSS.** Prefer `HttpOnly` cookies for sessions where appropriate, and reduce XSS surface.
- **Memory leaks.** Listeners, timers and closures retained after components are gone. Symptom is steadily growing memory in long sessions. Detect with heap snapshots.
- **Layout thrashing.** Interleaved DOM reads and writes. Detect in the performance panel as repeated forced reflows.
- **Mixed content and missing HTTPS.** Blocked or insecure subresources. Enforce HTTPS and HSTS.
- **Service worker serving old app forever.** Version it, and provide an update path.

## In Practice

### Debugging habits

1. Open the network panel first. What was requested, how long, from where (cache or network)?
2. Check the console for errors and CSP or CORS messages.
3. Use the performance panel to find long tasks and forced layouts.
4. Reproduce on a throttled network and a slower CPU. Your laptop is not your user.
5. Test in more than one browser engine.

### Heuristics

- Ship less JavaScript. Every byte of script costs download, parse and execute time.
- Let the browser do work natively before reimplementing it (forms, links, `<dialog>`, lazy loading).
- Treat all data from outside your code as untrusted when inserting into the DOM.

### Developer vs. engineer

The developer fixes a CORS error by setting the allowed origin to `*`. The engineer reads why the browser blocked it, decides which origins truly need access and whether credentials are involved, and sets the narrowest policy that works.

### Verifying AI-generated code

Generated frontend code often uses `innerHTML` with unsanitized data, stores tokens in `localStorage`, sets permissive CORS headers to "make the error go away", and ignores cache headers. Check each of these deliberately.

## Further Reading

- MDN Web Docs, sections on HTTP caching, CORS, the event loop and Content Security Policy.
- Ilya Grigorik, *High Performance Browser Networking*.
- Jake Archibald, "Tasks, microtasks, queues and schedules."
- Tali Garsiel and Paul Irish, "How Browsers Work."
- RFC 9111, HTTP Caching.
- RFC 6265, HTTP State Management Mechanism (cookies).
