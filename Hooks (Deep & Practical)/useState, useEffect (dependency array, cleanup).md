# useState, useEffect (dependency array, cleanup)

## Overview

`useState` and `useEffect` are two of the most fundamental React Hooks.

* **`useState`** manages component state.
* **`useEffect`** handles side effects like data fetching, subscriptions, timers, and DOM updates.

Understanding how the **dependency array** and **cleanup function** work in `useEffect` is essential for writing predictable, bug-free React code.

---

## 1. useState — Managing Component State

### Purpose

`useState` lets you add and manage state variables in functional components.

### Syntax

```javascript
const [state, setState] = useState(initialValue);
```

* **state**: Current state value.
* **setState**: Function to update the state.
* **initialValue**: The default value for the state.

### Example

```javascript
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

### Notes

* Updating state triggers a re-render.
* Multiple state updates may be batched together for performance.
* You can initialize state with a function:

  ```js
  const [value, setValue] = useState(() => computeInitialValue());
  ```

---

## 2. useEffect — Managing Side Effects

### Purpose

`useEffect` allows you to perform side effects (things outside the component’s render process), such as:

* Fetching data
* Setting up subscriptions or timers
* Manually changing the DOM
* Listening to events

### Syntax

```javascript
useEffect(() => {
  // side effect logic
  return () => {
    // cleanup (optional)
  };
}, [dependencies]);
```

* The **callback function** runs **after** the component renders.
* The **dependency array** controls when the effect runs.
* The **cleanup function** runs before the component unmounts or before the next effect runs.

---

## 3. Dependency Array — Controlling Execution

### Case 1 — No Dependency Array

```javascript
useEffect(() => {
  console.log("Runs after every render");
});
```

✅ Runs **after every render** (including state/prop changes).
⚠️ Can cause performance issues if not controlled properly.

---

### Case 2 — Empty Dependency Array `[]`

```javascript
useEffect(() => {
  console.log("Runs only once (on mount)");
}, []);
```

✅ Runs **only once** — after the initial render.
✅ Commonly used for API calls or event subscriptions.

---

### Case 3 — With Dependencies

```javascript
useEffect(() => {
  console.log("Runs when count changes");
}, [count]);
```

✅ Runs **only** when one of the specified dependencies changes.
✅ React uses shallow comparison to check dependency changes.

---

### Important Note

Always include all values that your effect depends on in the dependency array to avoid stale values.

If you intentionally want to skip dependencies, use `useCallback` or `useMemo` to stabilize values.

---

## 4. Cleanup Function — Preventing Memory Leaks

### Purpose

The function returned from `useEffect` runs during the **cleanup phase**:

* Before the component unmounts.
* Before the effect re-runs due to dependency changes.

It’s used to:

* Remove event listeners.
* Cancel API requests.
* Clear timers or intervals.
* Unsubscribe from services.

### Example — Cleanup

```javascript
import { useEffect, useState } from "react";

function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);

    return () => {
      clearInterval(interval); // cleanup
      console.log("Timer cleared");
    };
  }, []); // runs once

  return <p>Time: {seconds}s</p>;
}
```

Without cleanup, the timer would continue running even after the component unmounts — leading to a memory leak.

---

## 5. Common Patterns

### Fetch Data on Mount

```javascript
useEffect(() => {
  async function fetchData() {
    const res = await fetch("/api/data");
    const data = await res.json();
    console.log(data);
  }
  fetchData();
}, []); // run once
```

### Run Effect on State Change

```javascript
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]);
```

### Subscribe and Unsubscribe Example

```javascript
useEffect(() => {
  const handleResize = () => console.log(window.innerWidth);
  window.addEventListener("resize", handleResize);

  return () => window.removeEventListener("resize", handleResize);
}, []);
```

---

## 6. Summary Table

| Concept                       | Description                        | Runs When         |
| ----------------------------- | ---------------------------------- | ----------------- |
| `useState`                    | Manage state values                | On render/update  |
| `useEffect(() => {})`         | Runs after every render            | Always            |
| `useEffect(() => {}, [])`     | Runs once on mount                 | Component mount   |
| `useEffect(() => {}, [deps])` | Runs when dependencies change      | On dep change     |
| Cleanup function              | Runs before unmount or next effect | On re-run/unmount |

---

## 7. Interview-Ready Answer

`useState` manages local state in functional components, while `useEffect` handles side effects like data fetching, subscriptions, and DOM updates.
The dependency array determines when an effect runs:

* No array → runs after every render.
* Empty array → runs once (on mount).
* With dependencies → runs when dependencies change.
  The cleanup function is returned from `useEffect` and runs before the next effect or when the component unmounts, preventing memory leaks and stale subscriptions.
