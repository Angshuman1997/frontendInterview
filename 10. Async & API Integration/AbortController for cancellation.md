# AbortController for cancellation

## Overview

When working with asynchronous requests (like `fetch` or Axios) in React, a common issue occurs when **a component unmounts before the request completes**.
If you try to update state after unmount, React throws a warning:

> “Can’t perform a React state update on an unmounted component.”

To prevent this and improve resource management, you can use the **AbortController API** to **cancel ongoing fetch requests** when a component unmounts or when a new request starts.

---

## 1. What is AbortController?

**AbortController** is a built-in browser API that allows you to **abort (cancel)** one or more web requests — typically used with the `fetch()` API.

It provides:

* A **controller** object used to signal cancellation.
* A **signal** object passed into `fetch()` to link the request to that controller.

---

## 2. Basic Syntax

```javascript
const controller = new AbortController();
const signal = controller.signal;

fetch("https://api.example.com/data", { signal })
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => {
    if (err.name === "AbortError") {
      console.log("Request aborted");
    } else {
      console.error("Fetch error:", err);
    }
  });

// Cancel the request
controller.abort();
```

✅ Once `controller.abort()` is called:

* The fetch request is stopped immediately.
* A special `AbortError` is thrown.
* You can handle it gracefully in your `catch` block.

---

## 3. Using AbortController with `useEffect`

In React, you typically make API calls inside a `useEffect`.
To prevent memory leaks, **cancel the request** when the component unmounts or before the next effect runs.

### ✅ Example — Safe API Fetch with Cleanup

```javascript
import { useEffect, useState } from "react";

function UserData({ userId }) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    const { signal } = controller;

    async function fetchUser() {
      try {
        const res = await fetch(`https://jsonplaceholder.typicode.com/users/${userId}`, { signal });
        if (!res.ok) throw new Error("Network response was not ok");
        const json = await res.json();
        setData(json);
      } catch (err) {
        if (err.name === "AbortError") {
          console.log("Fetch aborted");
        } else {
          setError(err.message);
        }
      }
    }

    fetchUser();

    // Cleanup: abort request when component unmounts or before re-run
    return () => controller.abort();
  }, [userId]);

  if (error) return <p>Error: {error}</p>;
  if (!data) return <p>Loading...</p>;
  return <p>User: {data.name}</p>;
}
```

✅ **Explanation:**

* A new `AbortController` is created for each render.
* The signal is passed into the `fetch` call.
* When the component unmounts or `userId` changes, the cleanup function calls `controller.abort()`.
* This prevents unnecessary state updates or console warnings.

---

## 4. Using AbortController with Axios

Axios doesn’t use `AbortController` directly but has a similar mechanism using **cancel tokens** (or supports AbortController in newer versions).

### Example — Axios with AbortController

```javascript
import axios from "axios";
import { useEffect, useState } from "react";

function UserProfile({ id }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    axios.get(`https://jsonplaceholder.typicode.com/users/${id}`, {
      signal: controller.signal,
    })
    .then((res) => setUser(res.data))
    .catch((err) => {
      if (err.name === "CanceledError") {
        console.log("Request canceled");
      } else {
        console.error(err);
      }
    });

    return () => controller.abort();
  }, [id]);

  return user ? <p>{user.name}</p> : <p>Loading...</p>;
}
```

✅ Works with Axios v1.2+ (which supports AbortController signals).

---

## 5. Why Use AbortController in React?

| Problem                                                   | AbortController Solution                         |
| --------------------------------------------------------- | ------------------------------------------------ |
| Component unmounts before request finishes                | Cancels the pending request                      |
| Multiple rapid API calls (e.g., search input)             | Cancels older requests before starting a new one |
| Memory leaks or “setState on unmounted component” warning | Prevented                                        |
| Unnecessary network usage                                 | Reduced                                          |
| Better performance and user experience                    | Improved                                         |

---

## 6. Real-World Use Case — Search Debounce + Abort

```javascript
import { useState, useEffect } from "react";

function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (!query) return;

    const controller = new AbortController();

    const timer = setTimeout(async () => {
      try {
        const res = await fetch(`https://api.example.com/search?q=${query}`, {
          signal: controller.signal,
        });
        const data = await res.json();
        setResults(data.results);
      } catch (err) {
        if (err.name === "AbortError") {
          console.log("Previous request aborted");
        }
      }
    }, 500); // debounce delay

    return () => {
      clearTimeout(timer);
      controller.abort(); // abort on new search or unmount
    };
  }, [query]);

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} placeholder="Search..." />
      <ul>{results.map((r) => <li key={r.id}>{r.name}</li>)}</ul>
    </div>
  );
}
```

✅ Efficient, clean, and cancelable API calls for live searches.

---

## 7. Interview-Ready Answer

`AbortController` is a browser API used to **cancel ongoing asynchronous requests** like `fetch()`.
In React, it’s commonly used inside `useEffect` to prevent state updates on unmounted components or when a new request is triggered.
You create a controller, pass its `signal` to the request, and call `controller.abort()` in the cleanup function to cancel it.
This helps prevent memory leaks, improve performance, and ensure only relevant requests are processed.
