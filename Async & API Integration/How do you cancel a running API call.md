# How do you cancel a running API call

## Overview

In React applications, especially when fetching data inside `useEffect`, a component might **unmount or re-render** before an API call finishes.
If the request completes after unmounting, it may try to update state — leading to warnings like:

> “Can’t perform a React state update on an unmounted component.”

To prevent this and save resources, we **cancel the running API call** before the component unmounts or before a new request starts.

---

## 1. Using `AbortController` (Native Fetch API)

### Step-by-Step Explanation

1. Create an `AbortController` instance.
2. Pass its `signal` to the fetch request.
3. Call `controller.abort()` to cancel the request.
4. Handle the `AbortError` in your `catch` block.

---

### ✅ Example — Cancel Fetch Request in `useEffect`

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
        const res = await fetch(`https://jsonplaceholder.typicode.com/users/${userId}`, {
          signal, // attach abort signal
        });

        if (!res.ok) throw new Error("Network response was not ok");

        const result = await res.json();
        setData(result);
      } catch (err) {
        if (err.name === "AbortError") {
          console.log("Fetch aborted");
        } else {
          setError(err.message);
        }
      }
    }

    fetchUser();

    // Cleanup: abort request on unmount or dependency change
    return () => controller.abort();
  }, [userId]);

  if (error) return <p>Error: {error}</p>;
  if (!data) return <p>Loading...</p>;
  return <p>User: {data.name}</p>;
}
```

✅ **Explanation**

* The fetch request is tied to an `AbortController`.
* When the component unmounts or `userId` changes, the cleanup function runs and aborts the pending request.
* Prevents state updates on an unmounted component.

---

## 2. Using Axios (v1.2+ supports AbortController)

### ✅ Example — Cancel Axios Request

```javascript
import { useEffect, useState } from "react";
import axios from "axios";

function Posts() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    axios
      .get("https://jsonplaceholder.typicode.com/posts", {
        signal: controller.signal,
      })
      .then((res) => setPosts(res.data))
      .catch((err) => {
        if (err.name === "CanceledError") {
          console.log("Request canceled");
        } else {
          console.error(err);
        }
      });

    return () => controller.abort();
  }, []);

  return (
    <ul>
      {posts.slice(0, 5).map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

✅ Works the same way as Fetch — Axios internally respects the `AbortController.signal`.

---

### 🧠 Older Axios Versions (Before v1.2)

If you’re using Axios < v1.2, you can use **Cancel Tokens** (deprecated now but still seen in legacy projects).

```javascript
useEffect(() => {
  const source = axios.CancelToken.source();

  axios
    .get("https://api.example.com/data", { cancelToken: source.token })
    .then((res) => setData(res.data))
    .catch((err) => {
      if (axios.isCancel(err)) {
        console.log("Request canceled", err.message);
      }
    });

  return () => source.cancel("Operation canceled by user.");
}, []);
```

✅ Replace with `AbortController` for modern implementations.

---

## 3. Why Cancel API Calls?

| Problem                                        | Solution with Cancellation                    |
| ---------------------------------------------- | --------------------------------------------- |
| Component unmounts while API is still fetching | Prevents unwanted `setState()` calls          |
| User navigates away mid-request                | Frees resources, cancels network call         |
| User types quickly (e.g., search box)          | Cancels old request before sending new one    |
| Prevents race conditions                       | Ensures only the latest request updates state |

---

## 4. Real-World Example — Search with Cancel

```javascript
import { useEffect, useState } from "react";

function SearchBox() {
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
        if (err.name === "AbortError") console.log("Request aborted");
      }
    }, 500);

    return () => {
      clearTimeout(timer);
      controller.abort(); // cancel if user types again quickly
    };
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

✅ Only the latest search request is active.
✅ Older requests are automatically canceled if the user types quickly.

---

## 5. Summary Table

| Method             | Works With                    | Cancel API           | Cleanup Location    |
| ------------------ | ----------------------------- | -------------------- | ------------------- |
| `AbortController`  | `fetch()`, Axios v1.2+        | `controller.abort()` | `useEffect` cleanup |
| Axios Cancel Token | Axios < v1.2                  | `source.cancel()`    | `useEffect` cleanup |
| React Query / SWR  | Fetch/Axios (library-managed) | Automatic            | Built-in            |

---

## 6. Interview-Ready Answer

You can cancel a running API call in React using the **AbortController API**.
You create an `AbortController`, pass its `signal` to the request, and call `controller.abort()` in the cleanup function of `useEffect`.
This prevents state updates after unmount and avoids memory leaks.
Axios (v1.2+) also supports `AbortController`, while older versions use cancel tokens.
This approach is essential for **safe, efficient, and responsive** API integrations in React.
