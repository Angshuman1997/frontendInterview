# React Hooks Behavior Scenarios — `useSelector`

## Overview

`useSelector` is a **React-Redux hook** that allows you to **read data from the Redux store** inside a React component.
It replaces the older `connect(mapStateToProps)` pattern and provides a more direct, hooks-based way to access global state.

Each time the selected state changes, the component **automatically re-renders**, ensuring your UI stays in sync with the store.

---

## 1️⃣ Basic Usage

```jsx
import { useSelector } from "react-redux";

function Profile() {
  const user = useSelector((state) => state.user);

  return <h2>Welcome, {user.name}</h2>;
}
```

✅ **Explanation**

* `useSelector` takes a function (selector) that receives the entire Redux `state`.
* It returns the value of the part of the state you want to use.
* The component re-renders **only** when the selected data changes (shallow comparison by default).

✅ **Render Count:**

* Initial render
* +1 for each time `state.user` changes.

---

## 2️⃣ Selecting a Specific Slice

```jsx
const products = useSelector((state) => state.shop.products);
```

✅ **Explanation**

* You can access nested slices of state directly.
* If `state.shop.products` changes, the component re-renders.
* If other parts of the state change, the component remains unaffected.

---

## 3️⃣ Selecting Multiple Values

```jsx
const { name, age } = useSelector((state) => state.user);
```

✅ **Explanation**

* You can destructure multiple values at once.
* React compares the entire returned object — if **any** property changes, component re-renders.

✅ **Optimization Tip:**
Use separate selectors for each value to minimize re-renders:

```jsx
const name = useSelector((state) => state.user.name);
const age = useSelector((state) => state.user.age);
```

---

## 4️⃣ Avoiding Unnecessary Re-renders (Custom Equality Function)

```jsx
import { shallowEqual, useSelector } from "react-redux";

const user = useSelector((state) => state.user, shallowEqual);
```

✅ **Explanation**

* By default, `useSelector` uses **strict equality (`===`)**.
* Use `shallowEqual` for shallow object comparison (like React’s `PureComponent`).
* Prevents unnecessary re-renders when objects have same properties but different references.

---

## 5️⃣ Selecting Derived Data (Compute Inside Selector)

```jsx
const totalPrice = useSelector(
  (state) => state.cart.items.reduce((sum, item) => sum + item.price, 0)
);
```

✅ **Explanation**

* You can perform calculations inside the selector function.
* However, avoid heavy computations here — they run on every render.

✅ **Optimization:**
Use **Reselect** to memoize derived data (see next).

---

## 6️⃣ Using Reselect for Memoized Selectors

```jsx
import { createSelector } from "@reduxjs/toolkit";

const selectCartItems = (state) => state.cart.items;
const selectTotal = createSelector(
  [selectCartItems],
  (items) => items.reduce((sum, i) => sum + i.price, 0)
);

const total = useSelector(selectTotal);
```

✅ **Explanation**

* `createSelector` memoizes the computation.
* It only recomputes when dependencies (`cart.items`) change.
* Prevents expensive recalculations and re-renders.

---

## 7️⃣ useSelector with TypeScript (Typed State)

```tsx
import { RootState } from "../store";

const count = useSelector((state: RootState) => state.counter.value);
```

✅ **Explanation**

* Strongly types `state` for safety and autocomplete.
* Prevents runtime errors and mismatched state keys.

---

## 8️⃣ useSelector and useDispatch Together

```jsx
import { useSelector, useDispatch } from "react-redux";
import { increment } from "../store/counterSlice";

function Counter() {
  const count = useSelector((state) => state.counter.value);
  const dispatch = useDispatch();

  return (
    <>
      <p>Count: {count}</p>
      <button onClick={() => dispatch(increment())}>+</button>
    </>
  );
}
```

✅ **Explanation**

* `useSelector` reads the current state.
* `useDispatch` updates the state.
* Component re-renders whenever the selected state changes.

---

## 9️⃣ Avoiding Infinite Re-renders

❌ **Bad**

```jsx
const user = useSelector((state) => ({ name: state.user.name }));
```

✅ **Explanation**

* Returns a **new object** every time → React sees it as “changed” → re-renders infinitely.

✅ **Fix**

```jsx
const name = useSelector((state) => state.user.name);
```

Always return **primitive values** or memoized objects.

