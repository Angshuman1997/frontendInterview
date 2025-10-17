# What's the difference between Context and Redux

## Overview

Both **Context API** and **Redux** are tools for **state management** in React, but they serve different purposes and work at different scales.

* **Context API** is a **built-in React feature** for passing data deeply through the component tree without prop drilling.
* **Redux** is a **state management library** designed to handle **global, predictable, and complex state** with strict structure and debugging tools.

---

## 1. Core Concept

| Feature        | **Context API**                                                                  | **Redux**                                                                        |
| -------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Definition** | React’s built-in mechanism to share data across components without prop drilling | A third-party state management library for predictable, centralized global state |
| **Purpose**    | Avoid prop drilling for simple shared data                                       | Manage large, complex, and frequently updated global state                       |
| **Type**       | Context-based dependency injection                                               | Centralized global store with reducers and actions                               |

---

## 2. How They Work

### Context API Flow

```
Parent (Provider)
   ↓
Child Components (Consumers using useContext)
```

* You create a **Context** using `createContext()`.
* Wrap components in a **Provider** and pass a value.
* Any child component can access that value using `useContext()`.

**Example**

```javascript
const ThemeContext = createContext();

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Toolbar />
    </ThemeContext.Provider>
  );
}

function Toolbar() {
  const theme = useContext(ThemeContext);
  return <p>Theme: {theme}</p>;
}
```

✅ Simple, lightweight solution for sharing static or rarely changing data (e.g., theme, auth user, language).

---

### Redux Flow

```
Component → Dispatch(Action) → Reducer → Store (New State) → Components re-render
```

* Redux uses a **central store** that holds the entire app’s state.
* Components **dispatch actions** to trigger **reducers** that update the store immutably.
* Any component can access data from the store using `useSelector()`.

**Example**

```javascript
import { createSlice, configureStore } from "@reduxjs/toolkit";
import { useDispatch, useSelector } from "react-redux";

// Slice
const themeSlice = createSlice({
  name: "theme",
  initialState: { mode: "light" },
  reducers: {
    toggleTheme: (state) => {
      state.mode = state.mode === "light" ? "dark" : "light";
    },
  },
});

const store = configureStore({ reducer: { theme: themeSlice.reducer } });
export const { toggleTheme } = themeSlice.actions;

// Component
function Toolbar() {
  const mode = useSelector((state) => state.theme.mode);
  const dispatch = useDispatch();

  return (
    <div>
      <p>Theme: {mode}</p>
      <button onClick={() => dispatch(toggleTheme())}>Toggle</button>
    </div>
  );
}
```

✅ Scalable, structured, and predictable state management for large applications.

---

## 3. Feature Comparison

| Feature                  | **Context API**                             | **Redux**                                               |
| ------------------------ | ------------------------------------------- | ------------------------------------------------------- |
| **Setup Complexity**     | Very simple, built into React               | Requires setup (store, reducers, middleware)            |
| **Use Case**             | Avoid prop drilling, simple global data     | Complex state management, async flows                   |
| **Data Flow**            | One-way (Provider → Consumer)               | Strict unidirectional (Dispatch → Reducer → Store → UI) |
| **Performance**          | Re-renders all consumers on value change    | Selective re-renders via `useSelector`                  |
| **Async Logic Handling** | Not built-in                                | Built-in tools (`createAsyncThunk`, `redux-saga`, etc.) |
| **DevTools Support**     | Limited                                     | Excellent (Redux DevTools for time travel, debugging)   |
| **State Persistence**    | Manual                                      | Built-in or via middleware (e.g., `redux-persist`)      |
| **Scalability**          | Good for small/medium apps                  | Excellent for large/enterprise apps                     |
| **Testing**              | Harder to test due to direct React coupling | Easier due to pure functions (reducers)                 |
| **Middleware Support**   | ❌ None                                      | ✅ Available (thunk, saga, logger, etc.)                 |

---

## 4. When to Use Context API

✅ Use **Context API** when:

* You need to **avoid prop drilling** for simple global data (like theme, locale, user session).
* The shared data doesn’t change frequently.
* You want a **lightweight, no-dependency** solution.

**Examples:**

* Theme toggling (`light` / `dark`)
* Authentication info (current user)
* UI settings (language, layout)

⚠️ Avoid using Context for:

* Large or deeply nested state trees.
* Frequently changing state (can trigger unnecessary re-renders).
* Async or complex logic.

---

## 5. When to Use Redux

✅ Use **Redux (with Redux Toolkit)** when:

* You have **complex global state logic** shared by many components.
* You need **predictability** and **debugging tools** (Redux DevTools).
* You handle **asynchronous workflows** (API calls, side effects).
* You’re building **enterprise-grade apps** with large teams.

**Examples:**

* E-commerce cart management
* Large dashboard apps
* Multi-user state synchronization
* Complex API data handling

---

## 6. Combining Context + Redux

You can use both together:

* Use **Context API** to provide the Redux store to your app (via `<Provider>`).
* Use **Redux** to manage actual state updates and async logic.

```javascript
import { Provider } from "react-redux";
import { store } from "./store";

function App() {
  return (
    <Provider store={store}>
      <MyAppComponents />
    </Provider>
  );
}
```

---

## 7. Summary Table

| Category       | **Context API**                  | **Redux**                    |
| -------------- | -------------------------------- | ---------------------------- |
| Type           | Built-in React feature           | Third-party library          |
| Complexity     | Simple                           | Structured & complex         |
| Best For       | Small to medium apps             | Medium to large apps         |
| State Storage  | Context Provider                 | Centralized Store            |
| Performance    | May cause unnecessary re-renders | Optimized with selectors     |
| Async Handling | Manual (useEffect)               | Built-in with thunks/sagas   |
| DevTools       | Minimal                          | Advanced debugging tools     |
| Learning Curve | Easy                             | Moderate                     |
| Boilerplate    | Minimal                          | Moderate (simplified by RTK) |

---

## 8. Interview-Ready Answer

The **Context API** is React’s built-in solution for sharing data across components without prop drilling, ideal for simple, low-frequency global state like themes or user sessions.
**Redux**, on the other hand, is a powerful state management library designed for large-scale applications that require a predictable, centralized store and structured state flow.

Redux supports middleware, DevTools, async operations, and debugging capabilities, making it better suited for complex, data-driven applications.
In short:

* Use **Context API** for simple global state.
* Use **Redux** for complex, dynamic, and large-scale state management.

