# React Hooks Behavior Scenarios — `useNavigate`

## Overview

`useNavigate` is a React Router hook used to **navigate programmatically** between routes in your app.
It replaces the older `useHistory` API from React Router v5.

It’s part of **React Router v6** and works only inside components wrapped by `<BrowserRouter>` or `<Router>`.

---

## 1️⃣ Basic Usage

```jsx
import { useNavigate } from "react-router-dom";

function Home() {
  const navigate = useNavigate();

  const goToProfile = () => {
    navigate("/profile");
  };

  return <button onClick={goToProfile}>Go to Profile</button>;
}
```

✅ **Explanation**

* `useNavigate()` returns a `navigate` function.
* Calling `navigate(path)` changes the URL and renders the corresponding route.
* Works exactly like clicking a `<Link>` component, but programmatically.

✅ **Render Count:**

* Calling `navigate()` does **not re-render** current component.
* Only the **routed target** component re-renders.

---

## 2️⃣ Navigation with Parameters

```jsx
navigate(`/user/${userId}`);
```

✅ **Explanation**

* You can dynamically interpolate route params.
* Works seamlessly with routes defined like:

  ```jsx
  <Route path="/user/:id" element={<UserPage />} />
  ```

---

## 3️⃣ Navigation with State (Passing Data)

```jsx
navigate("/details", { state: { id: 5, from: "home" } });
```

Then in the target component:

```jsx
import { useLocation } from "react-router-dom";
const location = useLocation();
console.log(location.state); // { id: 5, from: "home" }
```

✅ **Use Case:**
Pass temporary or context-specific data between pages (e.g. form data, filters, etc.) without URL query params.

---

## 4️⃣ Backward / Forward Navigation

```jsx
navigate(-1); // Go back
navigate(1);  // Go forward
```

✅ **Explanation**

* Works like browser history’s `window.history.back()` and `forward()`.
* Keeps navigation stack intact.

---

## 5️⃣ Conditional Navigation (After Form Submit / Auth)

```jsx
if (isLoggedIn) {
  navigate("/dashboard");
} else {
  navigate("/login");
}
```

✅ **Explanation**
Used for redirects after authentication, successful form submissions, etc.

---

## 6️⃣ Replace Current Entry (Prevent Back)

```jsx
navigate("/home", { replace: true });
```

✅ **Explanation**

* Replaces current history entry instead of pushing a new one.
* User can’t go “back” to the replaced page.
* Useful for login redirects.

---

## 7️⃣ Relative Navigation

```jsx
navigate("../settings");
```

✅ **Explanation**
Navigates relative to the current route path.
For example, if you’re on `/user/profile`, navigating `../settings` takes you to `/user/settings`.

---

## 8️⃣ Delayed or Debounced Navigation

```jsx
setTimeout(() => navigate("/success"), 2000);
```

✅ **Use Case:**
Redirect after showing a “success” message or animation.

---

## 9️⃣ Using navigate in useEffect

```jsx
useEffect(() => {
  if (!user) navigate("/login");
}, [user, navigate]);
```

✅ **Explanation**

* Common pattern for protected routes or redirects.
* Safe to include `navigate` in dependencies — it’s **stable** (doesn’t change identity).

---

## 🔟 Handling Query Params with Navigation

```jsx
navigate(`/search?query=${encodeURIComponent(keyword)}`);
```

✅ **Explanation**
You can construct full URLs including query parameters.
Use `useSearchParams()` in React Router v6 to read them.

---

## 11️⃣ Scroll Restoration After Navigation

React Router v6 handles this automatically in most cases,
but you can control it manually:

```jsx
useEffect(() => {
  window.scrollTo(0, 0);
}, [location.pathname]);
```

✅ **Explanation**
Scrolls to top when route changes.

---

## 12️⃣ Programmatic Redirect After Logout

```jsx
const handleLogout = () => {
  dispatch(logout());
  navigate("/login", { replace: true });
};
```

