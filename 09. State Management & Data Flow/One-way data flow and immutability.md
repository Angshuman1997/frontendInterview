# One-way data flow and immutability

## Overview

React’s design is built around **one-way data flow** and **immutability**, two key principles that make UIs predictable, maintainable, and easy to debug.
Together, they ensure that React efficiently tracks state changes and updates only the necessary parts of the UI.

---

## 1. One-Way Data Flow

### Definition

**One-way data flow** means that data in a React application flows **in a single direction — from parent to child**.
The parent owns the state and passes it down to children through **props**, while children can communicate back using **callback functions**.

### Flow Direction

```
Parent Component → Child Component(s)
        ↓                  ↓
      Props             Render UI
```

* **Parent**: Holds the state.
* **Child**: Receives data and event handlers via props.
* **Child → Parent communication**: Achieved via callbacks (e.g., passing functions down as props).

---

### Example

```javascript
import { useState } from "react";

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <Child onIncrement={() => setCount(count + 1)} />
    </div>
  );
}

function Child({ onIncrement }) {
  return <button onClick={onIncrement}>Increment</button>;
}
```

Here:

1. The **Parent** holds the `count` state.
2. The **Child** receives a callback (`onIncrement`) as a prop.
3. When clicked, the **Child** triggers the parent’s function, updating the parent state.

✅ **Data flows downward**, while **actions flow upward**.
This keeps state management clear and predictable.

---

### Benefits of One-Way Data Flow

* **Predictable behavior:** You always know where state lives and how it changes.
* **Easier debugging:** Data changes are easy to trace up the component tree.
* **Improved performance:** Only affected components re-render when state changes.
* **Simpler reasoning:** Each component focuses on rendering based on its props.

---

## 2. Immutability in React

### Definition

**Immutability** means that state or data structures are **never modified directly**.
Instead, you **create a new copy** of the state with the updated values.

React relies on immutability to detect what has changed and efficiently update the Virtual DOM.

---

### Example — Mutable (❌ Wrong)

```javascript
const [user, setUser] = useState({ name: "Alice", age: 25 });

// ❌ This directly mutates the object — React may not detect the change
user.age = 26;
setUser(user);
```

React may **not re-render** because the reference to `user` hasn’t changed.

---

### Example — Immutable (✅ Correct)

```javascript
setUser(prevUser => ({ ...prevUser, age: 26 }));
```

✅ This creates a **new object** with a new reference.
React sees the change and triggers a re-render.

---

### Immutability with Arrays

#### ❌ Wrong (mutating array)

```javascript
const [items, setItems] = useState([1, 2, 3]);
items.push(4); // Mutates the array
setItems(items);
```

#### ✅ Correct (creating a new array)

```javascript
setItems(prev => [...prev, 4]);
```

✅ A new array is created and React re-renders accordingly.

---

### Why Immutability Matters

1. **Efficient Re-rendering (Virtual DOM):**
   React uses shallow comparison (`===`) to detect changes.
   If the reference changes, React knows the object has been updated.

2. **Predictable State Transitions:**
   State updates are easier to reason about — no side effects from shared references.

3. **Debugging and Time Travel:**
   Tools like Redux DevTools rely on immutability to “rewind” app state.

4. **Pure Components and Memoization:**
   Works well with `React.memo`, `useMemo`, and `useCallback`, which depend on immutable data for optimization.

---

### Example — Immutability in Practice

```javascript
function TodoList() {
  const [todos, setTodos] = useState(["Learn React"]);

  const addTodo = (todo) => {
    // ✅ Creates a new array instead of mutating
    setTodos(prevTodos => [...prevTodos, todo]);
  };

  return (
    <div>
      <button onClick={() => addTodo("Learn Hooks")}>Add</button>
      <ul>{todos.map((t, i) => <li key={i}>{t}</li>)}</ul>
    </div>
  );
}
```

Each time `addTodo` is called, React gets a new array reference and knows it must update the list.

---

## 3. Relationship Between One-Way Data Flow and Immutability

| Concept               | Role                                             | How It Supports React                                 |
| --------------------- | ------------------------------------------------ | ----------------------------------------------------- |
| **One-way data flow** | Ensures state flows in one predictable direction | Prevents circular updates and confusion               |
| **Immutability**      | Prevents direct data mutation                    | Allows React to detect and render changes efficiently |

Together, they enable:

* Predictable, consistent UIs
* Easier debugging and state management
* Efficient Virtual DOM diffing and updates

---

## 4. Interview-Ready Answer

React follows **one-way data flow**, where data moves from parent to child via props, and child components communicate back using callbacks.
To keep updates predictable, React enforces **immutability**, meaning state should never be modified directly — instead, you create new copies of data.
One-way data flow provides structure and clarity, while immutability ensures React efficiently detects and re-renders only what has changed.

