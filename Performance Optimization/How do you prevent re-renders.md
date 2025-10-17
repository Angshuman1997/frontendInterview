# How do you prevent re-renders

A compact, copy-paste ready guide with patterns, code snippets and checklist you can drop into your repo.

---

## Short answer

Prevent re-renders by reducing when React thinks something changed: keep props/state minimal and stable, avoid creating new object/function references each render, split components so only the minimum subtree updates, memoize computed values and components, and use appropriate hooks (`useCallback`, `useMemo`, `useRef`, `React.memo`, selector memoization). Always measure first.

---

## Strategies & examples

### 1. Split components (localize state)

Keep state as close to the consumer as possible so fewer components re-render.

```jsx
// Instead of a big component with many pieces of state:
function App() {
  const [user, setUser] = useState(...); // used by Profile
  const [list, setList] = useState(...); // used by List
  // Profile and List both re-render on any state change

  return <>
    <Profile user={user} />
    <List items={list} />
  </>;
}
```

Split into smaller components so updating `list` doesn't re-render `Profile`.

---

### 2. React.memo for pure presentational children

Wrap child components to skip renders when props are shallowly equal.

```jsx
const Item = React.memo(function Item({ item, onClick }) {
  console.log("Item render");
  return <div onClick={() => onClick(item.id)}>{item.name}</div>;
});
```

---

### 3. Stabilize callbacks with `useCallback`

Pass stable function references to `React.memo` children.

```jsx
// BAD: new function every render -> child re-renders
<Child onAction={() => doSomething(id)} />

// GOOD
const onAction = useCallback(() => doSomething(id), [id]);
<Child onAction={onAction} />
```

Use `useCallback` only when function identity matters (child is memoized or it's a dependency).

---

### 4. Stabilize objects/arrays with `useMemo`

Avoid creating new object/array references which cause props inequality.

```jsx
// BAD
<Child config={{ page: 1, sort: 'asc' }} />

// GOOD
const config = useMemo(() => ({ page: 1, sort: 'asc' }), []);
<Child config={config} />
```

Also use `useMemo` for expensive derived values:

```jsx
const filtered = useMemo(() => items.filter(x => x.active), [items]);
```

---

### 5. Use functional state updates to avoid stale closures

When new state depends on previous value, use functional update form.

```jsx
// Avoid stale reads
setCount(prev => prev + 1);
```

This also often reduces dependency array needs (and re-creations).

---

### 6. Avoid inline creation of values used by children/effects

Do not return new objects/arrays from render for props or selector results.

```jsx
// BAD: creates new array every render -> child re-renders
const data = items.map(...);
<Child data={data} />

// Consider memoizing:
const data = useMemo(() => items.map(...), [items]);
```

---

### 7. Memoize selectors (Redux / useSelector)

Select only minimum slice; memoize derived values with Reselect.

```js
// selector.js
const selectItems = state => state.items;
const selectActiveCount = createSelector([selectItems], items =>
  items.filter(i => i.active).length
);

// component
const activeCount = useSelector(selectActiveCount);
```

---

### 8. Use `useRef` for mutable non-UI values

Refs update without re-render. Use them for timers, previous values, caches.

```jsx
const timerRef = useRef(null);
timerRef.current = setInterval(...); // no re-render
```

---

### 9. Virtualize large lists

Render only visible rows (react-window / react-virtualized).

```jsx
import { FixedSizeList as List } from 'react-window';
<List height={400} itemCount={items.length} itemSize={35}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</List>
```

---

### 10. Debounce / throttle frequent updates

On fast events (typing, scroll), reduce update frequency before setting state.

```jsx
const debounced = useDebounce(query, 300);
useEffect(() => fetchResults(debounced), [debounced]);
```

---

### 11. Avoid context value re-creation

Memoize context provider value to prevent all consumers re-rendering.

```jsx
const value = useMemo(() => ({ user, setUser }), [user]);
<UserContext.Provider value={value}>{children}</UserContext.Provider>
```

Or split context so unrelated pieces don’t share the same provider.

---

### 12. Use `useTransition` for non-urgent updates (React 18+)

Mark expensive UI updates as low priority so urgent UI (input) remains responsive.

```jsx
const [isPending, startTransition] = useTransition();
startTransition(() => setExpensiveFilter(filter));
```

---

### 13. Prefer simple primitives in props

Prefer passing primitives over complex objects when possible. If object needed, memoize it.

---

### 14. Avoid unnecessary effects that update state

Effects that run and set state each render can cause loops. Add proper dependencies and guards.

```jsx
useEffect(() => {
  if (!shouldFetch) return;
  // fetch and set state
}, [shouldFetch]); // include all used variables
```

---

### 15. Profile & measure first

Use React DevTools Profiler and browser performance tools to find real hot paths. Optimize where it counts.

---

## Checklist before optimizing

* [ ] Measured excessive renders with Profiler?
* [ ] Can state be localized (split components)?
* [ ] Are callbacks passed to children memoized (`useCallback`)?
* [ ] Are objects/arrays memoized (`useMemo`)?
* [ ] Are context provider values stable (`useMemo`) or split into multiple contexts?
* [ ] Are derived selectors memoized (Reselect)?
* [ ] Are large lists virtualized?
* [ ] Is `useRef` used for non-visual mutable data?
* [ ] Are debouncing/throttling applied for frequent input events?
* [ ] Did re-run tests to confirm render reduction and no behavior regressions?

---

## Common anti-patterns (what to avoid)

* Passing inline functions/objects to memoized children without memoizing.
* Using index as key on reorderable lists.
* Selecting whole Redux state (`useSelector(state => state)`) instead of precise slice.
* Premature optimization before profiling.
* Heavy computation directly in render (do `useMemo` or web worker).

---

## Interview-ready summary

To prevent re-renders: keep component boundaries tight, memoize components (`React.memo`), stabilize function/object references (`useCallback`, `useMemo`), use `useRef` for non-UI mutable values, memoize selectors, virtualize large lists, debounce frequent inputs, and always measure with the React Profiler before optimizing. Focus on structural fixes (splitting, memoizing provider values) before adding micro-optimizations.

