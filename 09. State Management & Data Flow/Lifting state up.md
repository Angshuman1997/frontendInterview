# Lifting state up

## Overview

**Lifting state up** is a React pattern where you move shared state from child components **up to their nearest common parent**.
This allows multiple components to share and stay in sync with the same data.

Instead of each child maintaining its own state, the parent holds the state and passes it down via **props**.
This ensures a single source of truth and maintains React’s **one-way data flow**.

---

## 1. The Problem: Duplicate State Across Components

When two or more components need to share the same data, each might try to manage its own state — causing them to fall out of sync.

### ❌ Example — Independent States (Problem)

```javascript
function CelsiusInput() {
  const [celsius, setCelsius] = useState("");
  return (
    <input
      value={celsius}
      onChange={(e) => setCelsius(e.target.value)}
      placeholder="Celsius"
    />
  );
}

function FahrenheitInput() {
  const [fahrenheit, setFahrenheit] = useState("");
  return (
    <input
      value={fahrenheit}
      onChange={(e) => setFahrenheit(e.target.value)}
      placeholder="Fahrenheit"
    />
  );
}
```

Here, both components maintain separate state, so they can’t stay in sync when one changes.

---

## 2. The Solution: Lifting State Up

To synchronize them:

* Move the state to the **common parent**.
* Pass the state and a setter function down as **props**.
* Each child updates the parent state when changed.

### ✅ Example — Lifted State in Parent

```javascript
import { useState } from "react";

function TemperatureConverter() {
  const [temperature, setTemperature] = useState("");

  return (
    <div>
      <CelsiusInput
        temperature={temperature}
        onTemperatureChange={setTemperature}
      />
      <FahrenheitDisplay temperature={temperature} />
    </div>
  );
}

function CelsiusInput({ temperature, onTemperatureChange }) {
  return (
    <input
      value={temperature}
      onChange={(e) => onTemperatureChange(e.target.value)}
      placeholder="Celsius"
    />
  );
}

function FahrenheitDisplay({ temperature }) {
  const fahrenheit = temperature ? (temperature * 9) / 5 + 32 : "";
  return <p>{fahrenheit && `${fahrenheit} °F`}</p>;
}
```

Here:

1. The **parent** (`TemperatureConverter`) holds the shared state.
2. **Children** (`CelsiusInput`, `FahrenheitDisplay`) receive data and handlers via props.
3. Updates flow upward from the child → parent, then downward from parent → other children.

✅ **Result:** Both inputs stay in sync through one shared source of truth.

---

## 3. Why Lifting State Up Is Important

### Benefits

* **Single Source of Truth:** Shared state is managed in one place.
* **Synchronized Components:** All components using the data are automatically updated.
* **Predictable Data Flow:** Easier to track how data moves.
* **Simpler Debugging:** You know exactly where the state is managed.

### When to Apply

Use lifting state up when:

* Multiple components depend on the same data.
* State from one component affects another.
* You need to perform shared calculations or validation.

---

## 4. Example — Shared Form Inputs

```javascript
function ParentForm() {
  const [formData, setFormData] = useState({ name: "", email: "" });

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  return (
    <div>
      <NameInput name={formData.name} onChange={handleChange} />
      <EmailInput email={formData.email} onChange={handleChange} />
      <Summary name={formData.name} email={formData.email} />
    </div>
  );
}

function NameInput({ name, onChange }) {
  return <input name="name" value={name} onChange={onChange} placeholder="Name" />;
}

function EmailInput({ email, onChange }) {
  return <input name="email" value={email} onChange={onChange} placeholder="Email" />;
}

function Summary({ name, email }) {
  return (
    <p>
      Name: {name} | Email: {email}
    </p>
  );
}
```

✅ Both inputs share the same state in the parent, keeping them synchronized.

---

## 5. Common Mistakes

❌ **Keeping duplicate state in multiple components**
→ Leads to inconsistencies and bugs.

❌ **Over-lifting state unnecessarily**
→ Makes the parent component too complex.

✅ **Best practice:**
Only lift state **to the lowest common ancestor** that needs to share it.

---

## 6. Alternatives to Lifting State

If lifting state makes components too nested or complex:

* Use **Context API** to share state globally.
* Use **state management libraries** (Redux, Zustand, Recoil) for larger apps.

---

## 7. Interview-Ready Answer

Lifting state up means moving shared state from child components to their nearest common parent.
The parent becomes the single source of truth, passing state and handler functions down as props.
This ensures synchronized data across components and maintains React’s one-way data flow.
Use it when multiple components depend on or modify the same piece of state.
