# What are reflow and repaint in browsers

1. Quick answer

- Reflow (layout): browser recalculates positions and sizes of elements when the document structure or geometry changes.
- Repaint: browser redraws pixels for elements whose appearance changed but geometry did not.

2. Causes

- Reflow: changing width/height, adding/removing DOM nodes, changing fonts, or CSS that affects layout.
- Repaint: changing color, visibility, background, but not layout.

3. Performance tips

- Minimize layout thrashing: read layout values (offsetWidth) and write styles in batches.
- Use `transform` and `opacity` for animations to avoid reflow.
- Use requestAnimationFrame for animation timing.

4. Interview-ready summary

Reflow recalculates layout and is expensive; repaint redraws pixels. Minimize reflows by batching DOM reads/writes and using transform/opacity for animations.
