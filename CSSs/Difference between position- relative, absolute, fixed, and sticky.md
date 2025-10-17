# Difference between position: relative, absolute, fixed, and sticky

1. Quick answer

- `static`: default flow (not affected by top/right/bottom/left).
- `relative`: positioned relative to its normal position.
- `absolute`: positioned relative to the nearest positioned ancestor (`position` not `static`), removed from normal flow.
- `fixed`: positioned relative to the viewport, stays in place during scrolling.
- `sticky`: behaves like `relative` until a threshold is crossed, then behaves like `fixed` (sticky positioning within a scroll container).

2. When to use

- `relative`: minor nudges or as an anchor for absolutely positioned children.
- `absolute`: popovers, tooltips, custom positioned elements inside a container.
- `fixed`: persistent headers/footers or floating action buttons.
- `sticky`: table headers or navbars that should remain visible after scrolling past them.

3. Example

```html
<header style="position: sticky; top: 0;">Header</header>
<div style="position: relative;">
  <div style="position: absolute; right: 0;">Popover</div>
</div>
```

4. Edge cases

- `absolute` elements ignore the normal document flow; they can overlap other elements.
- `sticky` requires a scrolling container; it won’t work if the container has `overflow: hidden` that clips it.

5. Interview-ready summary

Use `relative` for small offsets and anchors, `absolute` for positioned children removed from flow, `fixed` for viewport-anchored elements, and `sticky` for elements that should stick after scrolling within a container.
