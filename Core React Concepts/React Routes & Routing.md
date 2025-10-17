# React Routes & Routing

## Overview

**React Router** is a powerful client-side routing library for React that enables single-page applications (SPAs) to simulate multi-page behavior **without full page reloads**.
It manages navigation, dynamic URLs, nested views, and conditional rendering based on the browser path — giving users a smooth, app-like experience.

Modern **React Router v6+** introduced:

* Simplified syntax with `<Routes>` and `<Route>`
* Hooks (`useNavigate`, `useParams`, `useLocation`)
* Nested and dynamic routing
* Relative paths and layouts

---

## 1️⃣ Basic Setup

```jsx
import { BrowserRouter, Routes, Route } from "react-router-dom";
import Home from "./Home";
import About from "./About";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
}
```

✅ **Explanation**

* `BrowserRouter`: Wraps the app and enables routing via the HTML5 history API.
* `Routes`: Defines route structure (replaces old `Switch`).
* `Route`: Maps a URL path to a React component.

---

## 2️⃣ Navigation — `Link` and `useNavigate`

```jsx
import { Link, useNavigate } from "react-router-dom";

function Navbar() {
  const navigate = useNavigate();

  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/about">About</Link>
      <button onClick={() => navigate("/contact")}>Go to Contact</button>
    </nav>
  );
}
```

✅ **Explanation**

* `Link`: Declarative navigation without reloading.
* `useNavigate`: Programmatic navigation — useful after form submissions or login redirects.

---

## 3️⃣ Dynamic Routes (URL Parameters)

```jsx
<Route path="/user/:id" element={<UserProfile />} />

function UserProfile() {
  const { id } = useParams();
  return <h3>User ID: {id}</h3>;
}
```

✅ **Explanation**

* `:id` defines a **dynamic parameter**.
* `useParams()` gives access to route variables.
* URL `/user/101` → renders with `id = 101`.

---

## 4️⃣ Nested Routes (Layouts & Subpages)

```jsx
<Route path="/dashboard" element={<Dashboard />}>
  <Route path="profile" element={<Profile />} />
  <Route path="settings" element={<Settings />} />
</Route>
```

✅ **Explanation**

* Nested routes share a **common layout** (`Dashboard` here).
* Inside `Dashboard`, use `<Outlet />` to render child routes dynamically.

```jsx
function Dashboard() {
  return (
    <div>
      <h2>Dashboard</h2>
      <Outlet /> {/* Nested route renders here */}
    </div>
  );
}
```

---

## 5️⃣ Default and Catch-All Routes

```jsx
<Route index element={<Home />} />             // Default route inside parent
<Route path="*" element={<NotFound />} />     // Catch-all (404 page)
```

✅ **Explanation**

* `index` route → acts as default for nested structure.
* `"*"` route → catches unmatched URLs.

---

## 6️⃣ Route Redirection

```jsx
import { Navigate } from "react-router-dom";

<Route path="/" element={<Navigate to="/home" />} />
```

✅ **Explanation**

* Redirects automatically from `/` → `/home`.
* Commonly used for **legacy paths** or **conditional routing**.

---

## 7️⃣ Protected Routes (Authentication Example)

```jsx
function ProtectedRoute({ children }) {
  const isAuth = localStorage.getItem("token");
  return isAuth ? children : <Navigate to="/login" replace />;
}

<Route path="/dashboard" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
```

✅ **Explanation**

* Restricts access to authenticated users.
* Redirects unauthenticated users to `/login`.

---

## 8️⃣ useLocation (Get Current Path & State)

```jsx
import { useLocation } from "react-router-dom";

function CurrentPage() {
  const location = useLocation();
  return <p>Current path: {location.pathname}</p>;
}
```

✅ **Explanation**

* `useLocation()` gives current URL info and route state.
* Useful for analytics, navigation breadcrumbs, and conditional UI rendering.

---

## 9️⃣ useSearchParams (Query Parameters)

```jsx
import { useSearchParams } from "react-router-dom";

function SearchPage() {
  const [params] = useSearchParams();
  const query = params.get("q");
  return <p>Searching for: {query}</p>;
}
```

✅ **URL:** `/search?q=react`
✅ **Output:** `Searching for: react`

✅ **Explanation**

* Simplifies reading/updating URL query params.
* Ideal for filters, pagination, or search pages.

---

## 🔟 Lazy Loading Routes (Code Splitting)

