# React Hooks Behavior Scenarios — `useCallback`

## Overview

`useCallback` is a React Hook that **memoizes a function reference**.
It ensures that the same function instance is returned between renders — **unless its dependencies change**.

This is crucial for **performance optimization** and **preventing unnecessary re-renders**, especially when passing callbacks to child components or using them in dependencies of other hooks (like `useEffect` or `useMemo`).

---

## 1️⃣ Basic Example — Function Memoization

```jsx
import { useState, useCallback } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState("light");

  const increment = useCallback(() => {
    setCount((prev) => prev + 1);
  }, []); // no dependencies

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>Increment</button>
      <button onClick={() => setTheme(t => (t === "light" ? "dark" : "light"))}>
        Toggle Theme
      </button>
    </div>
  );
}
```

**Explanation**

* `increment` function is created only **once** (memoized).
* Changing `theme` won’t create a new function reference.

✅ Prevents child components from re-rendering unnecessarily when passed as a prop.

---

## 2️⃣ Without `useCallback` (Problem Example)

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = () => setCount(count + 1);
  return <Child onClick={handleClick} />;
}
```

Every render creates a **new `handleClick` reference** → causes `Child` to re-render, even if `count` didn’t change.

✅ **Fix**

```jsx
const handleClick = useCallback(() => setCount(c => c + 1), []);
```

Now the function reference is stable, preventing unnecessary child renders.

---

## 3️⃣ `useCallback` + `React.memo`

```jsx
const Child = React.memo(({ onAction }) => {
  console.log("Child render");
  return <button onClick={onAction}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  const handleAction = useCallback(() => {
    console.log("Clicked");
  }, []);

  return (
    <>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Parent Increment</button>
      <Child onAction={handleAction} />
    </>
  );
}
```

**Explanation**

* Without `useCallback`, `Child` re-renders on every parent render.
* With `useCallback`, `Child` re-renders **only when necessary**.

✅ **Perfect combo:** `React.memo` + `useCallback`.

---

## 4️⃣ Dependencies in `useCallback`

```jsx
const handleClick = useCallback(() => {
  console.log("Count:", count);
}, [count]);
```

**Explanation**
Whenever `count` changes, a **new function reference** is created.
If dependencies don’t change, React reuses the memoized function.

✅ Always include all variables used inside the callback in the dependency array.

---

## 5️⃣ Using Empty Dependency Array (`[]`)

```jsx
const logMessage = useCallback(() => {
  console.log("Hello");
}, []);
```

**Explanation**
Function is created only once (on mount) and never changes.
Equivalent to defining the function outside the component scope.

✅ Good for static callbacks that don’t depend on state/props.

---

## 6️⃣ Passing Stable Functions to Children

```jsx
const Child = React.memo(({ onToggle }) => {
  console.log("Child render");
  return <button onClick={onToggle}>Toggle</button>;
});

function Parent() {
  const [open, setOpen] = useState(false);

  const toggle = useCallback(() => setOpen(o => !o), []);
  return <Child onToggle={toggle} />;
}
```

**Explanation**

* `toggle` function reference remains the same across renders.
* `Child` doesn’t re-render unnecessarily.

✅ Prevents wasted re-renders — very common in performance interviews.

---

## 7️⃣ useCallback Inside useEffect

```jsx
const fetchData = useCallback(() => {
  console.log("Fetching...");
}, []);

useEffect(() => {
  fetchData();
}, [fetchData]);
```

**Explanation**

* If `fetchData` wasn’t wrapped in `useCallback`, it would recreate every render, retriggering the effect infinitely.
* With `useCallback`, dependency is stable → safe to include in effect dependencies.

✅ Avoids infinite effect loops.

---

## 8️⃣ `useCallback` vs `useMemo`

| Hook            | Memoizes           | Returns               | Common Use                                     |
| --------------- | ------------------ | --------------------- | ---------------------------------------------- |
| **useMemo**     | Computed **value** | The computed result   | Cache expensive computations                   |
| **useCallback** | Function reference | The **same function** | Prevent unnecessary re-renders or effect loops |

✅ `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

---

## 9️⃣ When Not to Use `useCallback`

❌ Overuse example:

```jsx
const handleClick = useCallback(() => setVisible(true), []);
```

This is unnecessary — defining the function inline would perform identically unless passed as a prop to a memoized child.

✅ Use `useCallback` **only when**:

* The function is passed to a **child component** wrapped in `React.memo()`.
* The function is used as a **dependency** inside another hook (like `useEffect`, `useMemo`).

---

## 🔟 Using `useCallback` with Async Functions

```jsx
const fetchUsers = useCallback(async () => {
  const res = await fetch("/api/users");
  const data = await res.json();
  setUsers(data);
}, []);
```

**Explanation**
Even async functions can be memoized — only redefined when dependencies change.
Useful in data fetching hooks or effects.

---

## 11️⃣ useCallback and State Updates

```jsx
const increment = useCallback(() => {
  setCount(c => c + 1);
}, []);
```

**Explanation**
✅ Using functional state update `(c => c + 1)` makes callback **independent of state**.
Thus, no need to include `count` as a dependency.
Prevents unnecessary re-creation.

---

## 12️⃣ Common Mistake — Missing Dependency

```jsx
const handleClick = useCallback(() => {
  console.log(value);
}, []); // ❌ Missing dependency
```

**Explanation**
If `value` changes, callback still logs the old one (stale closure).
Always include `value` in dependencies.

✅ **Fix**

```jsx
const handleClick = useCallback(() => {
  console.log(value);
}, [value]);
```

---

## 13️⃣ React.memo Without useCallback (Re-render Trap)

```jsx
const Child = React.memo(({ onPress }) => {
  console.log("Child render");
  return <button onClick={onPress}>Click</button>;
});

function Parent() {
  const handleClick = () => console.log("Clicked");
  return <Child onPress={handleClick} />;
}
```

**Explanation**
Even though `Child` is memoized, it re-renders because `handleClick` creates a new reference on each render.

✅ **Fix**

```jsx
const handleClick = useCallback(() => console.log("Clicked"), []);
```

Now `Child` no longer re-renders unnecessarily.

---

## 14️⃣ Multiple `useCallback`s in One Component

```jsx
const inc = useCallback(() => setCount(c => c + 1), []);
const dec = useCallback(() => setCount(c => c - 1), []);
```

**Explanation**
Each callback is memoized separately.
React reuses the same references until dependencies change.

✅ Safe and efficient pattern for managing multiple handlers.

---

## 15️⃣ Performance Overhead of useCallback

* `useCallback` itself has a small cost (dependency tracking).
* Overusing it for simple functions (not passed as props) can degrade performance slightly.
* Always **measure or identify render issues first** before adding `useCallback`.

---

## Summary Table

| #  | Scenario              | Behavior                      | Key Takeaway                |
| -- | --------------------- | ----------------------------- | --------------------------- |
| 1  | Basic memoization     | Function created once         | Stable reference            |
| 2  | Without `useCallback` | New function per render       | Causes re-renders           |
| 3  | With `React.memo`     | Skips child renders           | Great optimization pair     |
| 4  | With dependencies     | New function when deps change | Accurate and safe           |
| 5  | Empty deps            | Constant callback             | Useful for static functions |
| 6  | Passed to child       | Prevents child re-renders     | Stable callback             |
| 7  | Used in useEffect     | Prevents infinite loops       | Safe dependency             |
| 8  | vs `useMemo`          | Memoizes function vs value    | Concept difference          |
| 9  | Overuse               | Unnecessary                   | Use only when needed        |
| 10 | Async functions       | Works same way                | Common in fetch calls       |
| 11 | Functional update     | Avoids dependency             | Most efficient pattern      |
| 12 | Missing deps          | Stale closure                 | Always include deps         |
| 13 | React.memo trap       | Re-render without useCallback | Needs memoization           |
| 14 | Multiple callbacks    | Independent                   | Safe practice               |
| 15 | Performance cost      | Minor                         | Use judiciously             |

---

## Interview-Ready Answer

`useCallback` memoizes a **function reference**, ensuring it remains stable across renders unless its dependencies change.
It’s mainly used to **prevent unnecessary re-renders** when passing callbacks to memoized child components or using them in hook dependencies.
Combined with `React.memo`, it helps React skip re-rendering components when props and callbacks haven’t changed.
Avoid overusing it — it’s beneficial only when function identity truly matters.
