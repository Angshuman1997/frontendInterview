# React Hooks Behavior Scenarios — `useContext`

## Overview

`useContext` is a React Hook that lets you **access data from a Context** directly within a functional component, without the need for prop drilling.
It works in conjunction with **React.createContext()** and a **Context.Provider**, enabling global or shared state between deeply nested components.

Unlike Redux or Zustand, it’s part of React’s core — ideal for **lightweight state sharing**, **theme toggling**, **authentication**, and **language settings**.

---

## 1️⃣ Basic Example — Accessing Context Value

```jsx
import { createContext, useContext } from "react";

const ThemeContext = createContext("light");

function Child() {
  const theme = useContext(ThemeContext);
  return <h2>Current Theme: {theme}</h2>;
}

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Child />
    </ThemeContext.Provider>
  );
}
```

✅ **Explanation**

* `createContext()` defines a context object.
* The `Provider` supplies the context value (`"dark"`).
* `useContext(ThemeContext)` lets `Child` directly read that value — no props needed.

✅ **Render Count:**
Re-renders occur **whenever the context value changes**.

---

## 2️⃣ Avoiding Prop Drilling with useContext

```jsx
function GrandParent() {
  return (
    <UserContext.Provider value={{ name: "Alice" }}>
      <Parent />
    </UserContext.Provider>
  );
}

function Parent() {
  return <Child />;
}

function Child() {
  const user = useContext(UserContext);
  return <p>Hello, {user.name}</p>;
}
```

✅ **Explanation**
Instead of passing `user` down through every component, `Child` accesses it directly via context.
Eliminates “prop drilling” — the tedious passing of props through intermediate layers.

---

## 3️⃣ Default Context Value

```jsx
const LanguageContext = createContext("English");

function App() {
  const lang = useContext(LanguageContext);
  return <p>Language: {lang}</p>;
}
```

✅ **Explanation**

* When no Provider is present, `useContext` returns the **default value** (`"English"`).
* This ensures safe fallback behavior.

---

## 4️⃣ Dynamic Context Values

```jsx
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={theme}>
      <Toolbar />
      <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
        Toggle Theme
      </button>
    </ThemeContext.Provider>
  );
}
```

✅ **Explanation**
When `setTheme` changes state, the Provider re-renders —
all consuming components (`useContext(ThemeContext)`) re-render with the new value.

---

## 5️⃣ useContext + useReducer (Global State Pattern)

```jsx
const CounterContext = createContext();

function counterReducer(state, action) {
  switch (action.type) {
    case "increment":
      return state + 1;
    default:
      return state;
  }
}

function CounterProvider({ children }) {
  const [count, dispatch] = useReducer(counterReducer, 0);
  return (
    <CounterContext.Provider value={{ count, dispatch }}>
      {children}
    </CounterContext.Provider>
  );
}

function CounterDisplay() {
  const { count, dispatch } = useContext(CounterContext);
  return (
    <>
      <p>Count: {count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
    </>
  );
}
```

✅ **Explanation**
Combines `useReducer` + `useContext` to create a **Redux-like global store**.
Commonly used to manage app-wide states without external libraries.

---

## 6️⃣ Multiple Contexts in One Component

```jsx
function App() {
  const theme = useContext(ThemeContext);
  const user = useContext(UserContext);

  return (
    <p style={{ color: theme === "dark" ? "white" : "black" }}>
      Welcome, {user.name}
    </p>
  );
}
```

✅ **Explanation**
A component can consume multiple contexts simultaneously.
However, too many contexts can lead to excessive re-renders — structure wisely.

---

## 7️⃣ Nested Context Providers

```jsx
<ThemeContext.Provider value="dark">
  <UserContext.Provider value={{ name: "Bob" }}>
    <Dashboard />
  </UserContext.Provider>
</ThemeContext.Provider>
```

✅ **Explanation**
Nested contexts allow scoping — inner providers override outer ones.
Each `useContext` call resolves to the nearest matching Provider above it.

---

## 8️⃣ Performance Concern — Context Re-Renders All Consumers

```jsx
const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext.Provider value={theme}>
      <Header />
      <Footer />
    </ThemeContext.Provider>
  );
}
```

✅ **Issue:**
When `theme` changes, **all components** using `useContext(ThemeContext)` will re-render.

✅ **Optimization:**
Split context or use memoized values to reduce re-renders.

```jsx
<ThemeContext.Provider value={useMemo(() => theme, [theme])}>
  ...
</ThemeContext.Provider>
```

---

## 9️⃣ Context with Objects (Stable Reference)

❌ **Bad (new object each render):**

```jsx
<ThemeContext.Provider value={{ mode: theme }}>
```

✅ **Good (memoized value):**

```jsx
const value = useMemo(() => ({ mode: theme }), [theme]);
<ThemeContext.Provider value={value}>
```

