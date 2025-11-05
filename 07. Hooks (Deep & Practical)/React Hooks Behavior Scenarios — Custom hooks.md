# React Hooks Behavior Scenarios — Custom Hooks

## Overview

**Custom Hooks** are user-defined functions in React that let you **reuse stateful logic** across multiple components.
They are not a new feature — they follow the same **rules of Hooks** (`useState`, `useEffect`, etc.), but allow encapsulating and reusing complex behaviors in a clean, composable way.

A **Custom Hook**:

* Always starts with `use` (e.g., `useFetch`, `useForm`, `useTheme`).
* Can call other Hooks internally.
* Helps remove duplicate logic and improve readability.

---

## 1️⃣ Basic Structure

```jsx
function useCustomHook() {
  // useState, useEffect, etc.
  return something;
}
```

✅ **Rule:**
Custom Hooks are **regular functions**, but they must follow **React’s Hook rules** (only called inside components or other hooks).

---

## 2️⃣ Example: useFetch (Reusable Data Fetching Logic)

```jsx
import { useState, useEffect } from "react";

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let isMounted = true;
    setLoading(true);

    fetch(url)
      .then((res) => res.json())
      .then((data) => isMounted && setData(data))
      .catch((err) => isMounted && setError(err))
      .finally(() => isMounted && setLoading(false));

    return () => {
      isMounted = false;
    };
  }, [url]);

  return { data, loading, error };
}
```

✅ **Usage:**

```jsx
function Users() {
  const { data, loading, error } = useFetch("/api/users");

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error!</p>;
  return <ul>{data.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

✅ **Explanation:**
Encapsulates repetitive `useEffect` + `useState` logic for fetching data, keeps UI code clean.

---

## 3️⃣ Example: useLocalStorage (Persistent State)

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}
```

✅ **Usage:**

```jsx
const [theme, setTheme] = useLocalStorage("theme", "light");
```

✅ **Explanation:**
Manages persistent state synced with `localStorage` — reusable for settings, tokens, etc.

---

## 4️⃣ Example: usePrevious (Track Previous Value)

```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

✅ **Usage:**

```jsx
const prevCount = usePrevious(count);
```

✅ **Explanation:**
Stores the previous value across renders. Useful for comparison logic and debugging.

---

## 5️⃣ Example: useDebounce (Delay Updates)

```jsx
function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

✅ **Usage:**

```jsx
const debouncedSearch = useDebounce(searchTerm, 300);
```

✅ **Explanation:**
Delays state updates to avoid firing expensive operations (like API calls) too frequently.

---

## 6️⃣ Example: useToggle (Boolean State Helper)

```jsx
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = () => setValue((v) => !v);
  return [value, toggle];
}
```

✅ **Usage:**

```jsx
const [isOpen, toggleOpen] = useToggle();
```

✅ **Explanation:**
Simplifies repetitive boolean toggle logic for modals, dropdowns, etc.

---

## 7️⃣ Example: useOnClickOutside (Detect Click Outside Element)

```jsx
function useOnClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (event) => {
      if (!ref.current || ref.current.contains(event.target)) return;
      handler(event);
    };
    document.addEventListener("mousedown", listener);
    return () => document.removeEventListener("mousedown", listener);
  }, [ref, handler]);
}
```

✅ **Usage:**

```jsx
const ref = useRef();
useOnClickOutside(ref, () => setOpen(false));
```

✅ **Explanation:**
Closes modals or dropdowns when clicking outside the element.

---

## 8️⃣ Example: useWindowSize (Responsive Layouts)

```jsx
function useWindowSize() {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });

  useEffect(() => {
    const handleResize = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return size;
}
```

✅ **Usage:**

```jsx
const { width, height } = useWindowSize();
```

✅ **Explanation:**
Updates component state whenever the browser window size changes — perfect for responsive UIs.

---

## 9️⃣ Example: useDocumentTitle

```jsx
function useDocumentTitle(title) {
  useEffect(() => {
    document.title = title;
  }, [title]);
}
```

✅ **Usage:**

```jsx
useDocumentTitle("Dashboard");
```

✅ **Explanation:**
Encapsulates side-effect of updating document title based on page or component state.

---

## 🔟 Example: useOnlineStatus

```jsx
function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setOnline(true);
    const handleOffline = () => setOnline(false);

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);
    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return online;
}
```

✅ **Usage:**

```jsx
const online = useOnlineStatus();
```

✅ **Explanation:**
Tracks internet connectivity — useful for showing offline banners or retry prompts.

---

## 11️⃣ Example: useTimeout (Delayed Action)

