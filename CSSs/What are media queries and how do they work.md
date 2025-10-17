# What are media queries and how do they work

1. Quick answer

Media queries are CSS rules that apply styles conditionally based on the viewport or device features (e.g., width, resolution, orientation). They enable responsive design.

2. Syntax examples

- Basic:
```css
@media (max-width: 768px) {
  .sidebar { display: none; }
}
```
- Using logical operators:
```css
@media (min-width: 480px) and (max-width: 1024px) { }
```

3. Common features

- width, height, orientation, resolution, aspect-ratio
- prefers-reduced-motion, prefers-color-scheme (dark/light mode)

4. Mobile-first vs desktop-first

- Mobile-first: define base (mobile) styles, use `min-width` media queries to scale up.
- Desktop-first: define base (desktop) styles, use `max-width` queries for smaller screens.

5. Performance tips

- Keep queries simple and limited in number; complex queries can lead to unnecessary repaints.
- Combine queries and use well-structured breakpoints.
- Use responsive images (`srcset`, `picture`) to reduce bandwidth.

6. Practical example

```css
/* mobile-first */
.card { padding: 12px; }
@media (min-width: 768px) {
  .card { padding: 20px; }
}
```

7. Interview-ready summary

Media queries let you apply CSS conditionally based on device or viewport properties. Prefer mobile-first breakpoints with `min-width`, keep queries simple, and test across devices.