---

## 🔟 Derived Selector with Filtering

```jsx
const activeTodos = useSelector(
  (state) => state.todos.filter((todo) => !todo.completed)
);
```

✅ **Explanation**

* Simple filtering logic in selector.
* Recomputed and re-renders only when `state.todos` changes.

✅ **Better:** Memoize with Reselect for large lists.

---

## 11️⃣ Handling Multiple Slices in One Component

```jsx
const user = useSelector((state) => state.user);
const settings = useSelector((state) => state.settings);
```

✅ **Explanation**

* Each selector independently subscribes to store updates.
* Component re-renders when **either** slice changes.

✅ **Best Practice:**
Split large components into smaller ones if slices are unrelated — improves performance.

---

## 12️⃣ Selector Performance Optimization (Reselect)

```jsx
import { createSelector } from "@reduxjs/toolkit";

const selectVisibleTodos = createSelector(
  [(state) => state.todos, (state) => state.filter],
  (todos, filter) => todos.filter((t) => t.status === filter)
);

const todos = useSelector(selectVisibleTodos);
```

✅ **Explanation**

* `createSelector` caches results.
* Avoids recomputation unless `todos` or `filter` change.

---

## 13️⃣ Using useSelector Inside Custom Hooks

```jsx
function useAuth() {
  const user = useSelector((state) => state.auth.user);
  const isAuthenticated = Boolean(user);
  return { user, isAuthenticated };
}
```

✅ **Explanation**

* Encapsulates Redux logic inside reusable hooks.
* Keeps components clean and logic centralized.

---

## 14️⃣ Using useSelector in Large Apps (Splitting Slices)

```jsx
const theme = useSelector((state) => state.theme.mode);
const notifications = useSelector((state) => state.notifications.list);
```

✅ **Explanation**

* React-Redux allows multiple concurrent selectors.
* Only the components depending on each slice will re-render when that slice updates — not the entire app.

---

## 15️⃣ Common Mistakes

❌ **1. Using useSelector outside <Provider>**

```jsx
const user = useSelector((state) => state.user); // ❌ Throws error
```

✅ Fix: Wrap app in `<Provider store={store}>`.

---

❌ **2. Returning new object/array in selector**

```jsx
const todos = useSelector((s) => [...s.todos]); // ❌ Always new array → re-render
```

✅ Fix: Return raw state or memoized result.

---

❌ **3. Over-selecting**

```jsx
const state = useSelector((state) => state); // ❌ Re-renders on ANY change
```

✅ Fix: Select only what you need (specific slices).

---

## Summary Table

| #  | Scenario           | Behavior                  | Key Takeaway           |
| -- | ------------------ | ------------------------- | ---------------------- |
| 1  | Basic usage        | Reads store data          | Direct state access    |
| 2  | Nested slice       | Reads deep state          | Efficient re-renders   |
| 3  | Multiple values    | Whole object comparison   | Use separate selectors |
| 4  | Custom equality    | Prevents re-renders       | Use shallowEqual       |
| 5  | Derived data       | Inline compute            | Optimize heavy logic   |
| 6  | Reselect           | Memoized selectors        | Prevent recomputation  |
| 7  | TypeScript         | Typed state               | Safer development      |
| 8  | With dispatch      | Read + update state       | Common pattern         |
| 9  | Infinite re-render | Avoid new objects         | Return primitives      |
| 10 | Filtered lists     | Derived filtering         | Memoize large data     |
| 11 | Multiple slices    | Independent reactivity    | Split components       |
| 12 | Performance        | Memoization               | Scalable pattern       |
| 13 | Custom hooks       | Encapsulate logic         | Reusable abstraction   |
| 14 | Large apps         | Slice-based selection     | Efficient re-renders   |
| 15 | Common mistakes    | Context or over-selection | Use best practices     |

---

## Interview-Ready Answer

`useSelector` is a React-Redux hook that allows React components to access specific parts of the global Redux store.
It re-renders the component only when the selected state changes, making it efficient and predictable.
To optimize performance, you can use **shallow equality**, **memoized selectors (Reselect)**, or **split selectors** by slice.
Unlike `useDispatch`, which triggers updates, `useSelector` is purely for **reading state reactively**.

Common pitfalls include selecting entire state objects, returning new references, or using it outside the `<Provider>` context.
