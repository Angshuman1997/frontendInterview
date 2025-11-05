# How to profile a React app’s performance

A single-file, copy-paste ready guide you can drop into your repo.
Covers tools, step-by-step workflows, what to look for, example commands/code, and actionable fixes.

---

## Quick summary

1. Reproduce a slow scenario (manual steps or automated test).
2. Use **React DevTools Profiler** to measure renders and commit costs.
3. Use **Chrome DevTools Performance** to capture CPU, paint, and long tasks.
4. Use **Lighthouse / WebPageTest** for load/TTI and network bottlenecks.
5. Use **bundle analyzers** to find large bundles.
6. Use `performance.mark` / `performance.measure` or Web Vitals for custom metrics.
7. Fix the hotspots (memoize, split, lazy load, virtualize) and re-profile.

---

## Tools you should know

* **React DevTools Profiler** (browser extension) — measures renders, commit durations, re-renders per component. Best for CPU/render profiling.
* **Chrome DevTools → Performance** — records JS execution, paints, layout, long tasks, flame charts, stack traces.
* **Lighthouse (Chrome Audits)** — gives metrics: FCP, LCP, TTI, CLS; suggests optimizations.
* **WebPageTest / PSI** — external performance checks across networks and devices.
* **Webpack Bundle Analyzer / source-map-explorer** — find large modules in bundle.
* **React Profiler API** (`<Profiler onRender={...}>`) — capture programmatic profiling.
* **Performance API** (`performance.mark`, `performance.measure`) — measure custom code spans.
* **Web Vitals** — record real-user metrics (FCP, LCP, CLS, FID/INP).
* **Why Did You Render** (dev tool) — detect avoidable re-renders (dev only).

---

## Workflow — Profiling renders (UI slowness, janky frames)

### 1. Reproduce reliably

Create a small scenario or automated test path that shows the slowness (e.g., typing in a search box, heavy list filtering, route navigation).

### 2. Profile with React DevTools Profiler

* Open app in the browser, open **React DevTools**, switch to the **Profiler** tab.
* Click **Start profiling**, perform the interaction, then **Stop**.
* Inspect:

  * **Flamegraph / Ranked** view — which components took the most time during commits.
  * **Commit timeline** — long commits indicate work to reduce.
  * **“Why did this render?”** (if available) — helps find prop/state changes causing renders.
  * Look for components with high **render time** or frequent renders.

**Actionable:** find the top 2–3 slowest commits/components and trace back to the props/state that caused them.

---

### 3. Correlate with Chrome DevTools Performance

* Open `DevTools → Performance`.
* Start recording (Preserve log optional), perform the action, stop.
* Analyze:

  * **Main thread**: long tasks (>50ms) and where JS is spending time.
  * **Call tree / Bottom-up / Flame chart**: find hot JS functions (library code, React reconciler, your functions).
  * **Paint/Layout**: see if rendering or style/layout is dominating.

**Tip:** use **“Capture DevTools timeline”** and enable **Screenshots** to see visual jank.

---

## Workflow — Profiling load & network (initial load, code-splitting)

### 1. Run Lighthouse (Chrome Audits)

* Open DevTools → Lighthouse → Generate Report.
* Examine FCP, LCP, TTI, Speed Index and the recommendations (server response, unused JS, caching, images).

### 2. Use Network panel

* Check largest resources, long TTFB, blocking scripts, uncompressed assets.
* Identify big JS bundles and long parse/eval times.

### 3. Bundle analysis

* Run a bundle analyzer after build:

  * Webpack: `npx webpack-bundle-analyzer build/stats.json` (or use plugin)
  * CRA: `GENERATE_SOURCEMAP=true npm run build` then `npx source-map-explorer build/static/js/*.js`
* Look for large libraries, duplicate copies, or vendor bundles that can be lazy-loaded or replaced.

---

## Workflow — Real-user & field data

* Integrate **Web Vitals** or Google Analytics to collect FCP, LCP, INP:

  ```bash
  npm install web-vitals
  ```

  ```js
  // in app entry
  import { getCLS, getFID, getLCP } from 'web-vitals';
  getCLS(console.log); getFID(console.log); getLCP(console.log);
  ```
* Use these metrics to prioritize fixes that impact real users.

---

## Programmatic profiling (React Profiler API)

Wrap expensive subtrees to log render timings:

```jsx
import { Profiler } from 'react';

function onRenderCallback(id, phase, actualDuration, baseDuration, startTime, commitTime, interactions) {
  console.log({ id, phase, actualDuration, baseDuration });
}

<Profiler id="ExpensiveList" onRender={onRenderCallback}>
  <ExpensiveList items={items} />
</Profiler>
```

