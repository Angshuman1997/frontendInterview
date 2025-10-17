# What is the Shadow DOM and how does it help with encapsulation

1. Quick answer

The Shadow DOM is a browser feature that enables DOM and style encapsulation for components: it creates a scoped subtree (shadow root) whose internal DOM and CSS are isolated from the main document.

2. Why it matters

- Encapsulation: prevents styles and IDs from leaking in or out of a component.
- Predictability: components behave the same regardless of page CSS.
- Reusability: safe to ship self-contained UI widgets.

3. How it works (basic example)

```js
const host = document.createElement('div');
const shadow = host.attachShadow({ mode: 'open' });
shadow.innerHTML = `
  <style>p{ color: red; }</style>
  <p>Shadow content</p>
`;
document.body.appendChild(host);
```
The `<p>` inside the shadow root is styled by the shadow's style, not affected by outer page CSS.

4. Styling and composition

- Styles in a shadow root don't cascade out; page styles don't cascade in (except CSS custom properties and ::slotted rules).
- `::slotted` lets you style light DOM children projected into a slot.

5. Limitations and trade-offs

- Accessibility: ensure correct ARIA roles and keyboard handling when encapsulating content.
- Performance: creating many shadow roots can have a small cost; use when encapsulation justifies it.
- Browser support is modern and stable in evergreen browsers.

6. Use cases

- Web Components, custom elements, complex third-party widgets, design systems where style isolation is required.

7. Interview-ready summary

Shadow DOM gives you component-local DOM and CSS isolation so that component styles don't leak and global styles don't unintentionally affect a component. It’s a core feature for building robust, reusable web components.
