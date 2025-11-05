# Loading - error UI patterns

## Overview

Loading and error states are essential UX during async operations (fetching data, submitting forms, etc.). Good patterns make apps feel responsive, reduce user frustration, and provide clear recovery paths.

---

## Basic Principles

* Show immediate feedback for user actions.
* Prefer contextual (in-place) feedback over global where possible.
* Provide affordances to retry or cancel on errors.
* Avoid blocking the whole UI unless absolutely necessary.
* Handle accessibility (ARIA live regions, roles).
* Cancel stale requests to avoid flicker and incorrect states.

---

## Common Patterns

### 1. Global Loading Indicator

Use when the whole app is waiting (initial app boot, auth).

* Top-level spinner or progress bar (e.g., at top of page).
* Good for blocking flows like initial boot or major navigation.
* Implementation: single piece of state in app/store or React Query global state.

Example (pseudo):

```jsx
{isAppLoading && <TopProgressBar />}
```

---

### 2. Local / Component-level Loading

Prefer for fetching component data (lists, cards).

* Show spinner/placeholders inside the component area.
* Keep surrounding UI interactive.

Example:

```jsx
function UsersList() {
  if (loading) return <PlaceholderList />;
  if (error) return <InlineError message={error.message} onRetry={refetch} />;
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

---

### 3. Skeleton Screens (Preferred over spinners)

* Show a lightweight mock of the content layout (skeletons) instead of a spinner.
* Improves perceived performance.
* Use for content-heavy areas (cards, lists, feeds).

Example:

```jsx
{loading ? <UserCardSkeleton /> : <UserCard data={user} />}
```

---

### 4. Inline Error Messages & Retry

* Display concise inline error in place of content with a clear retry action.
* Avoid cryptic messages; show friendly text + retry button.
* For non-fatal errors, allow partial UI to render.

Example:

```jsx
<InlineError
  title="Failed to load comments"
  message="Please check your connection."
  actionText="Retry"
  onAction={refetch}
/>
```

---

### 5. Toast Notifications for Non-blocking Errors

* Use toast for background errors (auto-save failed) or transient messages.
* Provide link or action in the toast to retry or open details.

---

### 6. Optimistic UI (for writes)

* Update UI immediately assuming success, then rollback on failure.
* Use when latency would harm UX (likes, inline edits).
* Always provide visual confirmation and rollback path.

Pattern:

1. Save previous state.
2. Update state locally (optimistic).
3. Send request.
4. On error, restore previous state and show error.

Example:

```js
const old = items;
setItems(items => items.filter(i => i.id !== id)); // optimistic remove
try { await api.delete(id) } catch { setItems(old); showError(); }
```

---

### 7. Progressive / Incremental Loading

* Load essential data first, lazy-load the rest.
* Example: show user profile immediately, load activity feed after.

---

### 8. Retry & Backoff Strategies

* Provide manual retry (button).
* Implement automatic retry with exponential backoff for transient errors.
* Limit retries and inform user after repeated failures.

Example backoff:

```js
const backoff = (attempt) => Math.min(1000 * 2 ** attempt, 30000);
```

---

### 9. Cancellation and Stale Data Handling

* Cancel earlier requests on dependent changes (AbortController).
* Ignore stale responses (compare request id or use refs).
* Prevent flicker: only show loading UI when request takes longer than X ms (e.g., 200ms).

Debounce loading indicator:

```jsx
useEffect(() => {
  const t = setTimeout(() => setShowSpinner(true), 200);
  fetchData().finally(() => { clearTimeout(t); setShowSpinner(false); });
}, [key]);
```

---

### 10. Error Boundaries (UI-level crashes)

* Use React Error Boundaries to catch rendering errors and show fallback UIs.
* Error boundaries do not catch async errors (use try/catch and error state for those).

Example:

```jsx
<ErrorBoundary fallback={<AppErrorFallback />}>
  <SomeComponent />
</ErrorBoundary>
```

---

## Accessibility (A11y)

* Use `role="status"` or `aria-live="polite"` for loading and non-critical messages.
* For urgent errors, use `aria-live="assertive"`.
* Ensure focus management for error dialogs: focus the first actionable item.
* Provide keyboard-accessible retry controls.

Example:

```jsx
<div role="status" aria-live="polite">
  {loading ? "Loading…" : null}
</div>
```

---

## UI Patterns Summary Table

| Pattern              | Use When                          | UX Benefit                   |
| -------------------- | --------------------------------- | ---------------------------- |
| Skeleton screens     | Data-heavy UI                     | Better perceived performance |
| Local spinner        | Small component fetch             | Scoped, non-blocking         |
| Global spinner       | App-level loading                 | Clear app state              |
| Inline error + retry | Failures in small UI              | Clear recovery path          |
| Toast                | Background or non-critical errors | Non-obtrusive alerts         |
| Optimistic UI        | Low-risk writes                   | Instant feedback             |
| Progressive loading  | Large payloads                    | Faster first paint           |
| Retry/backoff        | Flaky networks                    | Automated resilience         |

---

## Code Examples

### Inline fetch with skeleton, debounce spinner, abort

```jsx
function UserProfile({ id }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [showSpinner, setShowSpinner] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    const signal = controller.signal;
    let spinnerTimer = setTimeout(() => setShowSpinner(true), 200);

    setLoading(true); setError(null);

    (async () => {
      try {
        const res = await fetch(`/api/users/${id}`, { signal });
        if (!res.ok) throw new Error("Failed to load");
        const json = await res.json();
        setData(json);
      } catch (err) {
        if (err.name !== "AbortError") setError(err);
      } finally {
        clearTimeout(spinnerTimer);
        setShowSpinner(false);
        setLoading(false);
      }
    })();

    return () => {
      clearTimeout(spinnerTimer);
      controller.abort();
    };
  }, [id]);

  if (error) return <InlineError message={error.message} onRetry={() => {/* trigger refetch */}} />;
  if (!data) return showSpinner ? <Spinner /> : <UserSkeleton />;
  return <UserCard user={data} />;
}
```

### Optimistic update example

```js
async function handleToggleLike(postId) {
  const old = posts;
  setPosts(p => p.map(x => x.id === postId ? { ...x, liked: !x.liked } : x)); // optimistic
  try {
    await api.post(`/posts/${postId}/like`);
  } catch (err) {
    setPosts(old); // rollback
    showError("Could not like post");
  }
}
```

---

## Testing Loading & Error States

* Unit test components for all states: loading, success, error.
* Mock fetch/Axios with delayed resolves/rejects to ensure skeletons, spinners, and retries behave properly.
* Test accessibility attributes (`aria-live`, focus management).

---

## When to Use Libraries

* For caching, background refresh, retries, deduping use **React Query** or **SWR** — they implement many patterns: optimistic updates, retries, stale-while-revalidate, cancellation, and built-in loading/error states.

---

## Interview-ready Answer

Loading and error UI patterns include skeleton screens or local spinners for component-level loads, global progress for app-level loads, inline errors with retry buttons, toast notifications for non-blocking errors, optimistic UI for fast writes, and cancellation/backoff for safe network handling. Use `AbortController` and stale-result guards to avoid race conditions, debounce brief loads to prevent flicker, and ensure accessibility with ARIA live regions. For complex data flows prefer libraries like React Query that handle caching, retries, and cancellation out of the box.