* `actualDuration` — time spent rendering this subtree during the commit (useful to log/select hotspots).

---

## Use `performance.mark` / `measure` for custom spans

```js
performance.mark('fetch-start');
await fetch(...);
performance.mark('fetch-end');
performance.measure('api-fetch', 'fetch-start', 'fetch-end');

const measure = performance.getEntriesByName('api-fetch')[0];
console.log(measure.duration);
```

* Use `PerformanceObserver` to capture measurements for analytics.

---

## Interpreting results — what to look for

### Render/commit problems

* **Long commit times**: React doing lots of reconciliation or heavy renders. Fix by reducing work in render, memoize, split components.
* **Many small commits** instead of fewer large ones: maybe state updates are spread; batch updates or group state.
* **Repeated renders of same component** with same props: fix prop identity (useMemo/useCallback), `React.memo`, or avoid passing new object literals.

### CPU/JS problems

* **Long JS tasks** in DevTools: locate the function and optimize or move to web worker.
* **Parse/eval time** for large bundles: split code, lazy load, or migrate heavy libs to CDN.

### Network/Load problems

* **Large initial bundle** → split routes, lazy load components, import things dynamically.
* **Blocking third-party scripts** → async/defer or remove.

### Painting/Layout problems

* **Frequent layout thrash** (layout + paint): minimize forced synchronous layout reads, avoid heavy style recalculations, use transforms for animations.

---

## Typical fixes & where they come from profiling

* **Component-level render hotspots**

  * Use `React.memo` for pure components.
  * Stabilize props: `useMemo` for objects/arrays, `useCallback` for functions.
  * Move state lower (split component).
  * Avoid expensive computations in render; move to `useMemo` or worker.

* **Large lists**

  * Virtualize (react-window).
  * Window the dataset and lazy-load pages.

* **Expensive initial JS**

  * Code-split routes with `React.lazy` + `Suspense`.
  * Tree-shake and remove unused imports.
  * Use lighter-weight libraries / replace moment -> date-fns, lodash -> lodash-es selectively.

* **Network**

  * Compress assets, enable gzip/br, use HTTP/2, CDN, cache headers.
  * Defer non-critical scripts.

* **Animation/jank**

  * Use `requestAnimationFrame` and CSS transforms (GPU).
  * Use `useTransition` for non-urgent updates; keep input updates synchronous.

---

## Quick checklist to run now

* [ ] Reproduce slow UX reliably.
* [ ] Record React Profiler during that scenario. Identify top 3 expensive components.
* [ ] Record Chrome Performance, find long tasks, paint/layout hotspots.
* [ ] Run Lighthouse & bundle analyzer. Identify top bundle offenders.
* [ ] Apply targeted fixes (split components, memoize, lazy load, virtualize).
* [ ] Re-profile and compare before/after metrics.
* [ ] Add Web Vitals for real-user data.

---

## Helpful commands / snippets

**Generate production build (CRA) with source maps**

```bash
# CRA
GENERATE_SOURCEMAP=true npm run build
npx source-map-explorer build/static/js/*.js
```

**Bundle analyze (webpack plugin)**

```bash
# install and run after build
npx webpack-bundle-analyzer build/stats.json
```

**Use React Profiler API**

```jsx
<Profiler id="App" onRender={onRenderCallback}>
  <App />
</Profiler>
```

**Collect Web Vitals**

```js
import { getCLS, getFID, getLCP } from 'web-vitals';
getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getLCP(sendToAnalytics);
```

---

## Common pitfalls & tips

* **Don’t optimize blindfolded** — always measure first.
* **Dev vs Prod**: dev builds are slower (unminified, extra checks). Prefer profiling in production-like builds for parse/eval time. But React DevTools Profiler works well during development for commit times.
* **Overuse of memoization** can add overhead — only memoize when it reduces render cost measurably.
* **Third-party scripts** can dominate load times; monitor and lazy-load them.
* **Profile on real devices** (slow CPU mobile) — desktop is misleading.

---

## Interview-ready answer (short)

To profile a React app: reproduce the slow case, record with **React DevTools Profiler** to find long commits and expensive components, use **Chrome DevTools Performance** to find long JS tasks and paints, and run **Lighthouse + bundle analyzers** to identify load/network issues. Fix hotspots by splitting components, memoizing or stabilizing props (useMemo/useCallback/React.memo), virtualizing large lists, code-splitting, and optimizing heavy JS. Always measure before and after, and collect Web Vitals for real-user impact.
