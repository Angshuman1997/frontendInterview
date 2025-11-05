# How does React batch state updates

## Overview

React batches multiple state updates into a single render to improve performance.
Instead of re-rendering a component after every `setState` or `useState` call, React groups all updates that happen within the same event loop (like a click handler) and performs one final render with the latest state.

This reduces unnecessary re-renders and ensures smoother UI updates.

---

## How It Works

When you call multiple state updates in a single event, React doesn’t immediately apply them.
It adds them to a queue, waits until all synchronous code in that event has run, and then processes all updates together in one render pass.

### Example:

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  }

  console.log("Render:", count);

  return <button onClick={handleClick}>Increment</button>;
}
```

**Without batching:**
You might expect `count` to increment by 3.

**With batching:**
React batches the updates, so all `setCount` calls use the same `count` value (0), resulting in only one increment — final `count` = 1.

---

## Updating State Based on Previous Value

To handle multiple updates correctly, use the **functional update form** of `setState` or `useState`.

### Correct example:

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
  }

  console.log("Render:", count);

  return <button onClick={handleClick}>Increment</button>;
}
```

This ensures each update gets the latest value of `count`.
Final `count` = 3 (correct behavior).

---

## When Batching Happens

### Batching **does happen**:

* Inside React event handlers (onClick, onChange, etc.)
* Inside lifecycle methods or effects
* Inside React synthetic events

### Batching **does not happen** (before React 18):

* Inside `setTimeout`, `Promise.then`, or async callbacks

### In React 18 and later:

React automatically batches updates even in async scenarios like timeouts, fetch calls, or Promise callbacks.

Example (React 18+):

```javascript
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React batches both even though it's async
});
```

---

## Benefits of Batching

* **Improved performance:** Reduces unnecessary re-renders.
* **Consistency:** Keeps the UI in sync with the latest state.
* **Efficiency:** Merges multiple updates into one DOM operation.

---

## Interview-Ready Answer

React batches multiple state updates into a single render for performance optimization.
When multiple `setState` or `useState` calls happen within the same event loop, React queues them and performs one render with the final computed state.
This prevents unnecessary re-renders and keeps the UI efficient.
From React 18 onward, automatic batching also applies to asynchronous code like Promises and timeouts.
