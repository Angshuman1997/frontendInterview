# Avoid unnecessary renders

This file is a compact checklist + patterns you can copy into your repo.
Keep it handy for interviews and as a practical guide when optimizing React apps.

---

## Why it matters

Every render costs CPU time and can trigger DOM updates. Avoiding needless renders:

* reduces CPU/GPU work
* improves frame rate and responsiveness
* lowers power/battery usage (mobile)
* decreases JS-main thread contention (better for animations)

---

## High-level strategies

1. **Prevent re-renders** when props/state haven’t meaningfully changed.
2. **Stabilize references** passed to children (functions, objects, arrays).
3. **Split components** so changes affect the smallest subtree possible.
4. **Memoize expensive computations** and derived data.
5. **Profile** and target hotspots — don’t prematurely optimize.

---

## Techniques & patterns

### 1. `React.memo` for functional components

Wrap pure presentational components to skip renders when props are shallowly equal.

```jsx
const Item = React.memo(function Item({ name }) {
  console.log("Item render", name);
  return <div>{name}</div>;
});
```

If you need deep comparison:

```jsx
const Item = React.memo(Component, (prev, next) => {
  return prev.id === next.id && prev.value === next.value;
});
```

---

### 2. Stabilize callbacks with `useCallback`

Pass stable function references to memoized children.

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // BAD: new function every render -> breaks React.memo(child)
  // const handleClick = () => setCount(c => c + 1);

  const handleClick = useCallback(() => setCount(c => c + 1), []);
  return <Child onClick={handleClick} />;
}
```

**When to use:** only when the callback is passed to memoized child or used in deps of other hooks.

---

### 3. Memoize computed values with `useMemo`

Avoid recomputing heavy derived data each render and avoid creating new object/array references.

```jsx
const filtered = useMemo(() => users.filter(u => u.active), [users]);

// For object props:
const config = useMemo(() => ({ pageSize: 20, sort: "asc" }), [/* deps */]);
<Child config={config} />
```

**Caveat:** `useMemo` has overhead — use it for expensive work or to stabilize references.

---

### 4. Prefer functional `setState` when updating from previous state

Avoid stale closures and unnecessary re-renders due to wrong dependency inclusion.

```jsx
// Correct: uses latest state, safe inside callbacks and batching
setCount(prev => prev + 1);
```

---

### 5. Avoid passing new objects/arrays inline as props

Inline literals create new references on every render.

```jsx
// BAD
<Child options={{ page: 1 }} />

// GOOD
const options = useMemo(() => ({ page: 1 }), [/* deps */]);
<Child options={options} />
```

---

### 6. Split components / lift minimal state

Keep state local to where it’s needed. If only a small subtree depends on state, move state down so sibling components don’t re-render.

```jsx
// instead of one big component with many states, break into <Header />, <List />, <Footer />
```

---

### 7. Use `useRef` for mutable values that don’t affect UI

Refs don’t trigger re-renders.

```jsx
const countRef = useRef(0);
countRef.current += 1; // won't re-render
```

Use refs for timers, previous values, and caches.

---

### 8. Optimize lists: keys, virtualization, and item memoization

* Use stable keys (`id`), never index if list can reorder.
* Use virtualization (react-window / react-virtualized) for large lists.
* Memoize list items or render-only visible items.

```jsx
// virtualization example (react-window)
import { FixedSizeList as List } from "react-window";
<List height={400} itemCount={items.length} itemSize={35}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</List>
```

---

### 9. `useSelector` optimization (React-Redux)

* Select the smallest slice possible.
* Use `shallowEqual` or memoized selectors (Reselect) for derived data.

```js
const name = useSelector(state => state.user.name);
const total = useSelector(selectTotalPrice); // selectTotalPrice is memoized
```

---

### 10. Class components: `PureComponent` / `shouldComponentUpdate`

If using classes:

```jsx
class MyComponent extends React.PureComponent { /* shallow prop/state compare */ }

// or custom:
shouldComponentUpdate(nextProps, nextState) {
  return nextProps.value !== this.props.value;
}
```

---

### 11. Avoid heavy computation in render — move to `useMemo` or worker

If computation is CPU intensive, offload to:

* `useMemo` to do it rarely, or
* Web Worker for asynchronous heavy work.

---

### 12. Leverage React 18 features: automatic batching, `useTransition`

* **Automatic batching** reduces renders by grouping state updates.
* `useTransition` marks non-urgent updates to avoid blocking UI; use for large list filtering/slow renders.

```jsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setFilter(newFilter); // low-priority update
});
```

---

### 13. Debounce/throttle frequent updates

On frequent input events (typing, scroll), debounce or throttle updates to state.

```jsx
// useDebounce custom hook usage
const debouncedQuery = useDebounce(query, 300);
useEffect(() => fetchResults(debouncedQuery), [debouncedQuery]);
```

---

### 14. Prevent re-renders from context value changes

Context consumers re-render when provider value reference changes. Memoize provider value or split contexts.

```jsx
const contextValue = useMemo(() => ({ user, setUser }), [user]);
<UserContext.Provider value={contextValue} />
```

---

### 15. Use the Profiler & DevTools to find real bottlenecks

Target the actual hot paths — use React DevTools Profiler and browser performance tab. Optimize the functions/components that show high render counts or long paint times.

---

## Quick anti-patterns (avoid these)

* Creating functions/objects inline every render and passing to memoized children.
* Using indexes as keys for reorderable lists.
* Returning new arrays/objects from selectors without memoization.
* Fetching and setting global state in many sibling components causing redundant re-fetches.
* Heavy work inside render or synchronous effects that block paint.

---

## Checklist before optimizing

* [ ] Measure with Profiler — is this actually a problem?
* [ ] Can UI be split so fewer components re-render?
* [ ] Are function/object references stabilized with `useCallback`/`useMemo`?
* [ ] Are computed values memoized where expensive?
* [ ] Are lists virtualized when large?
* [ ] Are context/provider values memoized?
* [ ] Are selectors memoized (Reselect) for derived data?
* [ ] Did you confirm the fix reduces renders & improves frame times?

---

## Example: combine patterns (practical)

```jsx
// Parent.jsx
function Parent({ items }) {
  const [filter, setFilter] = useState("");
  const filtered = useMemo(() => items.filter(i => i.includes(filter)), [items, filter]);

  const onSelect = useCallback(id => {
    // stable callback for memoized child
    console.log(id);
  }, []);

  return (
    <>
      <Search value={filter} onChange={setFilter} />
      <ItemList items={filtered} onSelect={onSelect} />
    </>
  );
}

// ItemList.jsx
const ItemList = React.memo(function ItemList({ items, onSelect }) {
  return items.map(item => <Item key={item.id} item={item} onSelect={onSelect} />);
});
```

---

## Interview-ready answer

To avoid unnecessary renders: split components, memoize pure components with `React.memo`, stabilize function/object references with `useCallback`/`useMemo`, use functional state updates, memoize derived selectors (Reselect), virtualize large lists, and profile with React DevTools. Only optimize measured bottlenecks and prefer simple structural fixes (splitting, memoizing context values) before adding complexity.

