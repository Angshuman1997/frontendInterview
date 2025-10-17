# What is event delegation and why is it useful

1. Quick answer

Event delegation attaches a single event listener to a parent element to handle events triggered by its children, using event bubbling to identify the event target. It's efficient and reduces memory/handler overhead.

2. Example

```js
document.querySelector('#list').addEventListener('click', (e) => {
  const item = e.target.closest('.item');
  if (item) { /* handle item click */ }
});
```

3. Benefits

- Fewer event listeners, better performance for many child nodes.
- Works for dynamically added children without reattaching handlers.

4. Considerations

- Use delegation for events that bubble (e.g., `click`). Some events like `focus` don’t bubble (use focusin/focusout instead).
- Carefully identify targets and guard against unexpected matches.

5. Interview-ready summary

Event delegation uses a parent listener plus event bubbling to efficiently handle events from many child nodes and dynamically added elements, reducing memory and binding complexity.