```jsx
function useTimeout(callback, delay) {
  useEffect(() => {
    if (!delay) return;
    const timer = setTimeout(callback, delay);
    return () => clearTimeout(timer);
  }, [callback, delay]);
}
```

✅ **Usage:**

```jsx
useTimeout(() => console.log("Hello after 2s"), 2000);
```

✅ **Explanation:**
Runs a function once after a specified delay — good for notifications or animations.

---

## 12️⃣ Example: useInterval (Repeated Action)

```jsx
function useInterval(callback, delay) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return;
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

✅ **Usage:**

```jsx
useInterval(() => setCount((c) => c + 1), 1000);
```

✅ **Explanation:**
Like `setInterval`, but React-safe — avoids stale closures.

---

## 13️⃣ Example: useTheme (Global Context-Based Hook)

```jsx
const ThemeContext = createContext();

function useTheme() {
  return useContext(ThemeContext);
}
```

✅ **Usage:**

```jsx
const { theme, toggleTheme } = useTheme();
```

✅ **Explanation:**
Wraps `useContext` for consistent and readable theme access across multiple components.

---

## 14️⃣ Example: usePreviousState (Compare Old vs New)

```jsx
function usePreviousState(value) {
  const ref = useRef(value);
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

✅ **Usage:**

```jsx
const prevValue = usePreviousState(value);
```

✅ **Explanation:**
Helps track how state has changed across renders — used in performance monitoring or condition-based logic.

---

## 15️⃣ Best Practices for Custom Hooks

✅ **1. Start with `use`**
The name should always begin with `use` (e.g., `useForm`, `useScrollPosition`) — this lets React validate Hook rules.

✅ **2. Encapsulate reusable logic**
Don’t make Hooks for trivial one-time logic. Aim to reduce repetitive side effects or state management patterns.

✅ **3. Keep side effects scoped**
Clean up timers, listeners, or subscriptions to avoid memory leaks.

✅ **4. Return clear, minimal APIs**
Return only what consumers need — e.g., `[value, setter]` or `{ state, actions }`.

✅ **5. Compose Hooks**
Custom Hooks can call other Hooks — e.g., `useFetch` + `useDebounce` to build `useDebouncedFetch`.

✅ **6. Test individually**
Each Hook should be testable in isolation using libraries like `@testing-library/react-hooks`.

---

## Summary Table

| #  | Scenario         | Hook Example        | Use Case          | Key Benefit             |
| -- | ---------------- | ------------------- | ----------------- | ----------------------- |
| 1  | Reusable logic   | Generic template    | Base structure    | Reuse patterns          |
| 2  | API fetch        | `useFetch`          | Data fetching     | Cleaner effects         |
| 3  | Persistent state | `useLocalStorage`   | Save settings     | State persistence       |
| 4  | Track previous   | `usePrevious`       | Compare state     | Debug, animations       |
| 5  | Delay updates    | `useDebounce`       | Search inputs     | Performance             |
| 6  | Boolean toggles  | `useToggle`         | Modals, toggles   | Simplified logic        |
| 7  | Outside click    | `useOnClickOutside` | Dropdowns         | UX control              |
| 8  | Responsive       | `useWindowSize`     | Layouts           | Responsive design       |
| 9  | Title update     | `useDocumentTitle`  | Page title        | Side-effect management  |
| 10 | Connectivity     | `useOnlineStatus`   | Offline detection | User feedback           |
| 11 | Timeout          | `useTimeout`        | Delay effect      | Scheduled logic         |
| 12 | Interval         | `useInterval`       | Counters          | Controlled looping      |
| 13 | Theme            | `useTheme`          | Global context    | Consistent global state |
| 14 | Prev state       | `usePreviousState`  | Compare changes   | Debugging               |
| 15 | Best practices   | -                   | Design principles | Maintainability         |

---

## Interview-Ready Answer

A **Custom Hook** is a function that encapsulates and reuses stateful logic across multiple React components.
It can use built-in Hooks like `useState`, `useEffect`, or `useContext` inside, and returns state or functions that can be used elsewhere.

They’re used to:

* Avoid repeating logic (like data fetching or local storage handling).
* Improve readability and testability.
* Create shared utilities (e.g., `useAuth`, `useTheme`, `useFetch`).

**Key rules:**

1. Must start with `use`.
2. Follow the Rules of Hooks.
3. Keep it focused on a single concern.

Custom Hooks help **abstract complex logic**, making your React app more **maintainable, scalable, and DRY**.

---

✅ **Key Takeaway:**

> Custom Hooks = “Reusable Logic Containers.”
> Encapsulate repetitive side effects or behaviors using built-in Hooks for clean, composable React code.