✅ **Explanation**
React compares context values **by reference**.
If the Provider’s value is a new object each render, all consumers re-render unnecessarily.

---

## 🔟 Context in Custom Hook (Best Practice)

```jsx
function useTheme() {
  return useContext(ThemeContext);
}

function Toolbar() {
  const theme = useTheme();
  return <p>Theme: {theme}</p>;
}
```

✅ **Explanation**
Encapsulates `useContext` inside a custom hook —
makes logic reusable and readable (like `useAuth()`, `useUser()`, etc.).

---

## 11️⃣ Consuming Context in Deeply Nested Components

```jsx
function Level3() {
  const user = useContext(UserContext);
  return <p>{user.name}</p>;
}
```

✅ **Explanation**
Even components 10 levels deep can directly consume the context — no manual prop passing.

---

## 12️⃣ Updating Context from Child Component

```jsx
const ThemeUpdateContext = createContext();

function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeUpdateContext.Provider value={setTheme}>
      <Toolbar />
    </ThemeUpdateContext.Provider>
  );
}

function Toolbar() {
  const toggleTheme = useContext(ThemeUpdateContext);
  return <button onClick={() => toggleTheme((t) => (t === "light" ? "dark" : "light"))}>Toggle</button>;
}
```

✅ **Explanation**
Passing setters or dispatchers through context allows **children to modify global state** safely.

---

## 13️⃣ Context + Memoization to Prevent Extra Renders

```jsx
const UserContext = createContext();

function UserProvider({ children }) {
  const [user, setUser] = useState({ name: "Alice", age: 25 });
  const value = useMemo(() => ({ user, setUser }), [user]);
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}
```

✅ **Explanation**

* `useMemo` ensures the context value only changes when `user` does.
* Prevents unnecessary renders of all consumer components.

---

## 14️⃣ Context in Combination with Suspense (Async Contexts)

```jsx
const DataContext = createContext();

function DataProvider({ children }) {
  const [data, setData] = useState(null);

  useLayoutEffect(() => {
    fetch("/api/data")
      .then((res) => res.json())
      .then(setData);
  }, []);

  return <DataContext.Provider value={data}>{children}</DataContext.Provider>;
}
```

✅ **Explanation**
Async context setup is common in data-heavy apps (e.g., fetching initial config or localization).

---

## 15️⃣ Common Mistakes

❌ **1. Using `useContext` without Provider**

```jsx
const user = useContext(UserContext); // ❌ user = undefined
```

✅ **Fix:** Wrap component with `<UserContext.Provider>`.

---

❌ **2. Changing object values inline**

```jsx
<Context.Provider value={{ theme }}> // ❌ new object each render
```

✅ Fix: Memoize with `useMemo`.

---

❌ **3. Overusing Context**

* Context is global — avoid using it for rapidly changing data (like form inputs).
  ✅ Use local `useState` for frequent updates.

---

## Summary Table

| #  | Scenario            | Behavior                          | Key Takeaway             |
| -- | ------------------- | --------------------------------- | ------------------------ |
| 1  | Basic usage         | Reads value from nearest Provider | No prop drilling         |
| 2  | Avoid drilling      | Direct data access                | Clean architecture       |
| 3  | Default value       | Fallback when no Provider         | Safe defaults            |
| 4  | Dynamic updates     | Re-renders consumers              | Real-time sync           |
| 5  | With useReducer     | Centralized global logic          | Redux-lite pattern       |
| 6  | Multiple contexts   | Combine states                    | Modular design           |
| 7  | Nested providers    | Scoped values                     | Layered state            |
| 8  | Performance issues  | All consumers re-render           | UseMemo for optimization |
| 9  | Object references   | New objects = re-renders          | Memoize values           |
| 10 | Custom hook         | Cleaner usage                     | Reusable logic           |
| 11 | Deep consumption    | Any depth                         | Simplifies data flow     |
| 12 | Updating from child | Setters in context                | Controlled updates       |
| 13 | Memoized context    | Stable reference                  | Efficient rendering      |
| 14 | Async context       | Fetch data on mount               | Common for configs       |
| 15 | Common mistakes     | Missing Provider, inline objects  | Optimize context use     |

---

## Interview-Ready Answer

`useContext` allows React components to consume values from a **Context Provider** without prop drilling.
It’s ideal for **sharing global data** like themes, user info, or language preferences.
When the context value changes, all consuming components automatically re-render.

For performance:

* **Memoize the context value** with `useMemo`.
* Avoid passing new object/array references inline.
* Combine with `useReducer` for structured global state.

It’s lightweight and built into React — perfect for small to medium shared state, but **for large apps, Redux or Zustand** may provide better scalability and dev tooling.

---

✅ **Key Takeaway:**

> `useContext` = No prop drilling, shared state.
> Use `useReducer` or `useMemo` with it for scalable and optimized performance.
