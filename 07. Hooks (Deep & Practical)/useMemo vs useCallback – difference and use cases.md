# useMemo vs useCallback - difference and use cases

## Overview

Both `useMemo` and `useCallback` are React Hooks used for performance optimization.
They **memoize values or functions** to prevent unnecessary recalculations or re-creations during re-renders.
The difference lies in **what** they memoize:

* `useMemo` → memoizes **a computed value**
* `useCallback` → memoizes **a function reference**

---

## 1. useMemo — Memoize a Computed Value

### Purpose

`useMemo` caches the **result** of a calculation and recomputes it only when its dependencies change.

### Syntax

```javascript
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

### Example

```javascript
import { useMemo, useState } from "react";

function Numbers() {
  const [count, setCount] = useState(0);
  const [numbers] = useState([1, 2, 3, 4, 5]);

  const total = useMemo(() => {
    console.log("Calculating sum...");
    return numbers.reduce((acc, n) => acc + n, 0);
  }, [numbers]);

  return (
    <div>
      <p>Sum: {total}</p>
      <button onClick={() => setCount(count + 1)}>Increment ({count})</button>
    </div>
  );
}
```

**Without `useMemo`**, the sum recalculates on every re-render.
**With `useMemo`**, it recalculates only when `numbers` changes.

### Use Cases

* Expensive computations (sorting, filtering, calculations)
* Derived state that depends on other values
* Avoiding recalculations unless dependencies change

---

## 2. useCallback — Memoize a Function Reference

### Purpose

`useCallback` returns a **memoized version of a function** that only changes when its dependencies change.
This prevents unnecessary re-renders in child components that receive callback props.

### Syntax

```javascript
const memoizedFn = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### Example

```javascript
import { useState, useCallback } from "react";

function Button({ onClick }) {
  console.log("Button rendered");
  return <button onClick={onClick}>Click Me</button>;
}

const MemoButton = React.memo(Button);

function App() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log("Clicked!");
  }, []); // same function reference every render

  return (
    <div>
      <p>Count: {count}</p>
      <MemoButton onClick={handleClick} />
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

**Without `useCallback`**, `handleClick` is recreated on every render, causing `Button` to re-render even if not needed.
**With `useCallback`**, React.memo prevents unnecessary re-renders by reusing the same function reference.

### Use Cases

* Preventing unnecessary re-renders in memoized child components
* Stabilizing function references between renders
* Passing stable callbacks to dependencies inside `useEffect` or `useMemo`

---

## 3. Key Differences

| Feature          | useMemo                       | useCallback                       |
| ---------------- | ----------------------------- | --------------------------------- |
| Purpose          | Memoizes a **computed value** | Memoizes a **function reference** |
| Returns          | The computed **result**       | The **function** itself           |
| Typical Use Case | Expensive calculations        | Stable callback functions         |
| Example Return   | A number, array, or object    | A memoized function               |
| Usage with       | Computed data                 | React.memo or event handlers      |

---

## 4. Common Pitfalls

* Overusing either Hook can **hurt performance** due to dependency tracking overhead.
* Always include all dependencies in the dependency array.
* Don’t use them prematurely — profile first and use only where re-renders are costly.

---

## 5. Interview-Ready Answer

`useMemo` and `useCallback` both help optimize React performance by memoizing values between renders.
`useMemo` caches the result of a computation, while `useCallback` caches the function definition itself.
Use `useMemo` for expensive derived values, and `useCallback` to avoid unnecessary re-renders when passing functions to memoized child components.
