# React Hooks Behavior Scenarios — `useDispatch`

## Overview

`useDispatch` is a **React-Redux hook** used to **send actions to the Redux store**.
It replaces the older `connect()` HOC’s `mapDispatchToProps` pattern and allows dispatching actions directly from a React function component.

It’s part of the **React-Redux Hooks API**, alongside:

* `useSelector()` – to read data from the store
* `useDispatch()` – to update data via actions

---

## 1️⃣ Basic Usage

```jsx
import { useDispatch } from "react-redux";
import { increment } from "../features/counterSlice";

function CounterButton() {
  const dispatch = useDispatch();

  return (
    <button onClick={() => dispatch(increment())}>
      Increment
    </button>
  );
}
```

✅ **Explanation**

* `useDispatch()` gives access to the store’s `dispatch` function.
* You can dispatch **action creators**, **plain objects**, or **thunks** (async actions).
* Dispatching triggers reducers, updating the Redux state.

✅ **Render Count:**
Dispatch itself doesn’t cause re-renders — **only components using `useSelector`** will re-render when subscribed data changes.

---

## 2️⃣ Dispatching a Plain Action

```jsx
dispatch({ type: "cart/addItem", payload: { id: 1, name: "Laptop" } });
```

✅ **Explanation**

* The dispatched action flows through all reducers.
* The matching reducer handles the update and returns a new state.
* Components subscribed via `useSelector` re-render automatically.

---

## 3️⃣ Dispatching an Action Creator (Redux Toolkit)

```jsx
import { addItem } from "../store/cartSlice";

dispatch(addItem({ id: 2, name: "Phone" }));
```

✅ **Explanation**

* Redux Toolkit auto-generates action creators.
* Cleaner syntax compared to manual `{ type, payload }` objects.

---

## 4️⃣ Dispatching an Async Thunk (Side Effects)

```jsx
import { fetchUsers } from "../features/userSlice";

useEffect(() => {
  dispatch(fetchUsers());
}, [dispatch]);
```

✅ **Explanation**

* Async thunks return a function — Redux Toolkit middleware (`redux-thunk`) handles async logic.
* `fetchUsers` dispatches `pending`, `fulfilled`, and `rejected` actions automatically.

✅ **Common Pattern**
Used for API calls, data fetching, and server updates inside React components.

---

## 5️⃣ Dispatch Is Stable

```jsx
const dispatch = useDispatch();

useEffect(() => {
  console.log("Dispatch changed");
}, [dispatch]);
```

✅ **Explanation**
`dispatch` is **guaranteed to be stable** — it never changes identity between renders.
You can safely include it in `useEffect` dependency arrays without causing re-runs.

---

## 6️⃣ useDispatch with useCallback

```jsx
const dispatch = useDispatch();

const addTodo = useCallback(() => {
  dispatch({ type: "todos/add", payload: "Learn Hooks" });
}, [dispatch]);
```

✅ **Explanation**

* Though `dispatch` itself is stable, wrapping it with `useCallback` is good practice when passing functions down to memoized child components.
* Prevents child re-renders due to changing function references.

---

## 7️⃣ Dispatching Multiple Actions Sequentially

```jsx
dispatch(startLoading());
dispatch(fetchData());
dispatch(stopLoading());
```

✅ **Explanation**

* Each `dispatch` triggers a reducer call.
* React-Redux automatically batches multiple state updates (React 18 feature).
* Results in **one re-render** of subscribed components instead of three.

---

## 8️⃣ Conditional Dispatch (Avoiding Redundant Updates)

```jsx
const user = useSelector((state) => state.user);

if (!user.loaded) {
  dispatch(fetchUser());
}
```

✅ **Explanation**
Only dispatch when needed to avoid redundant re-fetching and unnecessary state transitions.

---

## 9️⃣ Using useDispatch in Async Event Handlers

```jsx
const dispatch = useDispatch();

const handleSubmit = async (formData) => {
  await dispatch(saveForm(formData));
  dispatch(showToast("Form saved successfully!"));
};
```

✅ **Explanation**
You can combine async thunks and normal actions inside async handlers.
Redux Toolkit’s `dispatch` returns the promise of the thunk result, making this pattern elegant.

---

