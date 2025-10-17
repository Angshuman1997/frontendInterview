# Props and State in React

## Overview

In React, **props** and **state** are two key concepts that control how components receive and manage data.
They both trigger re-renders when their values change, but they serve **different purposes**:

* **Props** → For *data flow into a component* (read-only).
* **State** → For *data managed inside a component* (mutable).

---

## 1. Props (Properties)

### Definition

**Props** are short for *properties*.
They are used to **pass data from a parent component to a child component** and are **immutable** inside the child.

### Key Points

* Passed **from parent → child**.
* **Read-only**: a child should never modify its props.
* Used to make components **reusable and configurable**.
* Trigger re-renders when the parent passes new prop values.

---

### Example

```jsx
function Welcome(props) {
  return <h2>Hello, {props.name}!</h2>;
}

function App() {
  return (
    <>
      <Welcome name="Alice" />
      <Welcome name="Bob" />
    </>
  );
}
```

✅ The `Welcome` component receives `name` as a prop and renders personalized output.

### Destructuring Props

```jsx
function Welcome({ name }) {
  return <h2>Hello, {name}!</h2>;
}
```

---

### Props Are Immutable

React components **should not modify props** directly:

```jsx
function WrongExample({ count }) {
  count++; // ❌ Wrong — props are read-only
}
```

To modify data, store it in **state** (explained below).

---

## 2. State

### Definition

**State** is **local data managed by the component itself**.
It can be updated over time (e.g., by user interactions or async responses).

### Key Points

* Managed inside the component.
* **Mutable** (via `setState` in class or `useState` in function components).
* Updating state triggers a **re-render**.
* Used for dynamic or interactive UI behavior.

---

### Example — Using `useState`

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0); // initialize state

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

✅ `count` is the current state value.
✅ `setCount` updates the value and causes React to re-render the component.

---

### State Is Local to the Component

Each component instance maintains its own isolated state:

```jsx
function App() {
  return (
    <>
      <Counter />  {/* independent state */}
      <Counter />  {/* independent state */}
    </>
  );
}
```

Each `<Counter>` maintains its own `count` — updating one doesn’t affect the other.

---

## 3. Props vs State — Comparison Table

| Feature            | **Props**                          | **State**                              |
| ------------------ | ---------------------------------- | -------------------------------------- |
| **Definition**     | Data passed *to* a component       | Data managed *within* a component      |
| **Mutability**     | Immutable                          | Mutable (via `setState` or `useState`) |
| **Ownership**      | Owned by parent component          | Owned by the component itself          |
| **Usage**          | For configuration and passing data | For internal data and interactivity    |
| **Change Trigger** | Parent re-render                   | Component updates its own data         |
| **Accessibility**  | Passed via JSX attributes          | Declared and used inside component     |
| **Examples**       | `props.name`, `props.age`          | `useState`, `this.state`               |

---

## 4. Props + State Together

They often work **together**:

* **Props** define *initial data or configuration*.
* **State** handles *dynamic changes* inside the component.

### Example

```jsx
function Greeting({ initialName }) {
  const [name, setName] = useState(initialName);

  return (
    <div>
      <p>Hello, {name}!</p>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
    </div>
  );
}

// Usage
<Greeting initialName="Alice" />
```

✅ `initialName` is a **prop** passed from the parent.
✅ `name` is **state** that changes when the user types.

---

## 5. Common Interview Questions

### Q1. What’s the difference between props and state?

**Answer:**
Props are read-only data passed from parent to child, while state is data managed locally within a component that can change over time. Changing state triggers re-render; props changes depend on parent updates.

---

### Q2. Can you modify props inside a component?

**Answer:**
No. Props are immutable. To change a value, the parent must pass new props or manage it through component state.

---

### Q3. When to use props vs state?

**Answer:**
Use **props** for static or external data (config, parent data).
Use **state** for dynamic, interactive data (user input, API response, toggles).

---

### Q4. How do props and state trigger re-rendering?

**Answer:**
When props or state values change, React’s reconciliation process compares the new virtual DOM to the previous one and re-renders only the affected parts.

---

## 6. Summary

| Concept   | Purpose                  | Managed By       | Mutable | Triggers Re-render        |
| --------- | ------------------------ | ---------------- | ------- | ------------------------- |
| **Props** | Receive data from parent | Parent component | ❌ No    | ✅ Yes (if parent changes) |
| **State** | Manage local data        | Same component   | ✅ Yes   | ✅ Yes                     |

---

## 7. Interview-Ready Answer

In React, **props** are used to pass data from parent to child components and are **immutable**, while **state** is local data managed by the component itself and can be **updated dynamically**.
Updating state triggers a re-render, whereas props are updated only when the parent passes new values.
Use props for *data input/configuration* and state for *interactive, changing data within the component*.
