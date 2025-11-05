# Controlled vs Uncontrolled components

## Overview

In React, form elements can be handled in two ways:

* **Controlled Components:** React manages the form data via component state.
* **Uncontrolled Components:** The DOM itself manages the form data via references (`ref`).

Both approaches can be used to handle user input, but controlled components provide more control and predictability.

---

## 1. Controlled Components

### Definition

A **controlled component** is a form element whose value is controlled by React state.
Every change in the input triggers an event that updates the component’s state, which in turn re-renders the input with the new value.

### Example

```javascript
import { useState } from "react";

function ControlledForm() {
  const [name, setName] = useState("");

  const handleChange = (e) => {
    setName(e.target.value);
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log("Submitted name:", name);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" value={name} onChange={handleChange} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Key Points

* The input’s `value` is bound to React state.
* Changes are handled through `onChange` events.
* React fully controls the form data.
* The UI is always in sync with the component’s state.

✅ **Advantages**

* Predictable and consistent UI.
* Easy validation and data manipulation.
* Centralized form state management.

❌ **Disadvantages**

* Slightly more code.
* Frequent re-renders if not optimized.

---

## 2. Uncontrolled Components

### Definition

An **uncontrolled component** stores its form data in the DOM instead of React state.
You access the data using `ref` (a reference to the actual DOM element).

### Example

```javascript
import { useRef } from "react";

function UncontrolledForm() {
  const nameRef = useRef();

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log("Submitted name:", nameRef.current.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" ref={nameRef} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Key Points

* The input’s value is handled by the DOM, not React state.
* You read the value using `ref.current.value`.
* React doesn’t re-render on each input change.

✅ **Advantages**

* Less re-rendering.
* Useful for quick or simple forms.

❌ **Disadvantages**

* Harder to perform validation.
* React can’t easily track or control user input.
* More difficult to debug and test.

---

## 3. Comparison Table

| Feature               | Controlled Component      | Uncontrolled Component          |
| --------------------- | ------------------------- | ------------------------------- |
| Data Source           | React state               | DOM (via ref)                   |
| Value Change Handling | `onChange` + `setState`   | Access via `ref.current.value`  |
| Real-time Validation  | Easy                      | Harder                          |
| React Control         | Full                      | Minimal                         |
| Code Complexity       | Slightly higher           | Simpler                         |
| Common Use            | Dynamic forms, validation | Quick input forms, file uploads |

---

## 4. When to Use

**Use Controlled Components When:**

* You need to validate or format input (e.g., trimming, limiting input).
* Form data is needed during typing.
* You want predictable, React-managed state.

**Use Uncontrolled Components When:**

* The form is simple or temporary.
* Performance is critical and state management is unnecessary.
* You just need to access data once (like on form submit).

---

## 5. Hybrid Approach (Controlled + Uncontrolled)

You can combine both approaches — for example, using controlled components for key fields and refs for optional fields.

---

## 6. Interview-Ready Answer

In React, **controlled components** store input values in component state and update via `onChange` handlers, making React the single source of truth.
**Uncontrolled components** rely on the DOM to manage their state and use refs to access values.
Controlled components provide better control, validation, and synchronization with React, while uncontrolled components are simpler but less predictable.
