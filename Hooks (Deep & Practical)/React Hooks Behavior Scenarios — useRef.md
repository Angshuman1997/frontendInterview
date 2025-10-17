# React Hooks Behavior Scenarios — `useRef`

## Overview

`useRef` is a React Hook that provides a **mutable reference object** with a `.current` property.
Unlike state, **changing a ref does not trigger a re-render**.
It’s used to:

* Access and manipulate **DOM elements** directly.
* Persist **mutable values** across renders (without re-rendering).
* Store **previous values** or **timeouts/intervals**.

---

## 1️⃣ Basic Example — DOM Access

```jsx
import { useRef, useEffect } from "react";

function InputFocus() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus(); // Auto-focus input on mount
  }, []);

  return <input ref={inputRef} placeholder="Focus me automatically" />;
}
```

✅ **Explanation**

* `useRef(null)` initializes a reference object `{ current: null }`.
* When assigned to `ref` on an element, React populates `.current` with the DOM node.
* Changing `.current` does **not** cause re-renders.

---

## 2️⃣ Persistent Mutable Value (No Re-render)

```jsx
function Counter() {
  const countRef = useRef(0);

  const increment = () => {
    countRef.current += 1;
    console.log("Current count:", countRef.current);
  };

  return <button onClick={increment}>Count: {countRef.current}</button>;
}
```

✅ **Explanation**

* `countRef.current` updates without triggering re-render.
* UI won’t show changes automatically since it’s not tied to React state.
* Useful for **tracking values between renders** (like timers, scroll positions, etc.).

---

## 3️⃣ useRef vs useState

| Feature                         | **useRef** | **useState** |
| ------------------------------- | ---------- | ------------ |
| Triggers re-render on change?   | ❌ No       | ✅ Yes        |
| Stores mutable data?            | ✅ Yes      | ✅ Yes        |
| Used for DOM access?            | ✅ Yes      | ❌ No         |
| Value persists between renders? | ✅ Yes      | ✅ Yes        |

---

## 4️⃣ Storing Previous Value Between Renders

```jsx
function PreviousValue({ value }) {
  const prevValue = useRef();

  useEffect(() => {
    prevValue.current = value;
  }, [value]);

  return (
    <div>
      <p>Current: {value}</p>
      <p>Previous: {prevValue.current}</p>
    </div>
  );
}
```

✅ **Explanation**

* `prevValue.current` holds the previous render’s `value`.
* Updates in `useEffect` after render, so it lags one render behind.
* Common interview question: *“How do you track the previous prop/state in React?”*

---

## 5️⃣ useRef for Timers or Intervals

```jsx
function Timer() {
  const intervalRef = useRef(null);
  const [seconds, setSeconds] = useState(0);

  const start = () => {
    if (intervalRef.current) return;
    intervalRef.current = setInterval(() => setSeconds(s => s + 1), 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
    intervalRef.current = null;
  };

  useEffect(() => () => clearInterval(intervalRef.current), []);

  return (
    <div>
      <p>{seconds}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

✅ **Explanation**

* `intervalRef` holds interval ID persistently.
* Not part of render cycle → updating it doesn’t cause re-render.
* Cleanup ensures no memory leaks when unmounted.

---

## 6️⃣ Tracking Render Count (Classic useRef Example)

```jsx
function RenderCounter() {
  const renders = useRef(0);
  const [value, setValue] = useState("");

  useEffect(() => {
    renders.current += 1;
  });

  return (
    <div>
      <input value={value} onChange={(e) => setValue(e.target.value)} />
      <p>Rendered {renders.current} times</p>
    </div>
  );
}
```

✅ **Explanation**

* `renders.current` persists and updates each render.
* Doesn’t cause another re-render — hence safe to track render count.

---

## 7️⃣ Storing Function References Safely

```jsx
function CallbackExample({ onSubmit }) {
  const callbackRef = useRef(onSubmit);

  useEffect(() => {
    callbackRef.current = onSubmit; // Always store latest callback
  });

  function handleClick() {
    callbackRef.current(); // Calls latest version
  }

  return <button onClick={handleClick}>Run Callback</button>;
}
```

✅ **Explanation**

* Prevents stale closure issues by always storing the latest function in `.current`.
* Common for event listeners or debounced actions.

---

## 8️⃣ Accessing Child Component’s DOM Node

```jsx
function ChildComponent(props, ref) {
  return <input ref={ref} />;
}
const ForwardedChild = React.forwardRef(ChildComponent);

