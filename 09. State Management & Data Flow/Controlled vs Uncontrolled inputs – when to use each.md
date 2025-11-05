# Controlled vs Uncontrolled inputs - when to use each

## Overview

In React, **inputs** (like `<input>`, `<textarea>`, and `<select>`) can be managed in two main ways:

* **Controlled Inputs** — React fully manages the input’s value through state.
* **Uncontrolled Inputs** — The DOM handles the input’s value internally, and React accesses it only when needed via a `ref`.

Choosing between them depends on how much control and synchronization you need between the UI and React’s state.

---

## 1. Controlled Inputs

### Definition

A controlled input’s value is **bound to React state**.
Every change to the input triggers an update in the state, and React re-renders the component with the new value.

### Example

```javascript
import { useState } from "react";

function ControlledInput() {
  const [email, setEmail] = useState("");

  return (
    <div>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Enter your email"
      />
      <p>{email}</p>
    </div>
  );
}
```

### When to Use Controlled Inputs

✅ Use **Controlled Inputs** when:

1. **You need real-time validation or formatting**

   * Example: validating email, enforcing numeric input, masking phone numbers.

   ```javascript
   if (value.includes("@")) setIsValid(true);
   ```

2. **You need to react immediately to user input**

   * Example: live search filters, auto-suggestions, form wizards.

3. **You need the input data during typing**

   * The React state always contains the latest input value.

4. **You want predictable state and one source of truth**

   * React’s state is always in sync with the input field.

5. **You need to reset or control input programmatically**

   * Example: clearing all fields after successful submission.

### Advantages

* Centralized and predictable state management.
* Easy to validate, manipulate, and test.
* Synchronized UI with React’s virtual DOM.

### Disadvantages

* More code and re-renders.
* Slight performance cost for very large forms.

---

## 2. Uncontrolled Inputs

### Definition

An uncontrolled input’s value is managed by the **DOM**, not React.
You use a **ref** to read the input value only when needed (like on form submission).

### Example

```javascript
import { useRef } from "react";

function UncontrolledInput() {
  const inputRef = useRef();

  const handleSubmit = () => {
    alert(`Entered value: ${inputRef.current.value}`);
  };

  return (
    <div>
      <input type="text" ref={inputRef} placeholder="Enter your name" />
      <button onClick={handleSubmit}>Submit</button>
    </div>
  );
}
```

### When to Use Uncontrolled Inputs

✅ Use **Uncontrolled Inputs** when:

1. **You only need the value once**

   * Example: reading a value when the user submits a form.

2. **You don’t need live updates or validation**

   * Example: forms where inputs don’t affect UI while typing.

3. **You want less overhead for quick or simple inputs**

   * React doesn’t need to re-render for each keystroke.

4. **You’re integrating with non-React libraries**

   * Example: jQuery plugins, form libraries that handle DOM values directly.

### Advantages

* Minimal re-rendering → better performance in simple use cases.
* Less state management boilerplate.
* Useful for uncontrolled form libraries or file uploads.

### Disadvantages

* Harder to perform real-time validation or dynamic behavior.
* Data not synced with React state.
* Difficult to test and debug.

---

## 3. Quick Comparison Table

| Feature              | Controlled Input               | Uncontrolled Input               |
| -------------------- | ------------------------------ | -------------------------------- |
| Data Managed By      | React State                    | DOM                              |
| Value Access         | `state` variable               | `ref.current.value`              |
| Real-time Validation | Easy                           | Difficult                        |
| One Source of Truth  | React                          | DOM                              |
| Performance          | Slightly slower (more renders) | Faster for simple forms          |
| Use Case             | Dynamic, interactive forms     | Simple, static forms             |
| Example              | Live validation, search bars   | Basic input fields, file uploads |

---

## 4. When to Choose Which

| Scenario                                      | Recommended Type | Reason                                       |
| --------------------------------------------- | ---------------- | -------------------------------------------- |
| Real-time form validation                     | Controlled       | Easier to validate on each keystroke         |
| File uploads or 3rd-party library integration | Uncontrolled     | File inputs can’t be fully controlled        |
| Simple form, submit-only access               | Uncontrolled     | No need for extra re-renders                 |
| Large multi-step form                         | Controlled       | Predictable and centralized data handling    |
| Input resets or dynamic changes               | Controlled       | State can be programmatically cleared or set |

---

## 5. Interview-Ready Answer

A **controlled input** is one where React manages the input’s value via state, making it the single source of truth.
An **uncontrolled input** relies on the DOM to handle its value, and React accesses it using refs.

Use **controlled inputs** when you need real-time validation, dynamic updates, or predictable state management.
Use **uncontrolled inputs** when you only need to read the value once (e.g., on form submission) or for simple, static forms that don’t require React’s control.
