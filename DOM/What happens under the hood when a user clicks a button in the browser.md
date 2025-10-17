# What happens under the hood when a user clicks a button in the browser

1. Quick answer

A click triggers the pointer/mouse event sequence (pointerdown → mousedown → mouseup → click). The browser creates an Event object, dispatches it through capturing, target, and bubbling phases, invoking matching listeners. The event may cause DOM mutations, reflow, paint, or network activity.

2. Steps

- Input device reports the event to OS; browser receives it.
- Browser translates to DOM events (pointer, mouse, keyboard) and creates an Event object.
- Event dispatch: capturing (top-down), target, bubbling (bottom-up).
- Event handlers run synchronously; they can call `preventDefault()` or `stopPropagation()`.
- If DOM changes occur, layout (reflow) and paint may be triggered.

3. Performance notes

- Avoid heavy work inside event handlers; defer to requestAnimationFrame or web workers for intensive tasks.
- Batch DOM updates to minimize reflows.

4. Interview-ready summary

A click event is created and dispatched across the DOM, invoking listeners through capture/target/bubble phases. Handlers can mutate the DOM, which may trigger layout and paint. Keep handlers lightweight and defer heavy work to avoid jank.
