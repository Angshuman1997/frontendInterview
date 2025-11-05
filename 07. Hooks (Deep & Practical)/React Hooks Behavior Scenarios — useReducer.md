# React Hooks Behavior Scenarios — `useReducer`

## Overview

`useReducer` is a React Hook that provides a **more structured and predictable way** to manage complex state logic — especially when state transitions depend on **multiple actions or conditions**.

It’s similar to Redux but **local to a component**.
While `useState` is best for simple state, `useReducer` shines when you have **complex state updates**, **multiple related fields**, or **clear event-driven transitions**.

---

## 1️⃣ Basic Syntax

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

* **state** → current state value
* **dispatch(action)** → function to trigger state update
* **reducer(state, action)** → pure function that returns new state

---

## 2️⃣ Simple Counter Example

```jsx
import { useReducer } from "react";

const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return initialState;
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

✅ **Explanation**

* `dispatch` triggers the reducer function.
* Reducer returns a **new state object** each time.
* React re-renders the component with the updated state.

✅ **Render Count:**
Initial + 1 per dispatch action

---

## 3️⃣ When to Use `useReducer` Over `useState`

| Use `useState` When               | Use `useReducer` When                      |
| --------------------------------- | ------------------------------------------ |
| State is simple (single variable) | State logic is complex or interdependent   |
| Updates are isolated              | Multiple updates depend on one another     |
| Few event handlers                | Many event types / actions                 |
| Example: toggles, counters        | Example: form states, reducers, game logic |

✅ **Rule of thumb:**

> If you find yourself calling `setState` multiple times or deeply nesting objects → switch to `useReducer`.

---

## 4️⃣ Using `useReducer` with Complex State Objects

```jsx
const initialState = {
  name: "",
  age: 0,
  city: "",
};

function reducer(state, action) {
  switch (action.type) {
    case "update_name":
      return { ...state, name: action.payload };
    case "update_age":
      return { ...state, age: action.payload };
    case "update_city":
      return { ...state, city: action.payload };
    default:
      return state;
  }
}
```

✅ **Explanation**

* Cleaner and more predictable than multiple `useState`s.
* Avoids re-render storms from multiple state setters.

---

## 5️⃣ Functional Updates vs Reducers

**With `useState`:**

```jsx
setUser(prev => ({ ...prev, name: "John" }));
```

**With `useReducer`:**

```jsx
dispatch({ type: "update_name", payload: "John" });
```

✅ `useReducer` provides explicit **action-driven** updates — easier to debug and reason about in large components.

---

## 6️⃣ Lazy Initialization (Expensive Initial State)

```jsx
function init(initialCount) {
  console.log("Initializing...");
  return { count: initialCount };
}

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, 0, init);
}
```

✅ **Explanation**

* The third parameter (`init`) runs only once, during initial render.
* Prevents expensive setup computations from running on every render.

---

## 7️⃣ Preventing Unnecessary Re-renders

```jsx
const [state, dispatch] = useReducer(reducer, initialState);

const memoizedDispatch = useCallback(dispatch, []);
```

✅ `dispatch` itself is **stable** (doesn’t change identity across renders).
So memoization is **not needed**, unlike functions created with `useState`.

---

## 8️⃣ Dispatch Inside `useEffect`

```jsx
useEffect(() => {
  dispatch({ type: "increment" });
}, []);
```

✅ **Explanation**

* Works same as `setState` in `useEffect`.
* React batches state updates; causes only **one render**.

---

## 9️⃣ useReducer with Side Effects (Async Actions)

```jsx
function reducer(state, action) {
  switch (action.type) {
    case "fetch_start":
      return { ...state, loading: true, error: null };
    case "fetch_success":
      return { ...state, loading: false, data: action.payload };
    case "fetch_error":
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
}

function FetchData() {
  const [state, dispatch] = useReducer(reducer, {
    loading: false,
    data: null,
    error: null,
  });

  useEffect(() => {
    dispatch({ type: "fetch_start" });

    fetch("https://jsonplaceholder.typicode.com/users")
      .then(res => res.json())
      .then(data => dispatch({ type: "fetch_success", payload: data }))
      .catch(err => dispatch({ type: "fetch_error", payload: err.message }));
  }, []);
}
```

✅ **Explanation**

* Common pattern for API calls (like Redux async actions).
* Keeps loading, success, and error states in one predictable structure.

---

## 🔟 Combining useReducer with Context (Global Store Pattern)

```jsx
const initialState = { user: null };

function reducer(state, action) {
  switch (action.type) {
    case "login":
      return { ...state, user: action.payload };
    case "logout":
      return { ...state, user: null };
    default:
      return state;
  }
}

const AuthContext = createContext();

export function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <AuthContext.Provider value={{ state, dispatch }}>
      {children}
    </AuthContext.Provider>
  );
}
```

✅ **Explanation**

* `useReducer` + `useContext` mimics a **Redux-like global state**.
* Enables scalable, centralized state management.

---

## 11️⃣ Avoiding Re-renders with Stable Dispatch

```jsx
function Child({ onIncrement }) {
  console.log("Child render");
  return <button onClick={onIncrement}>+</button>;
}

