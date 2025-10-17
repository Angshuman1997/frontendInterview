# React Hooks Behavior Scenarios — `useEffect`

## Overview

`useEffect` is a React Hook used for **side effects**

---

## 1️⃣ Multiple `setState` Calls in the Same Effect

```jsx
useEffect(() => {
  setCount(1);
  setCount(2);
  setCount(3);
}, []);
```

**Explanation**
React batches all synchronous state updates inside one effect.
Only the **final value (`3`)** is applied, resulting in one re-render.

**Render Count**

* Initial render
* One re-render (count = 3)
  ✅ **Total = 2 renders**

---

## 2️⃣ Multiple Effects Each Updating State

```jsx
useEffect(() => setCount(1), []);
useEffect(() => setCount(2), []);
useEffect(() => setCount(3), []);
```

**Explanation**
Each effect runs independently after mount and triggers a separate state update.
React performs **3 re-renders**, one for each effect.

**Render Count**
Initial render + 3 updates → **Total = 4 renders**

---

## 3️⃣ `useEffect` Without Dependency Array

```jsx
useEffect(() => {
  setCount(count + 1);
});
```

**Explanation**
The effect runs after **every render** since no dependency array is provided.
Each `setCount()` triggers another re-render → infinite loop.

**Result**
❌ Infinite re-render loop

✅ **Fix**

```jsx
useEffect(() => {
  setCount(count + 1);
}, []); // Runs only once
```

---

## 4️⃣ State Update Depends on Previous Value

```jsx
useEffect(() => {
  setCount(count + 1);
}, []);
```

**Explanation**
On the first render, `count = 0`.
The effect runs once and sets `count = 1`.

**Render Count**
Initial + one update → **2 renders**

---

## 5️⃣ Multiple `setState` Calls Inside Event Handler

```jsx
<button
  onClick={() => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  }}
>
  +3
</button>
```

**Explanation**
All `setCount` calls use the same stale value of `count`.
React batches them → final count increases only by **1**.

**Render Count**
1 per click

✅ **Fix using functional updates**

```jsx
setCount(prev => prev + 1);
setCount(prev => prev + 1);
setCount(prev => prev + 1);
```

**Final Count:** `count + 3`

---

## 6️⃣ `useEffect` with State as Dependency

```jsx
useEffect(() => {
  setCount(count + 1);
}, [count]);
```

**Explanation**
Each update to `count` triggers the effect again, causing an **infinite loop**.

❌ **Result**
Infinite re-renders

✅ **Fix**
Add condition or remove dependency if unnecessary.

---

## 7️⃣ Conditional Effect Execution

```jsx
useEffect(() => {
  if (count === 0) setCount(1);
}, [count]);
```

**Explanation**
First render: count = 0 → updates to 1
Next render: count = 1 → condition false → stops updating.

✅ **Render Count:** 2
✅ **Final Count:** 1

---

## 8️⃣ Multiple State Variables Updated Together

```jsx
useEffect(() => {
  setA(a + 1);
  setB(b + 1);
}, []);
```

**Explanation**
React batches updates for multiple states within the same effect.
Triggers only one re-render.

✅ **Render Count:** 2 (initial + one update)

---

## 9️⃣ Effect Cleanup Function

```jsx
useEffect(() => {
  console.log("Mount");
  return () => console.log("Unmount");
}, []);
```

**Explanation**

* Runs once on mount.
* Cleanup runs on unmount or before the next effect re-runs.

✅ Useful for cleaning subscriptions, timers, or listeners.

---

## 🔟 State Update Inside `setTimeout`

```jsx
useEffect(() => {
  setTimeout(() => {
    setCount(1);
  }, 1000);
}, []);
```

**Explanation**
`setTimeout` executes asynchronously after 1 second.
React re-renders once when the state updates.

✅ **Render Count:** 2 (initial + one after timeout)

---

## 11️⃣ Async Function and Batching (React 18+)

```jsx
async function update() {
  setCount(count + 1);
  await fetch("/api/data");
  setCount(count + 1);
}
```

**Explanation**
Before React 18: async breaks batching → 2 renders.
React 18+ enables **automatic batching** → single re-render.

✅ **Render Count**

* React < 18 → 2 renders
* React ≥ 18 → 1 render

---

## 12️⃣ Derived State Missing Dependencies

```jsx
useEffect(() => {
  setFullName(firstName + " " + lastName);
}, [firstName]);
```

❌ **Explanation**
If `lastName` changes, effect doesn’t run → stale derived state.

✅ **Fix**

```jsx
useEffect(() => {
  setFullName(firstName + " " + lastName);
}, [firstName, lastName]);
```

✅ **Outcome:** Runs when either dependency changes.

---

## 13️⃣ Mount Detection Pattern

```jsx
useEffect(() => {
  if (mounted) return;
  setMounted(true);
}, [mounted]);
```

**Explanation**
Runs once to set `mounted` to true, then stops re-running.
Useful for detecting initial render.

✅ **Render Count:** 2

---

## 14️⃣ Parent → Child Re-renders via Props

```jsx
function Child({ value }) {
  console.log("Child render");
  return <p>{value}</p>;
}

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <Child value={count} />
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </>
  );
}
```

**Explanation**
Every time the parent updates `count`, the `Child` re-renders (prop changed).

✅ **Optimization**
Use `React.memo(Child)` to skip re-render if props are unchanged.

---

## 15️⃣ Setting State to Same Value

```jsx
setCount(5);
setCount(5);
```

**Explanation**
React does a shallow equality check — if the new state is the same, no re-render occurs.

✅ **Render Count:** 0 (if state already equals 5)

---

## Summary Table

| #  | Scenario                           | Renders      | Notes                            |
| -- | ---------------------------------- | ------------ | -------------------------------- |
| 1  | Multiple `setState` in same effect | 2            | Batched                          |
| 2  | Multiple effects updating state    | 4            | Sequential                       |
| 3  | No dependency array                | ∞            | Infinite loop                    |
| 4  | Increment once                     | 2            | Runs once                        |
| 5  | Multiple in handler                | 1 per click  | Functional updates required      |
| 6  | Effect depends on same state       | ∞            | Infinite                         |
| 7  | Conditional effect                 | 2            | Stops after one                  |
| 8  | Multiple states updated            | 2            | Batched                          |
| 9  | Cleanup example                    | Varies       | Cleanup on unmount               |
| 10 | `setTimeout` update                | 2            | Async re-render                  |
| 11 | Async batching (React 18)          | 1–2          | Version-dependent                |
| 12 | Missing deps                       | Wrong output | Add dependencies                 |
| 13 | Mount detection                    | 2            | Common pattern                   |
| 14 | Props from parent                  | Variable     | Child re-renders if props change |
| 15 | Same state value                   | 0            | No re-render                     |

---

## Interview-Ready Summary

React batches state updates that occur synchronously in the same tick.
`useEffect` with an empty dependency array runs once; without it, runs after every render (can loop infinitely).
React 18 introduced **automatic batching** even for async calls, minimizing renders.
Always include correct dependencies, use cleanup functions, and prefer functional updates to avoid stale values or infinite loops.
