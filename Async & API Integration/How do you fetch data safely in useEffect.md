# How do you fetch data safely in useEffect

## Short summary

Always perform fetches inside `useEffect` using an inner async function, handle loading/error states, cancel stale requests (via `AbortController` or Axios signal), avoid updating state after unmount, and include correct dependencies. For complex caching/retries use React Query / SWR.

---

## Principles

* Don’t make the `useEffect` callback `async` (it would return a Promise).
* Use an inner `async` function or an IIFE.
* Cancel pending requests on cleanup (prevent state updates after unmount).
* Guard against race conditions (older slower requests overwriting newer results).
* Keep dependency arrays correct (or intentionally memoize inputs).
* Consider using a library (React Query / SWR) for caching, retries, background refresh.

---

## Minimal safe pattern (fetch + AbortController)

```jsx
import { useEffect, useState } from "react";

function useUser(userId) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(Boolean(userId));
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!userId) {
      setData(null);
      setLoading(false);
      return;
    }

    const controller = new AbortController();
    const signal = controller.signal;

    async function fetchUser() {
      setLoading(true);
      setError(null);
      try {
        const res = await fetch(`/api/users/${userId}`, { signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const json = await res.json();
        setData(json);
      } catch (err) {
        if (err.name === "AbortError") {
          // request was cancelled — ignore
        } else {
          setError(err);
        }
      } finally {
        setLoading(false);
      }
    }

    fetchUser();

    return () => {
      controller.abort(); // cancel on unmount or when userId changes
    };
  }, [userId]); // re-run when userId changes

  return { data, loading, error };
}
```

---

## Avoiding race conditions (request id / latest-only)

If multiple requests can be inflight and earlier ones may resolve later, track a sequence id:

```jsx
useEffect(() => {
  let isLatest = true; // local flag ensures only latest write state
  const controller = new AbortController();

  (async () => {
    try {
      const res = await fetch(url, { signal: controller.signal });
      const json = await res.json();
      if (!isLatest) return;
      setData(json);
    } catch (err) {
      if (err.name !== "AbortError") setError(err);
    }
  })();

  return () => {
    isLatest = false;
    controller.abort();
  };
}, [url]);
```

Or increment a ref-based requestId:

```jsx
const requestRef = useRef(0);
useEffect(() => {
  const id = ++requestRef.current;
  const controller = new AbortController();
  (async () => {
    const res = await fetch(url, { signal: controller.signal });
    const json = await res.json();
    if (requestRef.current === id) setData(json);
  })();
  return () => controller.abort();
}, [url]);
```

---

## Axios example (with AbortController, v1.2+)

```jsx
import axios from "axios";
import { useEffect, useState } from "react";

useEffect(() => {
  const controller = new AbortController();

  axios.get(url, { signal: controller.signal })
    .then(res => setData(res.data))
    .catch(err => {
      if (err.name === "CanceledError" || err.name === "AbortError") return;
      setError(err);
    });

  return () => controller.abort();
}, [url]);
```

---

## Debounce / frequent queries (search box)

Combine debouncing and cancellation:

```jsx
const debouncedQuery = useDebounce(query, 400);
useEffect(() => {
  if (!debouncedQuery) return;
  const controller = new AbortController();

  (async () => {
    const res = await fetch(`/search?q=${debouncedQuery}`, { signal: controller.signal });
    setResults(await res.json());
  })();

  return () => controller.abort();
}, [debouncedQuery]);
```

---

## Common pitfalls & tips

* Don’t mark the effect callback `async` — use an inner async function.
* Always clean up with `controller.abort()` to prevent "setState on unmounted component".
* Include all referenced variables in the dependency array, or memoize them (e.g., `options` objects with `useMemo`).
* Avoid disabling `react-hooks/exhaustive-deps` unless you know what you’re doing.
* Handle non-2xx fetch responses (`res.ok`) — Fetch doesn’t throw automatically.
* Prefer functional updates for state when needed: `setState(prev => ...)`.
* For complex fetching needs (caching, retries, background refresh, pagination), use React Query or SWR — they handle cancellation, deduping, retries, and cache for you.

---

## Interview-ready answer (short)

Fetch data inside `useEffect` by calling an inner async function and passing correct dependencies. Use `AbortController` (or Axios signals) in the effect cleanup to cancel pending requests and avoid state updates after unmount. Guard against race conditions (e.g., using a request id or ignoring stale results). For advanced needs, use a library like React Query or SWR that manages cancellation, caching, and retries.
