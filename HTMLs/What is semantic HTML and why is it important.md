# What is semantic HTML and why is it important

1. Quick answer

Semantic HTML uses elements that reflect their meaning (`<header>`, `<main>`, `<article>`, `<nav>`) instead of generic containers (`div`, `span`). It improves accessibility, SEO, and maintainability.

2. Benefits

- Accessibility: screen readers and assistive tech can navigate content better.
- SEO: search engines better understand page structure.
- Readability: clearer structure for developers.
- Default browser behavior: semantic elements often have built-in roles and behaviors.

3. Example

Bad:
```html
<div id="header">...</div>
```
Good:
```html
<header>...</header>
```

4. ARIA and semantics

- Use ARIA when native semantics are insufficient, but prefer built-in HTML semantics first.
- Example: if building a custom widget, use `role` and ARIA attributes to make it accessible.

5. Interview-ready summary

Use semantic HTML to make content meaningful to users, assistive tech, and search engines. Prefer native elements and only add ARIA when necessary.
