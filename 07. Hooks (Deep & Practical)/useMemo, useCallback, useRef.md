# useMemo, useCallback, useRef

## Overview

React provides `useMemo`, `useCallback`, and `useRef` Hooks for performance optimization and value persistence across renders.
Though they seem similar, they serve **different purposes**:

| Hook            | Purpose                                                                 |
| --------------- | ----------------------------------------------------------------------- |
| **useMemo**     | Memoizes a computed value                                               |
| **useCallback** | Memoizes a function reference                                           |
| **useRef**      | Persists a mutable value or DOM reference without triggering re-renders |

---

## 1. useMemo — Memoize Computed Values

### Purpose

`useMemo` caches the **result** of an expensive computation and recomputes it only when its dependencies change.

### Syntax

```javascript
const memoizedValue = useMemo(() => computeValue(a, b), [a, b]);
```

### Example

```javascript
import { useState, useMemo } from "react";

function ExpensiveCalculation() {
  const [count, setCount] = useState(0);
  const [numbers] = useState([1, 2, 3, 4, 5]);

  const total = useMemo(() => {
    console.log("Calculating total...");
    return numbers.reduce((acc, n) => acc + n, 0);
  }, [numbers]);

  return (
    <div>
      <p>Total: {total}</p>
      <button onClick={() => setCount(count + 1)}>Increment ({count})</button>
    </div>
  );
}
```

### Use Cases

* Avoid re-running expensive calculations (sorting, filtering, aggregation).
* Prevent unnecessary recalculations when dependencies haven’t changed.
* Cache derived data.

---

## 2. useCallback — Memoize Function References

### Purpose

`useCallback` returns a **memoized version of a function** that only changes when its dependencies change.
This is useful when passing callbacks to memoized child components.

### Syntax

```javascript
const memoizedFn = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### Example

```javascript
import { useState, useCallback } from "react";

function Child({ onClick }) {
  console.log("Child rendered");
  return <button onClick={onClick}>Click Me</button>;
}

const MemoChild = React.memo(Child);

function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log("Clicked!");
  }, []); // same function reference each render

  return (
    <div>
      <p>Count: {count}</p>
      <MemoChild onClick={handleClick} />
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

### Use Cases

* Prevent unnecessary re-renders in memoized children (`React.memo`).
* Maintain stable function identity between renders.
* Pass stable callbacks into dependencies for `useEffect` or `useMemo`.

---

## 3. useRef — Store Mutable Values or Access DOM Elements

### Purpose

`useRef` holds a **mutable reference** that persists across renders **without causing re-renders** when updated.
It can store either:

* A DOM element reference (`ref` attribute).
* Any mutable value that needs to persist between renders (like a previous value, timer ID, etc.).

### Syntax

```javascript
const ref = useRef(initialValue);
```

### Example 1 — Accessing a DOM Element

```javascript
import { useRef, useEffect } from "react";

function InputFocus() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus(); // Access DOM element directly
  }, []);

  return <input ref={inputRef} placeholder="Auto focused input" />;
}
```

### Example 2 — Persisting a Value Without Triggering Re-renders

```javascript
import { useRef, useEffect, useState } from "react";

function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef();

  useEffect(() => {
    intervalRef.current = setInterval(() => setCount(c => c + 1), 1000);
    return () => clearInterval(intervalRef.current);
  }, []);

  return <p>Timer: {count}</p>;
}
```

### Example 3 — Tracking Previous Value

```javascript
import { useRef, useEffect } from "react";

function usePrevious(value) {
  const prev = useRef();
  useEffect(() => {
    prev.current = value;
  }, [value]);
  return prev.current;
}
```

---

## 4. Key Differences

| Hook            | Memoizes                         | Triggers Re-render on Change | Typical Use                          |
| --------------- | -------------------------------- | ---------------------------- | ------------------------------------ |
| **useMemo**     | Computed **value**               | No                           | Expensive calculations               |
| **useCallback** | **Function reference**           | No                           | Prevent unnecessary child re-renders |
| **useRef**      | **Mutable object** (ref.current) | No                           | DOM access or value persistence      |

---

## 5. When to Use Each

* **useMemo:** Optimize expensive operations or computed values.
* **useCallback:** Optimize re-renders of children relying on stable function props.
* **useRef:**

  * Persist mutable data across renders without re-rendering.
  * Access or manipulate DOM nodes directly.
  * Store timer IDs, previous values, or imperatively accessed elements.

---

## 6. Interview-Ready Answer

`useMemo`, `useCallback`, and `useRef` are React Hooks that help with performance and value persistence:

* **useMemo** caches the result of a computation and recalculates only when dependencies change.
* **useCallback** caches a function definition to prevent unnecessary re-renders of child components.
* **useRef** stores mutable values or DOM references without causing re-renders.

Use `useMemo` for expensive calculations, `useCallback` for stable functions, and `useRef` for persistent values or DOM access.

