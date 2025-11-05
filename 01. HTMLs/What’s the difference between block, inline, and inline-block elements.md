# What’s the difference between block, inline, and inline-block elements

1. Quick answer

- Block elements (`display: block`) start on a new line and take full available width (e.g., `div`, `p`).
- Inline elements (`display: inline`) flow within a line and only take the width of their content (e.g., `span`, `a`).
- Inline-block elements (`display: inline-block`) flow like inline elements but accept width/height like block elements.

2. Layout differences

- Block: accepts width/height, margin/padding stack vertically, causes line breaks.
- Inline: does not accept width/height (but padding/margin horizontally affects layout), cannot set top/bottom margins reliably.
- Inline-block: can set width/height and vertical padding/margin; preserves inline flow.

3. Use cases

- Block: layout sections (headers, footers, article containers).
- Inline: text-level semantics (links, strong, small inline decorations).
- Inline-block: buttons or small components that need a natural inline flow but also require explicit sizing.

4. Examples

```html
<div style="width:100px; height:50px; background: red;"></div>
<span style="padding: 4px;">Inline text</span>
<button style="display:inline-block; width:120px;">Click</button>
```

5. Interview-ready summary

Block elements break lines and span the container width, inline elements remain inside the line and size to content, and inline-blocks are hybrid — inline flow with block-like sizing.
