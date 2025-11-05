# What are pseudo-classes and pseudo-elements

1. Quick answer

Pseudo-classes represent states of elements (e.g., `:hover`, `:focus`). Pseudo-elements represent abstractions of elements or parts of elements (e.g., `::before`, `::after`).

2. Pseudo-classes (examples)

- `:hover`, `:active`, `:focus`
- `:nth-child(n)`, `:first-child`, `:last-child`
- `:not(selector)`

3. Pseudo-elements (examples)

- `::before`, `::after` — insert generated content
- `::first-line`, `::first-letter`
- `::marker` — list markers

4. Differences

- Pseudo-classes match existing elements in specific states.
- Pseudo-elements create or target portions of an element’s content.
- Syntax: modern convention uses `::` for pseudo-elements and `:` for pseudo-classes; single `:` still supported for historical pseudo-elements.

5. Browser support and performance

- Most pseudo-classes/elements are well supported, though newer ones like `::marker` may need checks.
- Using `:nth-child` and complex selectors can impact rendering performance if overused on large DOMs.

6. Practical examples

- Add an icon before text:
```css
.button::before { content: "🔍"; margin-right: 0.5rem; }
```
- Style alternate rows:
```css
.table tr:nth-child(even) { background: #f7f7f7; }
```

7. Interview-ready summary

Pseudo-classes and pseudo-elements let you style element states and parts of elements without extra markup. Use them to keep HTML clean and to implement UI effects declaratively in CSS.
