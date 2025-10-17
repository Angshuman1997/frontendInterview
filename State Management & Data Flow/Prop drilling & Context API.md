# Prop drilling & Context API

## Overview

In React, **prop drilling** refers to passing data through multiple layers of components that don’t need it — just so it can reach a deeply nested child.
The **Context API** solves this problem by providing a way to share data globally across the component tree **without manually passing props** at every level.

---

## 1. Prop Drilling

### Definition

**Prop drilling** happens when data is passed from a parent component to deeply nested child components via intermediate components that don’t use the data themselves.

### Example — Prop Drilling Problem

```javascript
function App() {
  const user = { name: "Alice" };
  return <Layout user={user} />;
}

function Layout({ user }) {
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {
  return <UserInfo user={user} />;
}

function UserInfo({ user }) {
  return <p>Hello, {user.name}</p>;
}
```

### Problem

* Each intermediate component (`Layout`, `Sidebar`) must receive and pass down the `user` prop even though they don’t need it.
* As the component tree grows, this becomes harder to maintain and refactor.

### Disadvantages of Prop Drilling

* Makes code verbose and repetitive.
* Harder to manage and maintain when components are deeply nested.
* Any prop name change requires updates across multiple files.
* Causes unnecessary re-renders in components that don’t use the data.

---

## 2. Context API — The Solution

### Definition

The **React Context API** provides a way to share values (like state, theme, or user data) across components **without passing props manually** at every level.
It acts as a **global data layer** for React components.

### Core Concepts

1. **`createContext()`** — Creates a Context object.
2. **`Provider`** — Supplies the data to components below it.
3. **`useContext()`** — Consumes data from the nearest Provider.

---

### Example — Using Context API

```javascript
import { createContext, useContext, useState } from "react";

// 1. Create a Context
const UserContext = createContext();

// 2. Create a Provider Component
function App() {
  const [user] = useState({ name: "Alice" });

  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

// 3. Consume Context Anywhere
function Layout() {
  return <Sidebar />;
}

function Sidebar() {
  return <UserInfo />;
}

function UserInfo() {
  const user = useContext(UserContext);
  return <p>Hello, {user.name}</p>;
}
```

### What’s Happening

* `UserContext.Provider` wraps the component tree and provides the `user` data.
* `useContext(UserContext)` in `UserInfo` directly accesses that data, **bypassing** all intermediate components.

✅ **No prop drilling.**
✅ **Cleaner, scalable, and easier-to-maintain code.**

---

## 3. When to Use Context API

### Use Context API When:

* Multiple components across different levels need access to the same data.
* You’re passing props through many layers unnecessarily.
* You have global data such as:

  * Authenticated user info
  * Theme (light/dark mode)
  * Language or locale
  * App-wide settings

### Avoid Context for:

* Frequently changing data (can cause unnecessary re-renders).
* Local state specific to one or two components.
* Performance-critical paths — in such cases, consider memoization or splitting contexts.

---

## 4. Example — Theme Context

```javascript
const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Toolbar />
    </ThemeContext.Provider>
  );
}

function Toolbar() {
  return (
    <div>
      <ThemeToggleButton />
      <ThemedContent />
    </div>
  );
}

function ThemeToggleButton() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      Toggle Theme
    </button>
  );
}

function ThemedContent() {
  const { theme } = useContext(ThemeContext);
  return <div style={{ background: theme === "light" ? "#fff" : "#333", color: theme === "light" ? "#000" : "#fff" }}>Themed Content</div>;
}
```

✅ The `theme` state and its updater are accessible anywhere in the tree — no prop drilling.

---

## 5. Context API Best Practices

| Best Practice                 | Description                                                                    |
| ----------------------------- | ------------------------------------------------------------------------------ |
| **Split Contexts**            | Use separate contexts for unrelated data (e.g., `UserContext`, `ThemeContext`) |
| **Keep Context Lightweight**  | Don’t put large or frequently updated objects in context                       |
| **Combine with Custom Hooks** | Wrap `useContext()` inside a custom hook for cleaner access                    |
| **Use Memoization**           | Use `useMemo()` when passing context value to prevent unnecessary re-renders   |

Example:

```javascript
const value = useMemo(() => ({ theme, setTheme }), [theme]);
<ThemeContext.Provider value={value}>...</ThemeContext.Provider>
```

---

## 6. Prop Drilling vs Context API — Comparison

| Feature            | Prop Drilling           | Context API                       |
| ------------------ | ----------------------- | --------------------------------- |
| Data Sharing       | Manual via props        | Global via Provider               |
| Scope              | Local (parent → child)  | Global (any component)            |
| Maintenance        | Difficult in deep trees | Easy and scalable                 |
| Re-render Overhead | Higher for all children | Controlled via Context boundaries |
| Ideal For          | Simple, small trees     | Shared/global state               |

---

## 7. Interview-Ready Answer

Prop drilling occurs when data is passed through multiple intermediate components just to reach a deeply nested child.
The **Context API** solves this problem by allowing global data sharing across components using `createContext`, `Provider`, and `useContext`.
It removes the need for manual prop passing, making code cleaner and easier to maintain.
Use Context for global data like authentication, theme, or user info — but avoid using it for rapidly changing data to prevent unnecessary re-renders.

