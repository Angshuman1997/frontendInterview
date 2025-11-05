# React Hooks Behavior Scenarios — `useState`

## Overview

`useState` is one of React’s core hooks that allows components to **store and manage local state**.
When the state changes, the component **re-renders** with the updated value.

However, how and when React re-renders — especially when you call `setState` multiple times — depends on **React’s batching**, **closures**, and **dependency context**.

---

## 1️⃣ Basic Example

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </>
  );
}
```

✅ **Explanation**

* `useState(0)` initializes state to `0`.
* Every call to `setCount` triggers a **re-render** with the updated value.
* React re-renders only the component (and children if props changed).

✅ **Render Count:** 1 (initial) + 1 (on each `setCount`)

---

## 2️⃣ State Updates Are Asynchronous (Batched)

```jsx
function Example() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  };

  return <button onClick={handleClick}>Count: {count}</button>;
}
```

❌ **Result:** Only increases by **1**, not 3.

✅ **Why?**
React **batches** synchronous state updates inside event handlers and effects.
Each `setCount(count + 1)` uses the same **stale value of count** (`0`).

✅ **Fix (Functional Update):**

```jsx
setCount(prev => prev + 1);
setCount(prev => prev + 1);
setCount(prev => prev + 1);
```

✅ **Now:** count increases by **3**
Because React calls each updater function with the **latest state**.

---

## 3️⃣ Multiple State Variables Updated Together

```jsx
const [a, setA] = useState(1);
const [b, setB] = useState(1);

function update() {
  setA(a + 1);
  setB(b + 1);
}
```

✅ **Explanation**
React batches multiple state updates inside the same event.
Both `setA` and `setB` trigger only **one re-render**.

✅ **Render Count:** 2 (initial + one batched re-render)

---

## 4️⃣ State Update Inside `useEffect`

```jsx
useEffect(() => {
  setCount(1);
  setCount(2);
  setCount(3);
}, []);
```

✅ **Explanation**
All three state updates are **batched** (same event loop tick).
React takes the **final value** (`3`) and performs **one re-render**.

✅ **Render Count:** 2 (initial + one update)

---

## 5️⃣ State Update Without Dependency Array

```jsx
useEffect(() => {
  setCount(count + 1);
});
```

❌ **Explanation**
Effect runs after every render → `setCount` triggers another render → infinite loop.

✅ **Fix:** Add a dependency array:

```jsx
useEffect(() => setCount(count + 1), []);
```

---

## 6️⃣ Using State with Async Code

```jsx
function AsyncExample() {
  const [value, setValue] = useState(0);

  const handleAsync = async () => {
    setValue(value + 1);
    await fetch("/api");
    setValue(value + 1);
  };
}
```

✅ **React 18+ Behavior:**
Automatic batching applies even across async boundaries.
All state updates in one event loop are batched → one re-render.

✅ **React < 18:**
No async batching → two separate renders.

---

## 7️⃣ Using Previous State Safely

```jsx
const [count, setCount] = useState(0);

const increment = () => {
  setCount(prev => prev + 1);
};
```

✅ **Explanation**
Always use functional updates when new state depends on old state.
Prevents bugs caused by stale state closures.

---

## 8️⃣ State Setter Identity Is Stable

```jsx
const [count, setCount] = useState(0);

useEffect(() => {
  console.log("setCount stable");
}, [setCount]);
```

✅ **Explanation**
`setCount` is guaranteed to be **stable** across renders —
it never changes identity, so you can safely use it in dependencies.

---

## 9️⃣ Initial State Function Optimization

```jsx
const [data, setData] = useState(() => expensiveComputation());
```

✅ **Explanation**

* When you pass a function to `useState`, React calls it **only once** (on mount).
* Avoids recalculating expensive operations on every render.
* Equivalent to:

  ```jsx
  useState(expensiveComputation()); // ❌ called every render
  ```

---

## 🔟 State Updates Are Merged, Not Replaced (for Objects)

```jsx
const [user, setUser] = useState({ name: "Alice", age: 25 });

// ❌ Wrong - replaces entire object
setUser({ age: 26 });

// ✅ Correct - merge manually
setUser(prev => ({ ...prev, age: 26 }));
```

✅ **Explanation**
Unlike class components, React **does not merge** state objects automatically.
Always spread previous state manually.

---

## 11️⃣ Conditional State Updates

```jsx
if (count < 5) setCount(count + 1);
```

✅ **Explanation**
React doesn’t batch **across conditions**, but state updates still happen once per render cycle.
Be cautious — can create loops if condition always true.

---

## 12️⃣ Using useState with Objects or Arrays

```jsx
const [list, setList] = useState([]);

const addItem = () => {
  setList(prev => [...prev, "New Item"]);
};
```

✅ **Explanation**
Always use **functional updates** to avoid depending on stale list references.

---

## 13️⃣ State Updates Are Deferred Until Render

```jsx
console.log(count);
setCount(count + 1);
console.log(count);
```

**Output:**

```
0
0
```

✅ **Explanation**
State updates are **asynchronous** — React schedules the re-render but doesn’t update immediately within the same function.

---

## 14️⃣ setState to Same Value

```jsx
setCount(5);
setCount(5);
```

✅ **Explanation**
React shallow-compares new and old state.
If they’re equal → **no re-render**.

✅ **Render Count:** 0 (if same value)

---

## 15️⃣ Lazy State Initialization Example

```jsx
function Component() {
  const [value] = useState(() => {
    console.log("Initialized once");
    return Math.random();
  });
}
```

✅ **Explanation**
Initializer function runs **only once**, even if the component re-renders.
Used for expensive or random initialization logic.

---

## Summary Table

| #  | Scenario           | Behavior                | Key Takeaway              |
| -- | ------------------ | ----------------------- | ------------------------- |
| 1  | Basic              | Updates + re-render     | Standard behavior         |
| 2  | Multiple setState  | Batched                 | Functional updates needed |
| 3  | Multiple states    | Batched                 | One re-render             |
| 4  | setState in effect | Batched                 | One update                |
| 5  | Missing deps       | Infinite loop           | Add dependency array      |
| 6  | Async batching     | React 18+ batches async | Version dependent         |
| 7  | Functional update  | Safe & latest           | Avoid stale state         |
| 8  | Setter stable      | Doesn’t change          | Safe for deps             |
| 9  | Lazy init          | Runs once               | Performance optimization  |
| 10 | Object updates     | No auto-merge           | Spread manually           |
| 11 | Conditional update | Careful with loops      | Condition must end        |
| 12 | Arrays             | Functional update       | Avoid mutation            |
| 13 | Immediate logs     | Async scheduling        | Understand reactivity     |
| 14 | Same value         | No render               | Optimization              |
| 15 | Lazy init func     | Runs once               | Expensive ops only        |

---

## Interview-Ready Answer

`useState` allows React components to store local state.
Updating state triggers a re-render, but React **batches multiple synchronous updates** into a single render for performance.
If a new state depends on the old one, always use the **functional form** (`setState(prev => ...)`).
State updates are asynchronous and don’t immediately reflect inside the same function body.
In React 18+, **automatic batching** also applies to async operations, minimizing unnecessary re-renders.
