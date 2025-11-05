# What’s the difference between localStorage, sessionStorage, and cookies

1. Quick answer

- `localStorage`: persisted key/value storage scoped to the origin, survives browser restarts until explicitly cleared.
- `sessionStorage`: similar to localStorage but scoped to the browser tab (cleared when the tab/window is closed).
- Cookies: small pieces of data sent with every HTTP request (unless `HttpOnly`), used for server-side sessions, with configurable expiration and flags (Secure, SameSite).

2. Capacity and performance

- localStorage/sessionStorage: typically ~5-10MB per origin depending on the browser.
- Cookies: around 4KB per cookie, and many cookies increase request size on each HTTP request.

3. Accessibility and security

- localStorage/sessionStorage: accessible from JavaScript (not sent to the server automatically) — vulnerable to XSS if data is sensitive.
- Cookies: if marked `HttpOnly`, not accessible from JS, helping mitigate XSS; `Secure` restricts to HTTPS; `SameSite` helps mitigate CSRF.

4. Use cases

- localStorage: client-side persistence for non-sensitive data (UI state, preferences).
- sessionStorage: data that should live only for a tab session (wizard state, temporary UI state).
- cookies: authentication/session tokens (with HttpOnly and Secure flags), server-driven state.

5. Example code

```js
// localStorage
localStorage.setItem('theme', 'dark');
// sessionStorage
sessionStorage.setItem('draft', JSON.stringify(formDraft));
// cookie (simple)
document.cookie = 'name=value; path=/; max-age=3600; Secure; SameSite=Lax';
```

6. Interview-ready summary

Use localStorage/sessionStorage for client-only data and not for secrets. Use cookies (with HttpOnly and Secure) for server-side sessions or authentication tokens. Keep cookies small to reduce request overhead.
