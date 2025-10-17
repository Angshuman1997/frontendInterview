# When to use useReducer over useState

## Overview

Both `useState` and `useReducer` are React Hooks used to manage component state.
While `useState` is ideal for simple, isolated state updates, `useReducer` is better suited for **complex or interrelated state logic** where multiple values or actions modify the state in predictable ways.

---

## 1. useState — Simple State Logic

### Use Case

When you’re managing **independent or simple state variables**.

### Example

```javascript
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

✅ **Best for:**

* Managing small pieces of state (like toggles, counters, form fields)
* Straightforward updates (`setValue(newValue)`)

---

## 2. useReducer — Complex State Logic

### Use Case

When state transitions depend on **multiple actions** or when you want to group related state logic together in a predictable way.

### Syntax

```javascript
const [state, dispatch] = useReducer(reducer, initialState);
```

### Example

```javascript
import { useReducer } from "react";

const initialState = { count: 0, step: 1 };

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + state.step };
    case "decrement":
      return { ...state, count: state.count - state.step };
    case "reset":
      return initialState;
    case "setStep":
      return { ...state, step: action.payload };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      <input
        type="number"
        value={state.step}
        onChange={e => dispatch({ type: "setStep", payload: Number(e.target.value) })}
      />
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>−</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

✅ **Best for:**

* Multiple related state variables that change together.
* Complex update logic with multiple conditions.
* When actions can describe how state changes (similar to Redux).
* When you need to centralize and simplify large state management logic.

---

## 3. Key Differences

| Feature         | useState                         | useReducer                              |
| --------------- | -------------------------------- | --------------------------------------- |
| Complexity      | Simple state logic               | Complex, structured logic               |
| State Structure | Usually single values            | Object with multiple values             |
| Update Method   | Direct state setter (`setValue`) | Dispatch actions (`dispatch({ type })`) |
| Readability     | Great for small components       | Better for scalable logic               |
| Debugging       | Harder for multi-state updates   | Predictable via reducer actions         |
| Performance     | Slightly faster for small state  | Better for managing grouped updates     |

---

## 4. When to Prefer useReducer Over useState

Use **`useReducer`** when:

1. The state involves **multiple sub-values** that change together.
   Example: managing a form or a complex UI component.

2. The state update logic is **complex or multi-step**.
   Example: multiple conditions or branching logic.

3. You want to **separate business logic** (reducer) from UI logic.
   This improves readability and maintainability.

4. You want to implement a **predictable state flow** similar to Redux.
   Actions define how the state transitions occur.

5. You’re using **context for global state**, as `useReducer` integrates cleanly with `useContext`.

---

## 5. Combined Example — Form Handling

```javascript
import { useReducer } from "react";

const initialState = { name: "", email: "", subscribed: false };

function reducer(state, action) {
  switch (action.type) {
    case "SET_NAME":
      return { ...state, name: action.payload };
    case "SET_EMAIL":
      return { ...state, email: action.payload };
    case "TOGGLE_SUBSCRIBE":
      return { ...state, subscribed: !state.subscribed };
    case "RESET":
      return initialState;
    default:
      return state;
  }
}

function SignupForm() {
  const [state, dispatch] = useReducer(reducer, initialState);

  const handleSubmit = e => {
    e.preventDefault();
    console.log(state);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={state.name}
        onChange={e => dispatch({ type: "SET_NAME", payload: e.target.value })}
        placeholder="Name"
      />
      <input
        value={state.email}
        onChange={e => dispatch({ type: "SET_EMAIL", payload: e.target.value })}
        placeholder="Email"
      />
      <label>
        <input
          type="checkbox"
          checked={state.subscribed}
          onChange={() => dispatch({ type: "TOGGLE_SUBSCRIBE" })}
        />
        Subscribe
      </label>
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## 6. Interview-Ready Answer

`useState` is ideal for managing simple, independent pieces of state, while `useReducer` is used for more complex state logic involving multiple related values or actions.
`useReducer` centralizes update logic in a reducer function, making it easier to maintain and test.
Use `useReducer` when state updates depend on previous state values or multiple actions, or when you need a predictable, structured state management pattern similar to Redux.