```jsx
import { lazy, Suspense } from "react";

const About = lazy(() => import("./About"));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<p>Loading...</p>}>
        <Routes>
          <Route path="/about" element={<About />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

✅ **Explanation**

* Dynamically imports route components only when needed.
* Improves performance via **code-splitting**.

---

## 11️⃣ Relative vs Absolute Paths

```jsx
<Route path="dashboard">
  <Route path="stats" element={<Stats />} />     // Relative path → /dashboard/stats
  <Route path="/help" element={<Help />} />      // Absolute path → /help
</Route>
```

✅ **Explanation**

* **Relative paths** nest under the parent route.
* **Absolute paths** ignore nesting and start from root.

---

## 12️⃣ Scroll to Top on Route Change

```jsx
function ScrollToTop() {
  const { pathname } = useLocation();
  useEffect(() => window.scrollTo(0, 0), [pathname]);
  return null;
}
```

✅ **Explanation**
Ensures each route change scrolls to the top of the page — improves navigation experience.

---

## 13️⃣ Route-Level Code Organization

Organize routes in a separate file for maintainability:

```jsx
// routes.js
export const routes = [
  { path: "/", element: <Home /> },
  { path: "/about", element: <About /> },
  { path: "/login", element: <Login /> },
];

// App.js
<Routes>
  {routes.map((r) => (
    <Route key={r.path} path={r.path} element={r.element} />
  ))}
</Routes>
```

✅ **Explanation**
Keeps routing configuration clean and scalable.

---

## 14️⃣ BrowserRouter vs HashRouter vs MemoryRouter

| Router Type       | Use Case               | Behavior                           |
| ----------------- | ---------------------- | ---------------------------------- |
| **BrowserRouter** | Web apps (default)     | Uses history API (`/path`)         |
| **HashRouter**    | Static file hosting    | Uses `/#/path` URLs                |
| **MemoryRouter**  | Testing / React Native | Keeps route history in memory only |

✅ **Explanation**
Choose based on environment — `BrowserRouter` for SPAs, `HashRouter` for static pages.

---

## 15️⃣ Common Mistakes

❌ **1. Forgetting BrowserRouter**

```jsx
<Routes> ... </Routes> // ❌ Error: useNavigate() may be used only in context of Router
```

✅ Fix:

```jsx
<BrowserRouter>
  <Routes> ... </Routes>
</BrowserRouter>
```

---

❌ **2. Using `component=` instead of `element=`**
Old React Router v5 syntax:

```jsx
<Route path="/" component={Home} /> // ❌ Invalid in v6
```

✅ New syntax:

```jsx
<Route path="/" element={<Home />} />
```

---

❌ **3. Calling hooks outside Router scope**
Hooks like `useNavigate`, `useParams`, `useLocation` only work **inside `<Router>`**.

---

## Summary Table

| #  | Feature          | Hook/Component                           | Description            | Example                |
| -- | ---------------- | ---------------------------------------- | ---------------------- | ---------------------- |
| 1  | Basic Routing    | `<BrowserRouter>`, `<Routes>`, `<Route>` | Define routes          | `/about` → `<About />` |
| 2  | Navigation       | `Link`, `useNavigate`                    | Move between pages     | `navigate("/home")`    |
| 3  | Dynamic Params   | `useParams`                              | Access URL variables   | `/user/:id`            |
| 4  | Nested Routes    | `<Outlet>`                               | Layout with sub-routes | `/dashboard/profile`   |
| 5  | Redirects        | `<Navigate>`                             | Auto redirect          | `/` → `/home`          |
| 6  | Protected Routes | Custom wrapper                           | Auth-based routing     | `<ProtectedRoute>`     |
| 7  | Current Path     | `useLocation`                            | Get route path & state | `location.pathname`    |
| 8  | Query Params     | `useSearchParams`                        | URL params             | `?q=term`              |
| 9  | Lazy Loading     | `React.lazy`                             | Load components async  | Performance boost      |
| 10 | Scroll Restore   | `useEffect`                              | Reset scroll           | Smooth UX              |

---

## Interview-Ready Answer

React Router enables navigation and routing in React single-page applications without reloading the browser.
It uses the **history API** to map URLs to components.
With `<BrowserRouter>`, `<Routes>`, and `<Route>`, you can define multiple pages, while hooks like `useNavigate`, `useParams`, and `useLocation` give full control over dynamic navigation, path parameters, and query strings.

Modern versions support **nested routes**, **lazy loading**, and **protected routes**, making routing both declarative and powerful for large-scale apps.

---

✅ **Key Takeaway:**

> React Router lets you build dynamic, nested, and interactive navigation in SPAs using declarative routes and hooks — with no full page reloads.
