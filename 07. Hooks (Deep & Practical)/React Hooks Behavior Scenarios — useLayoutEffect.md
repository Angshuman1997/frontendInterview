# React Hooks Behavior Scenarios — `useLayoutEffect`

## Overview

`useLayoutEffect` is a React hook that runs **synchronously after React has performed DOM mutations but before the browser paints the screen**.
It is similar to `useEffect`, but its timing is different — it executes **earlier**, allowing you to measure or modify the DOM **before the browser visually updates**.

This makes it perfect for:

* Measuring DOM elements (size, position)
* Adjusting styles or scroll positions
* Preventing layout flickers during updates

---

## 1️⃣ Basic Example

```jsx
import { useLayoutEffect, useRef } from "react";

function Box() {
  const boxRef = useRef();

  useLayoutEffect(() => {
    const { width, height } = boxRef.current.getBoundingClientRect();
    console.log("Box size:", width, height);
  }, []);

  return <div ref={boxRef} style={{ width: 200, height: 100, background: "tomato" }} />;
}
```

✅ **Explanation**

* Runs **after DOM changes** but **before the browser paints**.
* Ensures that any measurements (like width/height) are accurate before the user sees the page.

---

## 2️⃣ Difference Between `useEffect` and `useLayoutEffect`

| Feature                  | `useEffect`            | `useLayoutEffect`                      |
| ------------------------ | ---------------------- | -------------------------------------- |
| Execution timing         | After paint (async)    | Before paint (sync)                    |
| Affects user experience? | May cause flicker      | Prevents flicker                       |
| Use case                 | Async logic, API calls | DOM reads/writes, scroll, measurements |
| Performance impact       | Non-blocking           | Blocking (if heavy)                    |

✅ **Rule:**

> Always use `useEffect` by default.
> Use `useLayoutEffect` **only** when you need precise DOM read/write synchronization.

---

## 3️⃣ Measuring DOM Layout Before Paint

```jsx
function Tooltip() {
  const ref = useRef();
  const [position, setPosition] = useState(0);

  useLayoutEffect(() => {
    const rect = ref.current.getBoundingClientRect();
    setPosition(rect.top);
  }, []);

  return (
    <div ref={ref} style={{ position: "absolute", top: position }}>
      Tooltip here
    </div>
  );
}
```

✅ **Explanation**

* Reads element position before paint → avoids visible "jump" after rendering.
* Ensures layout is correct from the first frame.

---

## 4️⃣ Preventing Flicker During Layout Updates

```jsx
function Modal({ isOpen }) {
  const ref = useRef();

  useLayoutEffect(() => {
    if (isOpen) {
      document.body.style.overflow = "hidden";
    } else {
      document.body.style.overflow = "";
    }
  }, [isOpen]);

  return isOpen ? <div ref={ref}>Modal Opened</div> : null;
}
```

✅ **Explanation**

* Ensures the body overflow is adjusted before the screen re-renders.
* Prevents flickering or scrolling glitches when toggling UI components.

---

## 5️⃣ Adjusting Scroll Before Paint

```jsx
function ChatBox({ messages }) {
  const ref = useRef();

  useLayoutEffect(() => {
    ref.current.scrollTop = ref.current.scrollHeight;
  }, [messages]);

  return (
    <div ref={ref} style={{ height: 200, overflowY: "auto" }}>
      {messages.map((msg) => (
        <p key={msg.id}>{msg.text}</p>
      ))}
    </div>
  );
}
```

✅ **Explanation**

* Adjusts scroll position *before paint*, so the user never sees the old scroll position.
* Essential for chat or log viewers.

---

## 6️⃣ Cleanup Runs Before Next Layout or Unmount

```jsx
useLayoutEffect(() => {
  console.log("Mounted");
  return () => console.log("Cleanup before next paint or unmount");
}, []);
```

✅ **Explanation**

* Cleanup happens synchronously **before the next layout effect or unmount**, not after paint.
* Similar to `componentWillUnmount`.

---

## 7️⃣ Reading and Writing Layout Together (Synchronous)

```jsx
useLayoutEffect(() => {
  const width = ref.current.offsetWidth; // Read layout
  ref.current.style.height = `${width}px`; // Write layout
});
```

✅ **Explanation**

* Useful for keeping an element square (height = width).
* Running synchronously ensures consistent layout before browser paint.

---

## 8️⃣ Avoid SSR Warnings (Next.js, Remix, etc.)

```jsx
// Prevent SSR warning: "useLayoutEffect does nothing on the server"
const useIsomorphicLayoutEffect =
  typeof window !== "undefined" ? useLayoutEffect : useEffect;
```

✅ **Explanation**

* `useLayoutEffect` depends on the DOM — not available during server-side rendering.
* This pattern safely replaces it with `useEffect` in SSR environments.

---

## 9️⃣ Perfect for Animations (Prevent Frame Jump)

```jsx
function AnimatedBox({ x }) {
  const ref = useRef();

  useLayoutEffect(() => {
    ref.current.style.transform = `translateX(${x}px)`;
  }, [x]);

  return <div ref={ref} className="animated-box" />;
}
```

