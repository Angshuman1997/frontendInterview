# Difference between Flexbox and CSS Grid

1. Quick answer

Flexbox is a one-dimensional layout model (row OR column). Grid is a two-dimensional layout model (rows AND columns). Use Flexbox for linear layouts and Grid for complex, two-axis layouts.

2. When to use

- Flexbox: centering, small components, navs, list layouts, distribution of space along one axis.
- Grid: overall page layouts, complex component layouts, overlapping elements, masonry-like layouts (with grid-auto-rows tricks).

3. Capabilities

- Flexbox handles flexible sizing and ordering along a single axis.
- Grid handles explicit placement with grid lines, area templates, and fine-grained control over both axes.

4. Example

Flexbox use-case:
```css
.header { display: flex; align-items: center; justify-content: space-between; }
```
Grid use-case:
```css
.grid { display: grid; grid-template-columns: 1fr 2fr; grid-template-rows: auto 1fr; grid-gap: 16px; }
```

5. Combining

- Use Grid for the overall page and Flexbox for small components/rows inside grid items.

6. Interview-ready summary

Flexbox is best for 1D layout (aligning items along a row or column); Grid is best for 2D layout (complex rows and columns). Combine both: Grid for the page, Flexbox for components.
