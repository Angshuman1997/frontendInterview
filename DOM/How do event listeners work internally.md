# How do event listeners work internally

1. Quick answer

Event listeners are registered callbacks stored by the browser for specific event types on nodes. When an event is dispatched, the browser walks the propagation path and invokes matching listeners in the appropriate phase.

2. Registration and storage

- `addEventListener` registers a listener with options (`capture`, `once`, `passive`).
- Browsers maintain internal listener lists per node with metadata.

3. Dispatch and invocation

- Dispatching constructs an Event object and determines the propagation path.
- The browser invokes listeners in capture/target/bubble order.
- `passive: true` tells the browser the listener won’t call `preventDefault()`, allowing optimizations for touch scrolling.

4. Memory and removal

- Remove listeners with `removeEventListener` using the same function reference and options.
- Anonymous inline listeners cannot be removed easily.

5. Interview-ready summary

Internally, the browser stores listeners per node and invokes them during event dispatch in capture/target/bubble order. Use options like `passive` for performance, and always unregister listeners to avoid leaks when necessary.
