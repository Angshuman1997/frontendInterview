# React Hooks Behavior Scenarios — `useMemo`

## Overview

`useMemo` is a React Hook that **memoizes** (caches) the *result* of a computation between renders.
It re-computes the value **only when one of its dependencies changes**.

The main goal is to **optimize performance** — avoid recalculating expensive logic or re-creating derived objects on every render.

---

## 1️⃣ Basic Example — Memoizing a Computation

```jsx
import { useMemo, useState } from "react";

function ExpensiveCalculation() {
  const [count, setCount] = useState(0);
  const [other, setOther] = useState(false);

  const result = useMemo(() => {
    console.log("Calculating...");
    return count * 2;
  }, [count]);

  return (
    <div>
      <p>Count: {count}</p>
      <p>Result: {result}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setOther(!other)}>Toggle</button>
    </div>
  );
}
```

**Explanation**

* When `count` changes → `useMemo` recalculates `result`.
* When `other` changes → cached value reused (no re-calculation).

**Logs:**

```
Calculating... // only when count changes
```

✅ **Render Count:** Re-renders on every state change, but **calculation runs only when dependencies change.**

---

## 2️⃣ Without `useMemo` (No Optimization)

```jsx
const result = (() => {
  console.log("Calculating...");
  return count * 2;
})();
```

**Explanation**
This re-computes on *every render* — even when unrelated state changes.
Use `useMemo` to cache results and avoid redundant heavy computation.

---

## 3️⃣ Returning Objects or Arrays — Preventing Re-renders

### Problem Example

```jsx
function Child({ config }) {
  console.log("Child render");
  return <p>{config.value}</p>;
}

function Parent({ value }) {
  const config = { value }; // new object every render
  return <Child config={config} />;
}
```

**Explanation**
Each render creates a **new object reference** → causes `Child` to re-render even if value is same.

✅ **Fix**

```jsx
const config = useMemo(() => ({ value }), [value]);
```

**Result:**
Child re-renders only when `value` changes.

---

## 4️⃣ Derived Value Computation

```jsx
const filteredUsers = useMemo(() => {
  return users.filter((u) => u.active);
}, [users]);
```

**Explanation**
If `users` array changes, recompute filtered list.
If other states change, reuse cached filtered list.

✅ Prevents unnecessary `.filter()` calls on every render.

---

## 5️⃣ Empty Dependency Array (`[]`)

```jsx
const memoValue = useMemo(() => {
  console.log("Run once");
  return heavyCalculation();
}, []);
```

**Explanation**
`useMemo` runs **only once** (on mount).
Useful for constants or expensive setup logic.

✅ **Behaves like `useEffect(() => {...}, [])` but returns a value.**

---

## 6️⃣ Changing Dependencies

```jsx
const value = useMemo(() => {
  console.log("Recomputing...");
  return a + b;
}, [a, b]);
```

* If `a` or `b` changes → recompute.
* If neither changes → reuse last computed value.

✅ Avoid unnecessary recomputation, especially when values come from props or context.

---

## 7️⃣ Expensive Function Example

```jsx
function factorial(n) {
  console.log("Computing factorial...");
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}

const fact = useMemo(() => factorial(number), [number]);
```

**Without `useMemo`:** factorial runs on every render.
**With `useMemo`:** only runs when `number` changes.

✅ Huge performance gain for expensive computations.

---

## 8️⃣ Avoid Misuse — `useMemo` ≠ “Performance Fix”

❌ Wrong:

```jsx
const doubled = useMemo(() => count * 2, [count]);
```

✅ Correct:

```jsx
const doubled = count * 2;
```

**Explanation**
Use `useMemo` *only* for **expensive** operations or objects causing re-renders — not for trivial calculations.

---

## 9️⃣ Using `useMemo` to Stabilize Dependencies in `useEffect`

```jsx
const options = useMemo(() => ({ theme, layout }), [theme, layout]);

useEffect(() => {
  applySettings(options);
}, [options]);
```

**Explanation**
Without `useMemo`, `options` creates a new object every render → triggers effect again unnecessarily.
With `useMemo`, effect runs only when actual dependencies change.

✅ Common pattern to prevent infinite loops in effects.

---

