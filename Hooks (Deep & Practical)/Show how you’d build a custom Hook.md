# Show how you'd build a custom Hook

## Purpose

A reusable `useFetch` Hook that:

* Fetches JSON data from an API URL
* Exposes `data`, `error`, and `loading` states
* Supports cancellation with `AbortController`
* Avoids state updates after unmount
* Accepts a `options` object and a `deps` array to re-fetch when needed

---

## Hook implementation (JavaScript)

```javascript
import { useState, useEffect, useRef } from "react";

/**
 * useFetch
 * @param {string} url - resource URL
 * @param {object} [options] - fetch options (headers, method, body, etc.)
 * @param {Array} [deps=[]] - dependency array that triggers re-fetch when changed
 * @returns {{ data: any, loading: boolean, error: Error|null, refetch: Function }}
 */
export function useFetch(url, options = {}, deps = []) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(Boolean(url));
  const [error, setError] = useState(null);

  // keep a stable ref to the current abort controller and mounted state
  const abortRef = useRef(null);
  const mountedRef = useRef(true);
  const triggerRef = useRef(0);

  useEffect(() => {
    mountedRef.current = true;
    return () => {
      mountedRef.current = false;
      if (abortRef.current) {
        abortRef.current.abort();
      }
    };
  }, []);

  useEffect(() => {
    if (!url) {
      setData(null);
      setLoading(false);
      setError(null);
      return;
    }

    const controller = new AbortController();
    abortRef.current = controller;
    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal, ...options })
      .then(async res => {
        if (!res.ok) {
          const text = await res.text().catch(() => "");
          const err = new Error(`Fetch error: ${res.status} ${res.statusText} ${text}`);
          err.status = res.status;
          throw err;
        }
        return res.json();
      })
      .then(result => {
        if (!mountedRef.current) return;
        setData(result);
      })
      .catch(err => {
        if (!mountedRef.current) return;
        // Ignore abort errors
        if (err.name === "AbortError") return;
        setError(err);
      })
      .finally(() => {
        if (!mountedRef.current) return;
        setLoading(false);
      });

    // cleanup for this effect: abort the fetch if deps change or component unmounts
    return () => {
      controller.abort();
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [url, triggerRef.current, ...deps]);

  // manual refetch function
  const refetch = () => {
    triggerRef.current += 1;
    // force an update by changing triggerRef.current (effect depends on it)
  };

  return { data, loading, error, refetch };
}
```

---

## Usage example

```jsx
import React from "react";
import { useFetch } from "./useFetch";

function UsersList() {
  const { data, loading, error, refetch } = useFetch("https://jsonplaceholder.typicode.com/users");

  if (loading) return <p>Loading...</p>;
  if (error) return (
    <div>
      <p>Error: {error.message}</p>
      <button onClick={refetch}>Retry</button>
    </div>
  );

  return (
    <ul>
      {data.map(user => <li key={user.id}>{user.name} ({user.email})</li>)}
    </ul>
  );
}
```

---

## Notes & Best Practices

* Always include cleanup (AbortController) to avoid memory leaks and to cancel stale requests.
* Use the functional update pattern or refs for force-refetching instead of depending on object identity.
* Provide `deps` to allow consumers to re-run fetch when dependent values change (e.g., query params).
* Avoid putting non-serializable objects in `deps`. If an option object changes identity frequently, memoize it with `useMemo` in the consumer.
* Consider adding caching or deduplication (simple cache map keyed by URL+options) for production usage to avoid repeated network calls.
* In React 18+, state updates are batched automatically (including async), but still avoid updating state after unmount — the mountedRef guard prevents it.

---

## Variations & Enhancements

* **TypeScript**: Add generics to type `data` (`useFetch<T>(...)`).
* **Retries**: Add retry logic with exponential backoff.
* **Caching**: Add a shared cache (Map) to return cached data instantly and refresh in background.
* **Pagination**: Extend to support `page` and `limit` params and return `hasMore`.
* **Abort on param changes**: Current implementation aborts on deps change via cleanup.

---

## Interview-ready answer (short)

A custom Hook like `useFetch` encapsulates fetch logic, loading/error states, cancellation (AbortController), and cleanup. It returns `{ data, loading, error, refetch }`, keeps UI code clean, and centralizes reusable logic. Key considerations: add cleanup to avoid state updates after unmount, accept dependencies for re-fetching, and avoid stale closures by using refs or functional updates.

