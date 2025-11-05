# Debounce - Throttle for input handling

## Overview

When handling user input or scroll/resize events in React, repeatedly calling functions (like API requests or state updates) on **every keystroke or event** can lead to performance issues.

To prevent this, we use **Debouncing** and **Throttling** — two optimization techniques that control how frequently a function is executed.

---

## 1. Debouncing — Wait Until User Stops Typing

### Definition

**Debouncing** ensures that a function executes **only after a specified delay** has passed **since the last time it was invoked**.

If the function is triggered again within that delay, the timer resets.

✅ Best for:

* Input search boxes
* Form validations
* Auto-suggest fields
* Window resize handlers

---

### Example — Debounce Concept

```
User types: "React"
Keystrokes: R | e | a | c | t
Calls API: Only once, after user stops typing for X ms
```

---

### React Example — Debounced Search

```javascript
import { useState, useEffect } from "react";

function DebouncedSearch() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (!query) return;

    const timer = setTimeout(async () => {
      const res = await fetch(`https://api.example.com/search?q=${query}`);
      const data = await res.json();
      setResults(data.results);
    }, 500); // 500ms debounce delay

    return () => clearTimeout(timer); // cleanup previous timer
  }, [query]);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <ul>{results.map((r) => <li key={r.id}>{r.name}</li>)}</ul>
    </div>
  );
}
```

✅ Debounce ensures the API is called only after the user stops typing for 500ms.
✅ Prevents multiple unnecessary API calls.

---

### Utility Function — Custom Debounce Hook

```javascript
import { useEffect, useState } from "react";

export function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

Usage:

```javascript
const debouncedQuery = useDebounce(query, 500);

useEffect(() => {
  if (!debouncedQuery) return;
  fetchData(debouncedQuery);
}, [debouncedQuery]);
```

---

## 2. Throttling — Execute at Controlled Intervals

### Definition

**Throttling** ensures that a function executes **at most once every X milliseconds**, no matter how many times it’s triggered.

✅ Best for:

* Scroll or resize events
* Dragging, mouse movement
* Continuous button clicks
* Infinite scrolling / progress tracking

---

### Example — Throttle Concept

```
User scrolls continuously → Throttle ensures callback runs every 200ms,
not on every pixel of movement.
```

---

### React Example — Throttled Scroll Event

```javascript
import { useEffect } from "react";

function useThrottle(callback, delay) {
  let lastCall = 0;

  return (...args) => {
    const now = new Date().getTime();
    if (now - lastCall >= delay) {
      lastCall = now;
      callback(...args);
    }
  };
}

function ThrottledScroll() {
  useEffect(() => {
    const handleScroll = useThrottle(() => {
      console.log("Scroll position:", window.scrollY);
    }, 300);

    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, []);

  return <div style={{ height: "200vh" }}>Scroll Down</div>;
}
```

✅ Throttle ensures `handleScroll` fires once every 300ms instead of on every scroll event.
✅ Improves performance and prevents excessive re-renders.

---

## 3. Debounce vs Throttle — Key Difference

| Feature                    | **Debounce**                                                       | **Throttle**                                            |
| -------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------- |
| **Definition**             | Delays execution until after a specified time since the last event | Executes function at most once every specified interval |
| **Best for**               | Search inputs, resize, validation                                  | Scroll, drag, continuous clicks                         |
| **Frequency of execution** | Once after the event stops firing                                  | Periodically during event firing                        |
| **Example Delay**          | “Wait until user stops typing for 500ms”                           | “Run every 300ms while scrolling”                       |
| **Use Case Example**       | Auto-suggest input box                                             | Window scroll tracking                                  |

---

## 4. Example — Using Lodash for Debounce & Throttle

The **lodash** library provides production-ready debounce and throttle utilities.

### Debounce with Lodash

```javascript
import { debounce } from "lodash";
import { useMemo } from "react";

function SearchInput() {
  const fetchResults = (value) => console.log("API Call:", value);

  const debouncedSearch = useMemo(() => debounce(fetchResults, 500), []);

  return <input onChange={(e) => debouncedSearch(e.target.value)} />;
}
```

### Throttle with Lodash

```javascript
import { throttle } from "lodash";

function ScrollTracker() {
  useEffect(() => {
    const throttled = throttle(() => console.log(window.scrollY), 300);
    window.addEventListener("scroll", throttled);
    return () => window.removeEventListener("scroll", throttled);
  }, []);

  return <div style={{ height: "200vh" }}>Scroll to see throttling</div>;
}
```

✅ Lodash handles cleanup and edge cases internally.
✅ Recommended for production use.

---

## 5. When to Use Which

| Scenario                      | Recommended  |
| ----------------------------- | ------------ |
| API search suggestions        | **Debounce** |
| Window resizing events        | **Debounce** |
| Scroll position tracking      | **Throttle** |
| Infinite scroll loading       | **Throttle** |
| Input validation after typing | **Debounce** |
| Limiting button click spam    | **Throttle** |

---

## 6. Interview-Ready Answer

**Debounce** delays function execution until after a specified time has passed since the last event — ideal for search inputs and form validations.
**Throttle** ensures a function executes at most once every set interval — ideal for scroll, drag, and resize events.

In React, debouncing prevents excessive API calls from input fields, while throttling ensures smooth performance for continuous actions like scrolling. Both improve performance and user experience by limiting unnecessary function executions.
