# What are ARIA attributes and when are they used

1. Quick answer

ARIA (Accessible Rich Internet Applications) attributes add accessibility information to non-semantic or interactive elements (e.g., `role`, `aria-label`, `aria-hidden`). They should be used when native HTML semantics are insufficient.

2. Common ARIA attributes

- `role`: defines the widget type (e.g., `role="button"`).
- `aria-label` / `aria-labelledby`: accessible name for screen readers.
- `aria-hidden="true"`: hides content from assistive tech.
- `aria-expanded`, `aria-checked`, `aria-haspopup`: state attributes for interactive widgets.

3. Use only when necessary

- Prefer native HTML elements (e.g., `<button>`) because they have built-in accessibility.
- Use ARIA to fill gaps when building custom controls.

4. Examples

```html
<div role="button" tabindex="0" aria-pressed="false">Toggle</div>
```

5. Interview-ready summary

ARIA provides semantics and state to custom widgets and non-semantic elements. Prefer native elements first; use ARIA to bridge accessibility gaps with correct roles and state attributes.
