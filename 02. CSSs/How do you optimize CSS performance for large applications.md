# How do you optimize CSS performance for large applications

1. Quick answer

Optimize CSS by minimizing critical CSS, leveraging caching, reducing unused styles, and keeping selectors simple to minimize render cost.

2. Strategies

- Critical CSS: inline above-the-fold styles to speed first render, defer the rest.
- Code-splitting CSS: load page-specific CSS only when needed.
- Minification & compression: reduce payload size (CSSNano, CleanCSS).
- Remove unused CSS: PurgeCSS, UnCSS, or bundler integrations to drop dead styles.
- Use efficient selectors: avoid deep descendant selectors and `:not` with complex patterns.
- Use CSS containment and will-change sparingly for heavy animations.
- Use modern layout (Flexbox/Grid) to avoid layout thrashing.

3. Tooling and automation

- PurgeCSS: remove unused classes for frameworks like Tailwind.
- PostCSS: autoprefixer, minification, polyfills.
- Use HTTP/2 or HTTP/3 for parallel resource fetching.

4. Caching and delivery

- Use long cache headers for versioned CSS files and content hashing.
- Serve critical CSS inline and lazy-load the rest via rel=preload or dynamic import.

5. Example: critical CSS + defer

```html
<style><!-- minimal critical styles here --></style>
<link rel="preload" href="styles.css" as="style" onload="this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="styles.css"></noscript>
```

6. Interview-ready summary

Optimize CSS by extracting critical CSS, minimizing and purging unused styles, using modular CSS delivery (code-splitting), and keeping selectors efficient. Combine these with caching and modern delivery techniques for best results.