function Parent() {
  const inputRef = useRef();
  useEffect(() => {
    inputRef.current.focus();
  }, []);
  return <ForwardedChild ref={inputRef} />;
}
```

✅ **Explanation**

* You can forward a ref from parent → child using `React.forwardRef`.
* Allows parent components to directly access child DOM elements.

---

## 9️⃣ `useRef` for Avoiding Effect Re-run Loops

```jsx
function DataFetcher({ url }) {
  const prevUrl = useRef();

  useEffect(() => {
    if (prevUrl.current === url) return;
    prevUrl.current = url;

    fetch(url).then(res => res.json()).then(console.log);
  }, [url]);
}
```

✅ **Explanation**

* Prevents redundant fetch calls if the same `url` is passed repeatedly.
* Stores last fetched URL safely without triggering renders.

---

## 🔟 `useRef` and `useEffect` — Stable References

```jsx
function StableHandler() {
  const handlerRef = useRef(() => console.log("Hello"));
  useEffect(() => {
    const handler = handlerRef.current;
    document.addEventListener("click", handler);
    return () => document.removeEventListener("click", handler);
  }, []);
}
```

✅ **Explanation**

* Stores stable event handler reference.
* Prevents re-adding/removing listeners on every re-render.

---

## 11️⃣ Avoiding Re-renders with useRef Instead of State

```jsx
function ChatScroll() {
  const lastMessageRef = useRef(null);

  useEffect(() => {
    lastMessageRef.current?.scrollIntoView({ behavior: "smooth" });
  });

  return (
    <div>
      <div>...messages...</div>
      <div ref={lastMessageRef} />
    </div>
  );
}
```

✅ **Explanation**

* Ref tracks the last message element without triggering render.
* Efficient for continuous updates like chat apps or infinite scroll.

---

## 12️⃣ useRef as a Flag (Mount Tracking)

```jsx
function Component() {
  const isMounted = useRef(false);

  useEffect(() => {
    if (isMounted.current) {
      console.log("Component updated");
    } else {
      isMounted.current = true;
    }
  });
}
```

✅ **Explanation**

* Tracks whether the component has mounted already.
* Useful to skip initial effect behavior (like API calls).

---

## 13️⃣ Resetting Ref Values

```jsx
function Example() {
  const ref = useRef(0);
  const reset = () => (ref.current = 0);
}
```

✅ **Explanation**

* You can freely mutate `.current` anytime.
* It doesn’t affect render performance or cause re-rendering.

---

## 14️⃣ Common Mistake — Expecting Re-render

```jsx
function Example() {
  const countRef = useRef(0);
  const handleClick = () => {
    countRef.current++;
    console.log(countRef.current);
  };
  return <button onClick={handleClick}>{countRef.current}</button>;
}
```

❌ **Issue**

* Updating `.current` doesn’t re-render the component.
* Button label won’t update visually.

✅ **Fix**
Use `useState` if you want UI updates:

```jsx
const [count, setCount] = useState(0);
```

---

## 15️⃣ Combining useRef + useState

```jsx
function Stopwatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
    if (intervalRef.current) return;
    intervalRef.current = setInterval(() => setTime(t => t + 1), 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
    intervalRef.current = null;
  };

  const reset = () => {
    stop();
    setTime(0);
  };

  return (
    <div>
      <p>{time}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

✅ **Explanation**

* `useState` updates UI.
* `useRef` stores interval ID (mutable, no re-renders).
* Best pattern for timers or animations.

---

## Summary Table

| #  | Scenario            | Behavior                           | Key Takeaway                  |
| -- | ------------------- | ---------------------------------- | ----------------------------- |
| 1  | DOM access          | Get element via `ref`              | Direct element control        |
| 2  | Mutable values      | Updates persist w/o re-render      | For counters, caches          |
| 3  | useRef vs useState  | State triggers render; ref doesn’t | Use wisely                    |
| 4  | Previous value      | Tracks prior state                 | Common pattern                |
| 5  | Timers              | Store intervals safely             | Prevent leaks                 |
| 6  | Render count        | Count re-renders safely            | Debugging tool                |
| 7  | Function refs       | Avoid stale closures               | Keep callback current         |
| 8  | Forward refs        | Access child DOM                   | Parent-to-child communication |
| 9  | Avoid effect loops  | Skip repeated fetches              | Compare refs                  |
| 10 | Stable handlers     | Keep one event listener            | Performance                   |
| 11 | Scroll tracking     | Manage scroll DOM                  | Efficient updates             |
| 12 | Mount flag          | Detect mount/update                | Skips initial runs            |
| 13 | Reset refs          | Manual reset                       | No re-render                  |
| 14 | Expecting re-render | Wrong — useState needed            | Important gotcha              |
| 15 | Combine with state  | Best of both worlds                | Practical timers, counters    |

---

## Interview-Ready Answer

`useRef` is a hook that stores a **mutable reference** that persists across renders **without causing re-renders** when changed.
It’s commonly used for:

* **Accessing DOM elements**,
* **Persisting mutable data** (like timers, counters, or previous values),
* **Avoiding stale closures** in async callbacks, and
* **Tracking renders or mount state**.

Unlike `useState`, updating a ref doesn’t trigger re-render, making it perfect for non-visual data persistence and performance optimization.
