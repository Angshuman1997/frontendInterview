# React Fiber (what it solves and how it enables concurrent rendering)

React Fiber is the internal engine that powers React's rendering and reconciliation process, introduced in React 16. It was a complete rewrite of React's core algorithm to make rendering more flexible, interruptible, and efficient, especially for complex applications. The key goal of Fiber is to enable **incremental rendering** — the ability to split rendering work into small units and spread it over multiple frames instead of blocking the main thread.

1. Problem Before React Fiber
   In earlier versions of React (before v16), the reconciliation process was **synchronous**.
   When React started rendering a component tree, it couldn’t stop in the middle — it had to finish rendering the entire tree before responding to user input or other updates.
   This caused performance issues like:

* UI freezes when rendering large trees.
* Slow responsiveness during heavy computations.
* Lack of prioritization (urgent updates like user input couldn’t interrupt slow background updates).

Example problem (pre-Fiber):
If a large list component re-rendered due to a state change, React would block the main thread until the whole render finished, causing frame drops and lag.

2. What React Fiber Solves
   React Fiber introduced a new architecture to make rendering:

* **Interruptible:** React can pause work in progress and resume it later.
* **Prioritized:** React can assign priorities to updates (e.g., user interactions > background updates).
* **Chunked:** React splits rendering work into smaller units of work called “fibers”.
* **Recoverable:** React can reuse partially completed work if needed.

This makes React apps smoother, especially when dealing with animations, transitions, and large UIs.

3. How Fiber Works Internally
   React Fiber breaks down the rendering work into individual units (fibers). Each component instance becomes a “fiber node” with information about:

* The component type (function/class)
* Props and state
* The component’s parent, child, and sibling relationships
* Work priority and expiration time

Rendering now happens in two main phases:

1. **Render (Reconciliation) Phase:**
   React builds or updates the virtual DOM tree. This phase can be paused, aborted, or resumed.
2. **Commit Phase:**
   React applies the changes to the actual DOM. This phase is always synchronous and fast.

During the render phase, React processes each fiber node, decides what changed, and can yield control back to the browser to keep the UI responsive.

Example behavior:
React can render part of a component tree, pause to handle a user scroll or input, then continue rendering the rest later without losing state.

4. Concurrent Rendering
   Fiber is the foundation for **Concurrent React** (enabled via features like `useTransition` and `Suspense`). Concurrent rendering allows React to:

* Start rendering multiple versions of the UI simultaneously.
* Interrupt non-urgent updates if something more important happens (like user input).
* Prepare updates in the background without blocking the visible UI.

This improves perceived performance and makes UIs feel faster even when heavy computations occur.

Example:
If the user types into a search box, React can prioritize updating the input field (high priority) while deferring the rendering of a large result list (low priority) to avoid lag.

5. Key Concepts in Fiber

* **Work units (fibers):** Small chunks of render work.
* **Priority levels:** Urgent, normal, or background updates.
* **Yielding:** Pausing work to allow the browser to handle high-priority tasks.
* **Commit phase:** The only synchronous step where the DOM updates occur.

6. Benefits of React Fiber

* Smooth user experience even in complex UI.
* Better performance for animations and transitions.
* Fine-grained control over rendering priorities.
* Foundation for features like Suspense and Concurrent Mode.

7. Interview-ready answer:
   React Fiber is a complete rewrite of React’s reconciliation engine introduced in React 16. It solves performance limitations of the old synchronous rendering model by making rendering incremental, interruptible, and prioritized. Fiber breaks rendering into small units of work called “fibers,” allowing React to pause, resume, or reorder work as needed. This enables concurrent rendering, where React can prepare updates in the background and keep the UI responsive during heavy updates. Fiber is the core architecture behind modern React features like Suspense and concurrent mode.
