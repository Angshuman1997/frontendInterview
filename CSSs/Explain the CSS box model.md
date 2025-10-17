# Explain the CSS box model

1. Quick answer

Every element is a rectangular box composed of (from inside out): content → padding → border → margin. Understanding this is crucial for layout and spacing.

2. Components

- Content: the element's content area (text, images)
- Padding: space between content and border
- Border: the element's edge
- Margin: outer spacing between elements

3. Box-sizing

- `content-box` (default): width/height apply to content area; padding and border add on top.
- `border-box`: width/height include padding and border — often easier for predictable layouts.

4. Example

```css
.box { box-sizing: border-box; width: 200px; padding: 16px; border: 2px solid #000; }
```
- With `border-box`, the total rendered width stays 200px; with `content-box` it would be 200 + padding + border.

5. Collapsing margins

- Vertical margins between block elements can collapse under certain conditions — the larger margin wins instead of adding.

6. Interview-ready summary

The CSS box model consists of content, padding, border, and margin. Use `box-sizing: border-box` for more intuitive sizing and be mindful of margin collapsing when spacing block-level elements.