function Parent() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return <Child onIncrement={() => dispatch({ type: "increment" })} />;
}
```

❌ **Issue:**
`Child` re-renders every time because inline arrow creates new function.

✅ **Fix:**

```jsx
const increment = useCallback(() => dispatch({ type: "increment" }), []);
<Child onIncrement={increment} />;
```

✅ Prevents unnecessary child re-renders.

---

## 12️⃣ Immutable State Principle

Always return a **new object** from the reducer — never mutate state.

❌ Wrong:

```jsx
state.count++;
return state;
```

✅ Correct:

```jsx
return { ...state, count: state.count + 1 };
```

**Why?**
React compares old and new state objects by reference.
If the same object is returned, React assumes nothing changed and skips re-render.

---

## 13️⃣ Combining Multiple Reducers (Composition)

```jsx
function userReducer(state, action) { ... }
function themeReducer(state, action) { ... }

function rootReducer(state, action) {
  return {
    user: userReducer(state.user, action),
    theme: themeReducer(state.theme, action),
  };
}
```

✅ **Explanation**

* You can compose multiple reducers for cleaner code.
* Pattern inspired by Redux’s `combineReducers`.

---

## 14️⃣ Avoid Infinite Loops in useReducer

```jsx
useEffect(() => {
  dispatch({ type: "increment" });
}, [state.count]);
```

❌ **Issue:**
Each state update changes `state.count`, retriggering effect → infinite loop.

✅ **Fix:**
Run once, or add condition inside the effect:

```jsx
useEffect(() => {
  if (state.count === 0) dispatch({ type: "increment" });
}, [state.count]);
```

---

## 15️⃣ useReducer vs Redux

| Feature       | `useReducer`                  | Redux                      |
| ------------- | ----------------------------- | -------------------------- |
| Scope         | Local (per component)         | Global                     |
| Boilerplate   | Minimal                       | High                       |
| Async support | Manual via `useEffect`        | Middleware (thunk/saga)    |
| DevTools      | No                            | Yes                        |
| Best for      | Component-level complex state | App-wide predictable state |

---

## Summary Table

| #  | Scenario              | Behavior                      | Key Takeaway                |
| -- | --------------------- | ----------------------------- | --------------------------- |
| 1  | Basic syntax          | Returns `[state, dispatch]`   | Foundation                  |
| 2  | Counter example       | Action-based updates          | Easy logic separation       |
| 3  | When to use           | Complex or structured updates | Clear choice over useState  |
| 4  | Complex state         | Clean and scalable            | Avoid multiple useState     |
| 5  | Functional vs reducer | Action-driven logic           | Predictable updates         |
| 6  | Lazy initialization   | Runs once                     | Performance optimized       |
| 7  | Stable dispatch       | No memo needed                | Dispatch is stable          |
| 8  | In useEffect          | Works like setState           | Batched updates             |
| 9  | Async actions         | Manages loading/error states  | Clean data fetching         |
| 10 | With context          | Global store                  | Local Redux pattern         |
| 11 | Child re-render       | Inline callback issue         | Use useCallback             |
| 12 | Immutable state       | Avoid mutations               | React compares by reference |
| 13 | Multiple reducers     | Modular                       | Clean structure             |
| 14 | Infinite loops        | Wrong dependency usage        | Guard logic                 |
| 15 | vs Redux              | Local vs global               | Choose based on scope       |

---

## Interview-Ready Answer

`useReducer` is a React hook used for managing **complex state logic**.
It replaces multiple `useState` calls with a **single, predictable reducer function** that handles all state transitions through `dispatch` actions.
It’s ideal when state changes depend on multiple variables or event types.
Unlike Redux, it’s **local to the component** and doesn’t require external libraries.
For global or shared state, `useReducer` is often combined with `useContext` to create a lightweight, Redux-like architecture.
