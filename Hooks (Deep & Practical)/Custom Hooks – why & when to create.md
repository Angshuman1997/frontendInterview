# Custom Hooks — Why & When to Create

## What Are Custom Hooks

Custom Hooks are reusable functions in React that encapsulate logic using built-in Hooks such as `useState`, `useEffect`, or `useContext`.
They allow you to reuse stateful logic across multiple components without repeating code.

---

## Why Create Custom Hooks

### 1. Code Reuse

If multiple components use the same logic (e.g., fetching data, handling forms, event listeners), move that logic into a custom Hook instead of duplicating it.

### 2. Cleaner Components

Custom Hooks keep components focused on rendering UI while moving non-UI logic (like state management and side effects) into separate reusable functions.

### 3. Separation of Concerns

They promote cleaner, modular code by separating logic from UI, improving readability and testability.

### 4. Easier Maintenance

When shared logic changes, you update it in one place (the Hook), and all components benefit automatically.

---

## When to Create a Custom Hook

You should create a custom Hook when:

* The same state or side-effect logic is used in multiple components.
* A component’s logic grows too large and you want to simplify it.
* You want to abstract API calls, event listeners, timers, or form logic.
* You want to share reusable stateful behavior between components.

---

## Syntax and Naming Convention

* Custom Hooks are regular JavaScript functions.
* Their names **must start with `use`** (e.g., `useFetch`, `useForm`, `useWindowWidth`).
* They can call other Hooks inside them.

---

## Example 1 — `useWindowWidth`

```javascript
import { useState, useEffect } from "react";

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return width;
}

// Usage
function App() {
  const width = useWindowWidth();
  return <p>Window width: {width}px</p>;
}
```

---

## Example 2 — `useFetch`

```javascript
import { useState, useEffect } from "react";

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
}

// Usage
function Users() {
  const { data, loading, error } = useFetch("https://api.example.com/users");

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return <ul>{data.map(user => <li key={user.id}>{user.name}</li>)}</ul>;
}
```

---

## Rules of Custom Hooks

* Must start with the word **`use`**.
* Can only call other Hooks inside them.
* Must follow the **Rules of Hooks**:

  * Only call Hooks inside React functions (not loops, conditions, or nested functions).
  * Only call Hooks at the top level.

---

## Interview-Ready Answer

A **custom Hook** is a reusable function that encapsulates stateful or side-effect logic using React’s built-in Hooks.
You create custom Hooks when multiple components share the same logic or when you want to separate non-UI logic from the component.
They simplify components, improve code reuse, and must start with the word `use` to follow React’s Hook rules.