✅ **Explanation**

* Ensures style updates happen before paint → avoids a single-frame flicker between states.
* Common in custom animation implementations.

---

## 🔟 Lifecycle Analogy

| Class Component        | Equivalent Hook                        |
| ---------------------- | -------------------------------------- |
| `componentDidMount`    | `useLayoutEffect(() => {...}, [])`     |
| `componentDidUpdate`   | `useLayoutEffect(() => {...}, [deps])` |
| `componentWillUnmount` | Cleanup function in `useLayoutEffect`  |

✅ **Explanation**
`useLayoutEffect` matches the lifecycle timing of `componentDidMount` and `componentDidUpdate` — but runs **before** browser paint.

---

## 11️⃣ Order of Execution

1. React renders the component.
2. React commits DOM updates.
3. **`useLayoutEffect` runs synchronously.**
4. Browser paints the UI.
5. `useEffect` runs asynchronously afterward.

✅ **Visualization:**

```
Render → Commit DOM → useLayoutEffect → Paint → useEffect
```

---

## 12️⃣ Performance Consideration

❌ **Bad Example**

```jsx
useLayoutEffect(() => {
  heavyComputation(); // ❌ Blocks render
});
```

✅ **Good Example**

```jsx
useLayoutEffect(() => {
  measureAndAdjust(); // ✅ Quick DOM read/write
});
```

**Rule:**
Keep it **fast and minimal**, since it blocks painting.

---

## 13️⃣ Synchronizing Layout Between Components

```jsx
function Parent() {
  const [height, setHeight] = useState(0);
  const ref = useRef();

  useLayoutEffect(() => {
    setHeight(ref.current.offsetHeight);
  });

  return (
    <>
      <div ref={ref}>Parent Content</div>
      <Child offset={height} />
    </>
  );
}
```

✅ **Explanation**

* Ensures the child receives an accurate layout measurement before it renders.
* Prevents incorrect positioning.

---

## 14️⃣ Real-World Example: React Animation Libraries

Libraries like **Framer Motion**, **React Spring**, and **GSAP React** use `useLayoutEffect` internally to:

* Measure DOM elements before starting animations.
* Sync animation frames with React updates.
* Avoid layout flicker during motion.

✅ **Key Idea:**
For **synchronous visual updates**, animation libraries rely on `useLayoutEffect` instead of `useEffect`.

---

## 15️⃣ Common Mistakes

❌ **1. Doing async work**

```jsx
useLayoutEffect(async () => { await fetch("/data"); });
```

✅ **Fix:**
Async operations should use `useEffect` instead.

---

❌ **2. Using on the server**

* Causes hydration warnings in SSR apps.
  ✅ Use `useIsomorphicLayoutEffect` (see example above).

---

❌ **3. Heavy synchronous operations**

* Blocks paint and makes UI feel laggy.
  ✅ Keep it light — only DOM reads/writes.

---

## Summary Table

| #  | Scenario          | Behavior                       | Key Takeaway          |
| -- | ----------------- | ------------------------------ | --------------------- |
| 1  | Basic usage       | After DOM update, before paint | Perfect for measuring |
| 2  | vs useEffect      | Synchronous pre-paint          | Avoid flicker         |
| 3  | DOM measurement   | Layout read/write              | Accurate results      |
| 4  | Prevent flicker   | Adjust layout pre-paint        | Better UX             |
| 5  | Scroll updates    | Before paint                   | Prevent jumps         |
| 6  | Cleanup timing    | Before unmount/next effect     | Predictable           |
| 7  | Sync read/write   | Synchronous layout logic       | Smooth transitions    |
| 8  | SSR safety        | useIsomorphicLayoutEffect      | Prevent warnings      |
| 9  | Animations        | Runs before paint              | Frame-perfect motion  |
| 10 | Lifecycle match   | Mount/Update/Unmount           | Same timing           |
| 11 | Execution order   | Pre-paint effect               | Deterministic flow    |
| 12 | Performance       | Can block paint                | Keep lightweight      |
| 13 | Parent-child sync | Pass measured data             | Layout coordination   |
| 14 | Used in libs      | Framer Motion, etc.            | Animation sync        |
| 15 | Common mistakes   | Async, SSR, heavy ops          | Avoid misuse          |

---

## Interview-Ready Answer

`useLayoutEffect` is similar to `useEffect`, but it runs **synchronously after the DOM has been updated and before the browser paints the UI**.
It’s used for **DOM measurements**, **scroll adjustments**, and **synchronous visual updates** to prevent flicker or incorrect layouts.

Unlike `useEffect`, it **blocks painting**, so you should keep it minimal.
In server-rendered environments, use an **isomorphic hook** fallback to avoid warnings.
It’s often used internally by animation and layout libraries for frame-perfect UI updates.

---

✅ **Key takeaway:**

> Use `useEffect` for side effects and async logic.
> Use `useLayoutEffect` **only** for layout synchronization or DOM measurement **before paint**.