✅ **Explanation**

* Clears session.
* Redirects to login, replacing current history entry so user can’t go “back” to the protected page.

---

## 13️⃣ Prevent Navigation Conditionally (Confirmation)

```jsx
const navigate = useNavigate();

const handleDelete = () => {
  if (window.confirm("Are you sure?")) {
    navigate("/deleted");
  }
};
```

✅ **Explanation**

* You can wrap navigation logic in conditions or prompts.
* No direct blocking API in React Router v6, but you can implement your own confirmations before navigation.

---

## 14️⃣ Navigating from Nested Routes

```jsx
<Route path="dashboard/*" element={<Dashboard />}>
  <Route path="reports" element={<Reports />} />
</Route>
```

Inside `Reports`:

```jsx
const navigate = useNavigate();
navigate("/dashboard");
```

✅ **Explanation**
Works seamlessly in nested routes — React Router auto-resolves paths correctly.

---

## 15️⃣ Common Mistakes

❌ **1. Using `useNavigate` outside Router**

```jsx
const navigate = useNavigate(); // ❌ throws error if no <BrowserRouter>
```

✅ Fix: Wrap your app in `<BrowserRouter>`.

---

❌ **2. Expecting navigate to trigger re-render of same component**

* If you navigate to the same route, React Router **won’t re-render** unless route params or keys change.
  ✅ Workaround: use `navigate("/route", { replace: true })` or append query param changes.

---

❌ **3. Using navigate() during render**

* Never call `navigate()` directly in render — only in **effects or event handlers**.

✅ Correct:

```jsx
useEffect(() => {
  if (!authUser) navigate("/login");
}, [authUser]);
```

---

## Summary Table

| #  | Scenario            | Behavior                       | Key Takeaway              |
| -- | ------------------- | ------------------------------ | ------------------------- |
| 1  | Basic navigation    | `navigate("/path")`            | Programmatic routing      |
| 2  | With params         | Template literals              | Dynamic routing           |
| 3  | With state          | `navigate("/page", { state })` | Pass non-URL data         |
| 4  | Back/Forward        | `navigate(-1)` / `navigate(1)` | History control           |
| 5  | Conditional         | Auth redirects                 | Logic-driven navigation   |
| 6  | Replace current     | `replace: true`                | Prevent back navigation   |
| 7  | Relative navigation | `../route`                     | Route-relative navigation |
| 8  | Delayed navigation  | Inside timeout                 | Post-action redirects     |
| 9  | useEffect redirect  | In dependency                  | Safe, stable              |
| 10 | Query params        | Template strings               | SEO + filter routing      |
| 11 | Scroll restore      | Manual control                 | UX enhancement            |
| 12 | Logout redirect     | Replace history                | Security pattern          |
| 13 | Confirm before nav  | Condition check                | UX safety                 |
| 14 | Nested routes       | Auto path resolve              | Easy modular routing      |
| 15 | Common mistakes     | Outside router / same path     | Must be inside Router     |

---

## Interview-Ready Answer

`useNavigate` is a React Router hook that allows programmatic navigation between routes.
It replaces `useHistory` from React Router v5.
You can navigate to paths, pass state, move forward/backward in history, and replace entries to control user navigation flow.
It’s **stable**, safe for `useEffect` dependencies, and works inside any component under a `<Router>` provider.
Common uses include **redirects after login/logout**, **conditional navigation**, and **dynamic route transitions**.

---

### 🔍 Bonus: If you meant the *browser API* (`navigator`)

You can create your own **custom `useNavigator` hook**:

```jsx
function useNavigator() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setOnline(true);
    const handleOffline = () => setOnline(false);
    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);
    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return { online, userAgent: navigator.userAgent, language: navigator.language };
}
```

✅ **Use Case:** Detect network, language, or device info — but note this is a *custom hook*, not part of React or React Router.