## 🔟 Dispatching from Custom Hooks

```jsx
function useAuthActions() {
  const dispatch = useDispatch();
  return {
    login: (user) => dispatch(login(user)),
    logout: () => dispatch(logout()),
  };
}
```

✅ **Explanation**
Encapsulate all Redux action logic in a custom hook.
Keeps UI components clean and focused only on rendering.

---

## 11️⃣ Dispatching with Payload Validation

```jsx
const handleAdd = (product) => {
  if (!product.id) return console.warn("Invalid product");
  dispatch(addProduct(product));
};
```

✅ **Explanation**
Always validate payloads before dispatching.
Reducers should be pure and not contain runtime validation logic.

---

## 12️⃣ Dispatch in useEffect (Fetch on Mount)

```jsx
useEffect(() => {
  dispatch(fetchTodos());
}, [dispatch]);
```

✅ **Explanation**

* Common pattern for data loading.
* Since `dispatch` is stable, the effect runs only once (on mount).

---

## 13️⃣ Dispatch with Middleware (Redux Thunk or Saga)

```jsx
dispatch(fetchUserDataAsync());
```

✅ **Explanation**
Middleware like **redux-thunk** or **redux-saga** intercepts the dispatched action and runs async logic before updating state.

✅ Useful for:

* API calls
* Logging
* Analytics
* Delayed dispatch

---

## 14️⃣ Dispatch with Context/Reducer (Hybrid Pattern)

If you use `useReducer` + Context (without Redux):

```jsx
const dispatch = useContext(AppDispatchContext);
dispatch({ type: "add_todo", payload: "Buy milk" });
```

✅ **Explanation**
Even in non-Redux setups, the **dispatch pattern** is universal.
You can use custom reducers to manage global or local state in React apps.

---

## 15️⃣ Common Mistakes

❌ **1. Dispatch inside render**

```jsx
dispatch(doSomething()); // ❌ Causes infinite re-renders
```

✅ **Fix:** Dispatch inside event handlers or effects.

---

❌ **2. Mutating state inside reducers**
Reducers must be **pure** and **immutable**.

✅ Use Redux Toolkit’s `createSlice()` — it safely mutates under the hood using Immer.

---

❌ **3. Forgetting to wrap app in `<Provider>`**

```jsx
<Provider store={store}>
  <App />
</Provider>
```

`useDispatch()` and `useSelector()` will throw errors if no store is available in context.

---

## Summary Table

| #  | Scenario             | Behavior                 | Key Takeaway            |
| -- | -------------------- | ------------------------ | ----------------------- |
| 1  | Basic usage          | Dispatch actions         | Trigger reducer updates |
| 2  | Plain actions        | Send `{ type, payload }` | Manual approach         |
| 3  | Action creators      | Cleaner syntax           | Redux Toolkit friendly  |
| 4  | Async thunks         | Dispatch async logic     | Handles API calls       |
| 5  | Dispatch stable      | Doesn’t change           | Safe in deps            |
| 6  | useCallback          | For child props          | Prevents re-renders     |
| 7  | Multiple dispatches  | Batched (React 18)       | Efficient updates       |
| 8  | Conditional dispatch | Avoid redundancy         | Optimize logic          |
| 9  | Async handlers       | Combine sync + async     | Modern pattern          |
| 10 | Custom hooks         | Encapsulate logic        | Cleaner UI code         |
| 11 | Validation           | Avoid bad payloads       | Pure reducers           |
| 12 | useEffect mount      | Common for fetches       | Standard pattern        |
| 13 | Middleware           | Intercept actions        | Async orchestration     |
| 14 | With context         | Same concept             | Scalable state pattern  |
| 15 | Common mistakes      | Wrong usage patterns     | Avoid infinite loops    |

---

## Interview-Ready Answer

`useDispatch` is a React-Redux hook that returns the store’s `dispatch` function, allowing components to send actions that update global state.
You can dispatch **plain actions**, **Redux Toolkit action creators**, or **async thunks**.
It’s stable across renders, safe to use in effects, and doesn’t itself cause re-renders — only components using `useSelector` re-render when the store changes.
Combined with `useSelector`, `useDispatch` forms the backbone of modern Redux-based React applications.

