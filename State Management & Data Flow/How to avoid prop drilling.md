# How to avoid prop drilling

## Overview

**Prop drilling** occurs when data is passed down through multiple intermediate components just to reach a deeply nested child.
While React’s **one-way data flow** is powerful, excessive prop drilling can make code harder to maintain and understand.

Example of prop drilling problem:

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

Here, the `user` prop is passed through multiple layers (`Layout` → `Sidebar` → `UserInfo`), even though only the last component actually needs it.

---

## 1. What Is Prop Drilling

**Prop drilling** = passing props through components that don’t need them, just to reach a deeper component.

It causes:

* Repetitive and hard-to-read code.
* Tight coupling between components.
* Unnecessary re-renders if parent props change.
* Difficult maintenance as the component tree grows.

---

## 2. Ways to Avoid Prop Drilling

### **1️⃣ React Context API (Best Built-in Solution)**

The **Context API** provides a way to share data globally without passing props manually through every level.

#### Example:

```javascript
import { createContext, useContext, useState } from "react";

// 1. Create context
const UserContext = createContext();

// 2. Provide context value at top level
function App() {
  const [user] = useState({ name: "Alice" });
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

// 3. Access context anywhere
function UserInfo() {
  const user = useContext(UserContext);
  return <p>Hello, {user.name}</p>;
}

function Layout() {
  return <Sidebar />;
}

function Sidebar() {
  return <UserInfo />; // No props needed!
}
```

✅ **Advantages:**

* Eliminates unnecessary prop passing.
* Centralized data management.
* Easy to use for themes, auth, language, and global settings.

⚠️ **Use Context carefully:**

* Avoid putting rapidly changing data (like input states) in context — it can cause unnecessary re-renders.
* Use memoization (`React.memo` or `useMemo`) when providing complex objects.

---

### **2️⃣ useContext + useReducer (for global state)**

When state management becomes complex, combine `useReducer` with `useContext` to create a **lightweight global store**.

#### Example:

```javascript
const AppContext = createContext();

function reducer(state, action) {
  switch (action.type) {
    case "LOGIN":
      return { ...state, user: action.payload };
    default:
      return state;
  }
}

function AppProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, { user: null });
  return (
    <AppContext.Provider value={{ state, dispatch }}>
      {children}
    </AppContext.Provider>
  );
}

function useApp() {
  return useContext(AppContext);
}
```

Now, any component can use:

```javascript
const { state, dispatch } = useApp();
```

✅ Great for global state and actions (Redux alternative).

---

### **3️⃣ Component Composition**

Instead of drilling props through multiple layers, pass children components directly and render them where needed.

#### Example:

```javascript
function Card({ header, body, footer }) {
  return (
    <div className="card">
      <div>{header}</div>
      <div>{body}</div>
      <div>{footer}</div>
    </div>
  );
}

// Usage
<Card
  header={<h2>Title</h2>}
  body={<p>Some content</p>}
  footer={<button>OK</button>}
/>
```

✅ Reduces intermediate prop passing by using **composition** instead of prop forwarding.

---

### **4️⃣ State Management Libraries (For Larger Apps)**

For large-scale applications, consider global state management solutions:

* **Redux Toolkit** — Centralized store, predictable state updates.
* **Zustand** — Lightweight, minimal, and built with Hooks.
* **Recoil** — Provides reactive shared state management.
* **Jotai / MobX** — Alternative stores for global state.

These libraries help manage deeply nested or frequently changing state efficiently.

---

### **5️⃣ Custom Hooks**

Encapsulate logic in custom Hooks to share state or behavior without prop drilling.

#### Example:

```javascript
function useUser() {
  const [user, setUser] = useState({ name: "Alice" });
  return { user, setUser };
}

// Components can import and use this Hook directly:
function Profile() {
  const { user } = useUser();
  return <p>{user.name}</p>;
}
```

✅ Keeps state logic reusable and independent of component hierarchy.

---

## 3. Summary Table

| Solution                             | When to Use                                         | Advantages                         |
| ------------------------------------ | --------------------------------------------------- | ---------------------------------- |
| **Context API**                      | Small to medium apps, global settings (theme, auth) | Simple, built-in, no extra library |
| **useContext + useReducer**          | Global complex state logic                          | Centralized updates, scalable      |
| **Composition**                      | UI prop forwarding                                  | Cleaner, reusable components       |
| **State Libraries (Redux, Zustand)** | Large-scale apps                                    | Predictable, global, optimized     |
| **Custom Hooks**                     | Reusable logic across components                    | No context needed, easy testing    |

---

## 4. Interview-Ready Answer

Prop drilling occurs when data is passed through multiple intermediate components that don’t need it, just to reach a deeply nested child.
It can be avoided by using the **Context API** to provide data globally, **custom Hooks** to encapsulate shared logic, or **state management libraries** like Redux or Zustand for larger applications.
Component composition can also reduce prop passing by organizing components more efficiently.

