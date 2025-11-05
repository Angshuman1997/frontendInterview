# Explain event bubbling and event capturing

1. Quick answer

Event capturing and bubbling are two phases of event propagation in the DOM. Capturing goes top-down (document → target), bubbling goes bottom-up (target → document). By default, event listeners are registered for the bubbling phase unless `{ capture: true }` is specified.

2. Phases

- Capturing phase: the event travels from the window/document down to the target element.
- Target phase: the event has reached the target.
- Bubbling phase: the event bubbles back up from the target to the document.

3. Example

```js
// Bubbling listener
element.addEventListener('click', handler);
// Capturing listener
element.addEventListener('click', handler, { capture: true });
```

4. stopPropagation and stopImmediatePropagation

- `event.stopPropagation()`: prevents further propagation in the current phase.
- `event.stopImmediatePropagation()`: prevents other listeners of the same event on the same element from running.

5. Use cases

- Use capturing when you need to intercept events before child handlers run (rare).
- Bubbling is commonly used for event delegation at parent containers.

6. Interview-ready summary

The DOM event model has capturing (top-down) and bubbling (bottom-up) phases. Listeners default to bubbling unless `{ capture: true }` is provided. Use `stopPropagation` to prevent further propagation and event delegation to handle many child events efficiently.
