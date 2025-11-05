# Fetch - Axios with useEffect

## Overview

In React, data fetching is most commonly done inside the `useEffect` hook to ensure that the API call executes **after** the component mounts (or when dependencies change).
Two popular ways to make API requests are:

* The **native Fetch API** (built into browsers)
* The **Axios library** (third-party, promise-based HTTP client)

Both can be used inside `useEffect`, but they differ in features and ease of use.

---

## 1. Using **Fetch API** with `useEffect`

### Example — Basic Data Fetch with `fetch()`

```javascript
import { useEffect, useState } from "react";

function UsersList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function fetchUsers() {
      try {
        const res = await fetch("https://jsonplaceholder.typicode.com/users");

        if (!res.ok) {
          throw new Error("Network response was not ok");
        }

        const data = await res.json();
        setUsers(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    fetchUsers();
  }, []); // run once on mount

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

✅ **Explanation**

* `fetch()` returns a Promise.
* You must manually handle non-2xx responses with `if (!res.ok) throw Error()`.
* You must call `.json()` to parse the body.
* Use `try...catch` for error handling.
* The `finally` block ensures the loading state is reset regardless of success/failure.

---

### Example — Fetch with **AbortController** (Safe Cleanup)

```javascript
useEffect(() => {
  const controller = new AbortController();

  async function fetchData() {
    try {
      const res = await fetch("https://jsonplaceholder.typicode.com/posts", {
        signal: controller.signal,
      });
      const data = await res.json();
      setPosts(data);
    } catch (err) {
      if (err.name === "AbortError") {
        console.log("Request canceled");
      } else {
        console.error(err);
      }
    }
  }

  fetchData();

  return () => controller.abort(); // Cleanup on unmount
}, []);
```

✅ Prevents state updates on unmounted components.
✅ Recommended for all async calls inside `useEffect`.

---

## 2. Using **Axios** with `useEffect`

### Example — Basic Data Fetch with Axios

```javascript
import { useEffect, useState } from "react";
import axios from "axios";

function PostsList() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function fetchPosts() {
      try {
        const res = await axios.get("https://jsonplaceholder.typicode.com/posts");
        setPosts(res.data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    fetchPosts();
  }, []);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <ul>
      {posts.slice(0, 5).map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

✅ **Advantages over Fetch**

* Automatically parses JSON response — no need for `.json()`.
* Better error handling (rejects on non-2xx status codes).
* Supports request cancellation, interceptors, and timeouts.

---

### Example — Axios with **AbortController**

```javascript
useEffect(() => {
  const controller = new AbortController();

  axios
    .get("https://jsonplaceholder.typicode.com/comments", {
      signal: controller.signal,
    })
    .then((res) => setComments(res.data))
    .catch((err) => {
      if (axios.isCancel(err)) {
        console.log("Request canceled");
      } else {
        console.error(err);
      }
    });

  return () => controller.abort();
}, []);
```

✅ Axios (v1.2+) supports the `AbortController` signal, just like fetch.
✅ Prevents memory leaks and canceled request warnings.

---

## 3. Key Differences — Fetch vs Axios

| Feature                      | **Fetch API**                      | **Axios**                               |
| ---------------------------- | ---------------------------------- | --------------------------------------- |
| **Library**                  | Native (built-in)                  | External (requires install)             |
| **JSON Parsing**             | Must call `.json()` manually       | Automatically parses JSON               |
| **Error Handling**           | Must handle manually with `res.ok` | Automatically rejects for non-2xx       |
| **Timeout Support**          | Manual implementation              | Built-in (`timeout` option)             |
| **Interceptors**             | Not available                      | Yes (for requests and responses)        |
| **Request Cancellation**     | Supported with `AbortController`   | Supported via `AbortController` (v1.2+) |
| **Upload/Download Progress** | Manual                             | Built-in progress tracking              |
| **Ease of Use**              | Simpler for small apps             | More robust for large apps              |

---

## 4. Axios with Interceptors (Advanced)

Interceptors are powerful features for **global request/response management** (e.g., adding auth tokens, handling errors).

```javascript
import axios from "axios";

// Request interceptor
axios.interceptors.request.use(
  (config) => {
    config.headers.Authorization = "Bearer " + localStorage.getItem("token");
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
axios.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      console.error("Unauthorized! Logging out...");
    }
    return Promise.reject(error);
  }
);
```

✅ Commonly used for authentication, error logging, and global loaders.

---

## 5. Best Practices for Fetching Data with `useEffect`

1. **Always handle loading and error states** — Avoid silent failures.
2. **Use cleanup functions** (with `AbortController`) to prevent memory leaks.
3. **Avoid using async directly in `useEffect`** — define async functions inside it.
4. **Add dependencies carefully** — to avoid infinite loops.
5. **Cache or memoize results** if the data is reused.
6. **Use libraries like React Query or SWR** for advanced caching and refetching patterns.

---

## 6. Example — Complete Comparison (Side-by-Side)

| Aspect          | Fetch                                | Axios                                |
| --------------- | ------------------------------------ | ------------------------------------ |
| Installation    | None (built-in)                      | `npm install axios`                  |
| Data Fetch      | `fetch(url).then(res => res.json())` | `axios.get(url)`                     |
| Error Handling  | Must check `res.ok`                  | Automatically handled                |
| JSON Parsing    | Manual                               | Automatic                            |
| Cancellation    | `AbortController`                    | `AbortController` or Cancel Token    |
| Interceptors    | ❌                                    | ✅                                    |
| Upload Progress | ❌                                    | ✅                                    |
| Use Case        | Simple APIs                          | Complex APIs, authenticated requests |

---

## 7. Interview-Ready Answer

In React, API calls are typically made inside the `useEffect` hook to fetch data after the component mounts.
The **Fetch API** is built-in and minimal but requires manual JSON parsing and error handling.
**Axios**, on the other hand, is a promise-based HTTP client that automatically parses JSON, handles errors more elegantly, and supports advanced features like interceptors, request cancellation, and timeouts.

Use **Fetch** for small, simple apps or quick prototypes, and **Axios** for larger applications that require cleaner syntax, better error handling, or global API management.
