# What is the difference between inline, internal, and external CSS

1. Quick answer

Inline CSS: styles applied directly on an element via the `style` attribute (e.g., `<div style="color: red">`).

Internal CSS: styles defined inside a `<style>` tag in the HTML document (usually within the `<head>`).

External CSS: styles defined in separate `.css` files and linked via `<link rel="stylesheet" href="styles.css">`.

2. When to use each

- Inline: small, one-off overrides or dynamically set styles from JavaScript. Use sparingly — hurts maintainability and specificity management.
- Internal: useful for single-page demos, email templates, or when you need page-specific styles without creating a file. Not ideal for large sites.
- External: preferred for production sites — promotes reuse, caching, separation of concerns, and easier maintenance.

3. Performance and caching

- External CSS benefits from browser caching: the file downloads once and is reused across pages.
- Internal and inline CSS are part of the HTML payload, increasing download size and preventing shared caching.

4. Specificity and overrides

- Inline styles have the highest specificity (except `!important` rules) and will override internal/external rules.
- Internal styles are evaluated after external styles if placed later in the document; they can override external definitions.

5. Maintainability and scalability

- External CSS scales best for teams and large apps: files can be modularized, processed by build tools (SASS, PostCSS), and linted.
- Inline CSS quickly becomes unmanageable for complex UIs and is hostile to style reuse.

6. Accessibility and progressive enhancement

- Inline styles mix presentation with markup and make it harder to provide fallbacks for users with different needs.
- External CSS enables progressive enhancement and easier conditions for different media queries or print styles.

7. Example

- Inline:
	`<button style="background: blue; color: white">Click</button>`

- Internal:
	`<head><style>button{ background: blue; color: white; }</style></head>`

- External:
	`<!-- head --> <link rel="stylesheet" href="/styles/button.css">`

8. Interview-ready summary

Use external CSS for real projects for caching, maintainability, and tooling support. Internal CSS is acceptable for small, single-page scenarios or component-scoped quick styles. Inline CSS should be reserved for tiny one-off overrides or dynamically injected styles where other options are not practical.

9. Pro tips

- Prefer CSS classes and external stylesheets. Avoid using inline styles for layout, animations, or large style sets.
- When using component libraries (React), prefer CSS Modules, CSS-in-JS (styled-components, Emotion), or utility-first frameworks (Tailwind) over inline styles.

10. Edge cases

- Email clients often require inline CSS for compatibility, so tooling that inlines styles (e.g., juice) is common in email workflows.
- Critical CSS: inline just the minimal CSS needed to render above-the-fold content, while deferring the rest in external files for faster LCP.

11. References and further reading

- MDN: Cascade and Specificity
- Google Web Fundamentals: Critical rendering path and critical CSS
