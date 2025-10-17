# Rules of Hooks and common pitfalls

## Overview

Hooks are special functions in React that let you use state and lifecycle features in functional components.
However, React enforces strict **Rules of Hooks** to ensure consistent behavior and predictable rendering.
Breaking these rules can cause errors, bugs, or unexpected re-renders.

---

## The Two Main Rules of Hooks

### **1. Only Call Hooks at the Top Level**

* Do **not** call Hooks inside loops, conditions, or nested functions.
* Always call Hooks at the **top level** of your component or custom Hook.

This ensures React can maintain the same **Hook order** between re-renders.
React uses the order in which Hooks are called to associate state and effects with the right components.

#### ❌ Incorrect:

```javascript
function Example({ isVisible }) {
  if (isVisible) {
    const [count, setCount] = useState(0); // ❌ breaks hook order
  }
}
```

#### ✅ Correct:

```javascript
function Example({ isVisible }) {
  const [count, setCount] = useState(0); // ✅ always called in the same order
  return isVisible ? <p>{count}</p> : null;
}
```

---

### **2. Only Call Hooks from React Functions**

Hooks can only be called:

* From **React function components**
* From **custom Hooks**

You **cannot** call Hooks from:

* Regular JavaScript functions
* Class components
* Event handlers or callback functions that are not Hooks

#### ❌ Incorrect:

```javascript
function notAComponent() {
  const [data, setData] = useState([]); // ❌ invalid
}
```

#### ✅ Correct:

```javascript
function MyComponent() {
  const [data, setData] = useState([]); // ✅ inside a component
}
```

Or inside a custom Hook:

```javascript
function useCustomData() {
  const [data, setData] = useState([]);
  return data;
}
```

---

## Common Pitfalls and Mistakes

### ❌ 1. Calling Hooks Conditionally

Hooks must always run in the same order.
Calling a Hook conditionally (inside an if statement) breaks this rule and causes mismatched state assignments.

#### Example:

```javascript
if (show) {
  useEffect(() => {
    console.log("Effect runs"); // ❌ Hook may not run consistently
  }, []);
}
```

Always call Hooks unconditionally:

```javascript
useEffect(() => {
  if (show) console.log("Effect runs");
}, [show]);
```

---

### ❌ 2. Forgetting Dependency Arrays in `useEffect`, `useCallback`, or `useMemo`

Leaving the dependency array empty or incorrect can cause infinite loops or stale values.

#### Example (wrong):

```javascript
useEffect(() => {
  fetchData();
}); // ❌ runs after every render
```

#### Correct:

```javascript
useEffect(() => {
  fetchData();
}, []); // ✅ runs only once on mount
```

---

### ❌ 3. Using Hooks Inside Loops or Nested Functions

Hooks inside loops or nested functions cause inconsistent order during re-renders.

#### Wrong:

```javascript
for (let i = 0; i < 3; i++) {
  useState(i); // ❌ changes the hook order each loop
}
```

#### Right:

Hooks should always be declared directly inside the component body.

---

### ❌ 4. Not Using Functional Updates When State Depends on Previous Value

If your state update depends on the previous state, use the **functional form** to avoid stale values.

#### Wrong:

```javascript
setCount(count + 1);
setCount(count + 1); // ❌ may use stale 'count'
```

#### Correct:

```javascript
setCount(prev => prev + 1);
setCount(prev => prev + 1);
```

---

### ❌ 5. Creating Infinite Loops with `useEffect`

If you update state directly inside an effect without proper dependencies, it causes continuous re-renders.

#### Example:

```javascript
useEffect(() => {
  setCount(count + 1); // ❌ triggers rerender infinitely
});
```

#### Correct:

Add a dependency array or condition:

```javascript
useEffect(() => {
  setCount(c => c + 1);
}, []); // ✅ runs only once
```

---

## Summary Table

| Rule / Mistake                 | Description                                                   | Fix                                    |
| ------------------------------ | ------------------------------------------------------------- | -------------------------------------- |
| Call Hooks at top level        | Don’t call Hooks in loops, conditions, or nested functions    | Always call in same order              |
| Only call from React functions | Hooks work only in components or custom Hooks                 | Don’t use Hooks in normal JS functions |
| Correct dependency arrays      | Avoid missing dependencies in useEffect, useCallback, useMemo | Include all used variables             |
| Use functional updates         | Prevent stale values in multiple state updates                | Use setState(prev => ...)              |
| Avoid infinite loops           | Don’t trigger state updates unconditionally in effects        | Use dependency array or conditions     |

---

## Interview-Ready Answer

React Hooks follow two main rules:

1. Only call Hooks at the top level of your component or custom Hook.
2. Only call Hooks from React function components or custom Hooks.

These rules ensure React can correctly track Hook order and state between renders.
Common pitfalls include calling Hooks conditionally, forgetting dependency arrays, and causing infinite loops in effects.
Following these rules keeps components predictable, efficient, and bug-free.
