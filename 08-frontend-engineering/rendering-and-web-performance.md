# Rendering and Web Performance

> Web performance is how quickly a page loads, responds and stays stable as people actually experience it, and it is shaped by rendering strategy, payload size and main-thread work.

## Overview

Performance is a feature users feel and rarely mention. Slow pages lose people before they see content, and sluggish interactions make an app feel broken even when it is correct. Performance also varies enormously by device and network. A fast laptop on office Wi-Fi is the best case, not the typical case.

Good performance work starts with measurement. Guessing leads to optimizing the wrong thing. This topic covers the main user-centered metrics, the rendering strategies that determine them, and the techniques that move them.

## Core Concepts

### What to measure

Metrics that approximate user experience (Google's Core Web Vitals are a widely used set):

| Metric | Question it answers |
|---|---|
| Largest Contentful Paint (LCP) | When did the main content appear? |
| Interaction to Next Paint (INP) | How quickly does the page respond to input? |
| Cumulative Layout Shift (CLS) | Does content jump around while loading? |

Supporting metrics include Time to First Byte (server and network latency) and First Contentful Paint. Thresholds are published by the Web Vitals project and have been revised over time, so check the current guidance instead of memorizing numbers.

Two kinds of data matter:

- **Lab data** comes from controlled tests (Lighthouse, local profiling). It is repeatable and good for debugging.
- **Field data** comes from real users (real-user monitoring). It reflects true devices, networks and behavior. Optimize against field data and use lab data to diagnose.

Look at percentiles. The slowest quarter of users often has a very different experience from the median.

### Rendering strategies

| Strategy | How it works | Good for | Cost |
|---|---|---|---|
| Client-side rendering (CSR) | Browser downloads a JS bundle, which builds the UI | Highly interactive apps behind login | Blank page until JS runs, weak first load on slow devices |
| Server-side rendering (SSR) | Server returns HTML for each request | Content that is dynamic or personalized, SEO | Server cost and latency, hydration work |
| Static generation (SSG) | HTML built ahead of time and served from a CDN | Content that changes rarely | Rebuilds on change, limited personalization |
| Incremental or on-demand regeneration | Static pages refreshed in the background | Large mostly-static sites | Staleness window, more moving parts |
| Streaming and partial hydration | Send HTML in pieces, hydrate only interactive parts | Large pages with mixed needs | Framework complexity |

Hydration is the step where client JavaScript attaches behavior to server-rendered HTML. The page may look ready before it responds. That gap is a common source of poor interaction metrics.

There is no universally best choice. Pick per page based on how often content changes, how personalized it is, and how interactive it needs to be.

### The critical rendering path

The browser can only paint after it has HTML, the CSS that applies, and any blocking scripts. Shortening this path improves first paint:

- Inline or prioritize the small amount of CSS needed for above-the-fold content.
- Load non-critical scripts with `defer` or dynamically.
- Preload resources discovered late, such as the hero image or key font.
- Avoid long chains where one file reveals the next.

### Reducing what you ship

The fastest byte is the one never sent.

- **Code splitting** loads code only for the current route or feature.
- **Tree shaking** drops unused exports, when modules and dependencies are written to allow it.
- **Compression** (gzip or Brotli) on text assets.
- **Dependency audit.** Measure what each library adds. A date or utility library can dominate a bundle.
- **Modern image formats**, correct dimensions, responsive `srcset`, and `loading="lazy"` for offscreen images.

```html
<img src="hero-800.jpg"
     srcset="hero-400.jpg 400w, hero-800.jpg 800w, hero-1600.jpg 1600w"
     sizes="(max-width: 600px) 100vw, 800px"
     width="800" height="450" alt="Product on a desk">
```

Setting `width` and `height` lets the browser reserve space, which prevents layout shift.

### Fonts

Web fonts can delay text or cause it to jump when they load. Use `font-display` to control behavior, preload critical fonts, subset them, and choose fallback fonts with similar metrics.

### Reducing main-thread work

Poor responsiveness usually comes from JavaScript. Tactics:

- Break long tasks into smaller ones and yield to the browser.
- Debounce or throttle high-frequency handlers (scroll, resize, input).
- Avoid unnecessary re-renders. Memoize carefully and only after measuring.
- Virtualize long lists so only visible rows exist in the DOM.
- Move heavy computation to a Web Worker.
- Prefer CSS transitions on `transform` and `opacity`, which can run on the compositor without layout.

### Caching and delivery

Serve static assets from a CDN with long-lived, content-hashed URLs. Use HTTP/2 or HTTP/3 where available so many small requests are cheap. Cache API responses where correctness allows. See [Browser Fundamentals](./browser-fundamentals.md).

### Performance budgets

A budget sets a limit, for example on bundle size or a metric, and fails the build or alerts when it is exceeded. Without a budget, performance degrades one reasonable-looking change at a time.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| More server rendering | Faster first paint | More server work, hydration cost, caching complexity |
| Aggressive code splitting | Smaller initial load | More requests, possible loading waterfalls, spinners on navigation |
| Lazy loading everything | Less upfront work | Delay when the user needs it, layout shifts if not reserved |
| Prefetching | Instant-feeling navigation | Wasted bandwidth, especially on metered connections |
| Heavy memoization | Fewer re-renders | Complexity, stale-value bugs, memory use |
| Third-party tags | Analytics, ads, chat | Main-thread cost and loss of control, often large |
| Optimizing for slow devices | Wider reach | Engineering time, fewer features |

## Failure Modes

- **Optimizing without measuring.** Weeks go into a micro-optimization that moves no metric. Profile first.
- **Lab and field disagree.** The synthetic score is green while real users on mid-range phones suffer. Collect field data.
- **Performance regression creep.** Each release adds a little. Enforce budgets in CI.
- **Hydration mismatch.** Server and client render differently, causing errors or flicker. Keep initial render deterministic.
- **Loading waterfalls.** Component fetches data only after it mounts, which happens only after its parent's data returns. Fetch in parallel and start early.
- **Layout shift.** Images, ads or banners without reserved space push content. Set dimensions.
- **Oversized third-party scripts.** A tag manager quietly loads many more. Audit regularly.
- **Cache poisoning of performance.** Personalized content cached at the CDN and served to the wrong user. This is also a security issue. Vary or bypass cache for private responses.
- **Fast on cable, broken on mobile.** Tested only on a developer machine with a fast network.

## In Practice

### A measurement-first workflow

1. Collect field data for the key pages and identify the worst metric and the worst segment.
2. Reproduce in the lab with throttled CPU and network.
3. Use a trace to find the cause: slow server, late resource, large bundle or long task.
4. Change one thing, measure again.
5. Add a budget or alert so the gain stays.

### Heuristics

- Start with the biggest resource on the critical path, usually the hero image, the main bundle or the first API call.
- Do work once, earlier, or not at all: cache it, precompute it, or delete it.
- If a framework feature fixes a problem you have measured, use it. If you have not measured, do not add it.

### Developer vs. engineer

The developer says "it is fast on my machine". The engineer asks about the 75th percentile mobile user, sets a budget, and proves the fix with field data.

### Verifying AI-generated code

Generated UI tends to import whole libraries for one function, render large lists without virtualization, fetch inside effects in nested components, and omit image dimensions. Run a bundle analyzer and a profile on anything non-trivial. Check network waterfalls for sequential requests.

## Further Reading

- Addy Osmani, *Image Optimization* and writings on loading performance.
- Web Vitals documentation and guidance from the Chrome team on web.dev.
- Ilya Grigorik, *High Performance Browser Networking*.
- Steve Souders, *High Performance Web Sites*.
- MDN Web Docs, "Web performance" guides.
