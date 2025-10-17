# What is specificity, and how is it calculated

1. Quick answer

Specificity is a scoring system used by browsers to determine which CSS rule applies when multiple rules target the same element. Higher specificity wins.

2. Specificity components (score breakdown)

- Inline styles: a = 1 (highest; e.g., `style="..."`)
- IDs: b = number of ID selectors in the selector (e.g., `#header`)
- Classes, attributes, pseudo-classes: c = number of those selectors (e.g., `.btn`, `[type="text"]`, `:hover`)
- Elements and pseudo-elements: d = number of element selectors and pseudo-elements (e.g., `div`, `::before`)

Specificity is compared lexicographically: (a, b, c, d).

3. Examples

- `#nav .item a` -> (0,1,1,1)
- `.btn.primary:hover` -> (0,0,3,0)
- `div` -> (0,0,0,1)
- `style="color:red"` -> (1,0,0,0)

4. Important vs specificity

The `!important` keyword overrides normal specificity rules and makes a declaration the highest priority. Between multiple `!important` rules, specificity and order still decide.

5. Common gotchas

- Overly relying on IDs for styling makes components hard to reuse and override.
- Deep selector chains increase specificity and reduce maintainability; prefer class-based composition.
- Reset or normalize styles can be overridden by more specific rules later in the cascade.

6. Best practices

- Prefer low specificity selectors (class-based) for component styles.
- Use utility classes or CSS-in-JS where appropriate to contain styles.
- Use `!important` only as a last resort; prefer adjusting specificity or refactoring.

7. Tools and debugging

- Browser devtools show which rule applied and the selector’s specificity.
- Linters like stylelint can warn about high-specificity selectors.

8. Interview-ready summary

Specificity is how browsers decide which CSS rule wins when multiple rules could apply. It’s calculated as a four-part score (inline, IDs, classes/attrs/pseudo-classes, elements/pseudo-elements) and compared lexicographically. Keep selectors low-specificity and class-based for better maintainability.
