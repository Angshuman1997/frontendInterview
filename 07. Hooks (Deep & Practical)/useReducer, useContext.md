# useReducer, useContext

## Overview

`useReducer` and `useContext` are two powerful React Hooks that help manage **state** and **data sharing** across components in a predictable and scalable way.
They are often used together as a lightweight alternative to **Redux** for global state management.

---

## 1. useReducer — Managing Complex State Logic

### Purpose

`useReducer` is an alternative to `useState` for managing **complex or interrelated state logic**.
It provides a **centralized way** to handle state transitions using a **reducer function** — similar to Redux.

---

### Syntax

```javascript
const [state, dispatch] = useReducer(reducer, initialState);
```

* **reducer:** A pure function `(state, action) => newState`
* **state:** Current state value
* **dispatch:** Function used to send actions to the reducer
* **initialState:** The starting state value

---

### Example — Counter

```javascript
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
      throw new Error("Unknown action type");
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>−</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

---

### When to Use `useReducer`

* State depends on multiple variables or has complex update logic.
* Multiple state updates are triggered by specific actions or events.
* The same logic is needed in multiple components (when combined with `useContext`).

---

### Benefits

* Centralized and predictable state management.
* Easy to debug and extend with more actions.
* Makes component logic cleaner when state updates depend on multiple factors.

---

## 2. useContext — Sharing Data Across Components

### Purpose

`useContext` provides a way to share **data** across the component tree **without prop drilling** (passing props manually through every level).

It works with the **Context API**, which allows global-like state sharing.

---

### Syntax

```javascript
const value = useContext(MyContext);
```

---

### Example — Theme Context

```javascript
import { createContext, useContext, useState } from "react";

const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  const toggleTheme = () => setTheme(t => (t === "light" ? "dark" : "light"));

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function Toolbar() {
  const { theme, toggleTheme } = useContext(ThemeContext);

  return (
    <div style={{ background: theme === "light" ? "#fff" : "#333", color: theme === "light" ? "#000" : "#fff" }}>
      <p>Current theme: {theme}</p>
      <button onClick={toggleTheme}>Toggle Theme</button>
    </div>
  );
}

function App() {
  return (
    <ThemeProvider>
      <Toolbar />
    </ThemeProvider>
  );
}
```

---

### When to Use `useContext`

* To share global data like theme, authentication, language, or user info.
* To avoid deeply nested prop passing.
* To create global stores for multiple components.

---

## 3. useReducer + useContext — Global State Management Example

Combining `useReducer` with `useContext` allows you to build a **global store** (like Redux) for state management.

### Example — App-Wide Counter

```javascript
import { createContext, useReducer, useContext } from "react";

// 1. Create Context
const CounterContext = createContext();

// 2. Reducer function
const reducer = (state, action) => {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    default:
      return state;
  }
};

// 3. Provider component
export function CounterProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <CounterContext.Provider value={{ state, dispatch }}>
      {children}
    </CounterContext.Provider>
  );
}

// 4. Custom Hook for consuming context
export const useCounter = () => useContext(CounterContext);

// 5. Usage
function CounterDisplay() {
  const { state, dispatch } = useCounter();
  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>−</button>
    </div>
  );
}

function App() {
  return (
    <CounterProvider>
      <CounterDisplay />
    </CounterProvider>
  );
}
```

---

## 4. Key Differences

| Hook             | Purpose                                              | Common Use                                       |
| ---------------- | ---------------------------------------------------- | ------------------------------------------------ |
| **useReducer**   | Manages complex local state logic                    | Component-level or global state management       |
| **useContext**   | Shares data between components without prop drilling | Global/shared data like theme or user            |
| **Combined Use** | Global state + central reducer + shared context      | Lightweight state management (Redux alternative) |

---

## 5. Interview-Ready Answer

`useReducer` is used to manage complex state logic where multiple state transitions depend on specific actions, while `useContext` allows sharing data across components without prop drilling.
When combined, they create a scalable global state management pattern similar to Redux — `useReducer` handles state updates, and `useContext` provides global access to that state and dispatch function across the app.