## 🔟 Interaction with `React.memo`

```jsx
const Child = React.memo(({ data }) => {
  console.log("Child render");
  return <div>{data.value}</div>;
});

function Parent({ value }) {
  const data = useMemo(() => ({ value }), [value]);
  return <Child data={data} />;
}
```

**Explanation**
`React.memo` prevents re-render when props are shallowly equal.
`useMemo` ensures `data` object reference is stable → avoids unwanted re-renders.

✅ Powerful combo for performance optimization.

---

## 11️⃣ Common Mistake — Missing Dependency

```jsx
const result = useMemo(() => compute(a, b), [a]);
```

**Issue**
If `b` changes, value doesn’t update → stale result.

✅ **Fix**

```jsx
const result = useMemo(() => compute(a, b), [a, b]);
```

Always include all external variables used inside the memo callback.

---

## 12️⃣ useMemo vs useCallback (Quick Difference)

| Hook            | Memoizes           | Returns  | Used For                              |
| --------------- | ------------------ | -------- | ------------------------------------- |
| **useMemo**     | Computation result | Value    | Expensive calculations / derived data |
| **useCallback** | Function reference | Function | Stable callback references            |

---

## 13️⃣ useMemo and Re-render Count Test

```jsx
function Test() {
  const [count, setCount] = useState(0);
  const [toggle, setToggle] = useState(false);

  const value = useMemo(() => {
    console.log("useMemo run");
    return count * 10;
  }, [count]);

  console.log("Component render");

  return (
    <>
      <p>{value}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={() => setToggle(t => !t)}>Toggle</button>
    </>
  );
}
```

**Logs:**

```
Component render
useMemo run  // only when count changes
```

✅ Changing `toggle` → re-render only (no recompute)
✅ Changing `count` → re-render + recompute

---

## 14️⃣ Returning Functions Inside `useMemo`

You can use `useMemo` to memoize function *results*, but not to stabilize functions — that’s what `useCallback` is for.

```jsx
const getDouble = useMemo(() => {
  return (n) => n * 2;
}, []);
```

**Explanation**
This recreates the same function once.
Usually better to use `useCallback(() => n * 2, [])` instead.

---

## 15️⃣ useMemo Performance Pitfall

Overusing `useMemo` can hurt performance.
React still needs to compare dependencies and store memoized values.

✅ Use it only when:

* Computation is **expensive**.
* The result prevents **child re-renders** or **re-execution** of heavy logic.

---

## Summary Table

| #  | Scenario                      | Behavior                             | Benefit                   |
| -- | ----------------------------- | ------------------------------------ | ------------------------- |
| 1  | Basic computation memoization | Recomputes only on dependency change | Avoid redundant work      |
| 2  | Without useMemo               | Always recomputes                    | No optimization           |
| 3  | Objects/arrays memoization    | Prevents re-renders                  | Stable references         |
| 4  | Derived values                | Cached filtering/mapping             | Better performance        |
| 5  | Empty dependency array        | Runs once                            | Constant setup            |
| 6  | Changing dependencies         | Recomputes selectively               | Accurate updates          |
| 7  | Expensive function            | Runs only when needed                | Faster renders            |
| 8  | Misuse on trivial ops         | No gain                              | Avoid overuse             |
| 9  | Stabilize useEffect deps      | Prevents infinite effect loops       | Stable dependency objects |
| 10 | With React.memo               | Prevents child re-render             | Optimal combo             |
| 11 | Missing deps                  | Stale result                         | Always list all deps      |
| 12 | useMemo vs useCallback        | Different purposes                   | Key distinction           |
| 13 | Re-render count               | Computation only when needed         | Debug clarity             |
| 14 | Returning functions           | Rare use                             | Prefer useCallback        |
| 15 | Overuse pitfall               | Unnecessary cost                     | Use sparingly             |

---

## Interview-Ready Answer

`useMemo` memoizes the **result of a computation** and reuses it across renders until its dependencies change.
It’s useful for **expensive calculations**, **derived state**, or **stabilizing object/array references** passed to children or effects.
React only recomputes when dependencies change, avoiding redundant work.
However, overusing `useMemo` can backfire — use it only when it solves a specific performance issue.
