# How do you create a responsive design

1. Quick answer

Responsive design adapts the layout and UI to various screen sizes using fluid layouts, flexible images, and media queries.

2. Building blocks

- Fluid grids (percentage-based widths or CSS Grid with fractional units)
- Flexible images (`max-width: 100%; height: auto;`) and srcset
- Media queries for breakpoints
- Relative units: rem, em, vh, vw, %
- Modern functions: clamp(), min(), max()

3. Approach and strategy

- Design mobile-first: start with baseline mobile styles and progressively enhance using `min-width` breakpoints.
- Define a limited number of meaningful breakpoints based on content needs.
- Prefer layout that adapts (Grid/Flexbox) over hiding and displaying too many variants.

4. Example

```css
.card { padding: 12px; }
@media (min-width: 768px) {
  .card { padding: 20px; }
}
```

5. Images and assets

- Use `srcset`/`sizes` and `picture` to deliver the right image resolution.
- Use vector formats (SVG) for icons where possible.

6. Accessibility and touch targets

- Ensure touch targets are large enough (minimum 44x44px recommended).
- Maintain keyboard accessibility and focus order after layout changes.

7. Interview-ready summary

Create responsive designs using fluid layouts, flexible images, relative units, media queries, and a mobile-first approach. Prefer layout primitives (Grid/Flexbox) and use responsive images and modern CSS functions for robust, maintainable UIs.
