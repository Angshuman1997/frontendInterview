# What are CSS preprocessors like SASS, and why use them

1. Quick answer

CSS preprocessors (SASS/SCSS, Less, Stylus) add programming-like features to CSS — variables, nesting, mixins, functions — improving maintainability and reducing repetition.

2. Key features

- Variables: reusable values for colors, spacing
- Nesting: hierarchical selector writing (use sparingly)
- Mixins: reusable sets of declarations
- Functions and operations: color manipulation, math
- Partials and imports: modular stylesheets

3. Example (SASS)

```scss
$primary: #0070f3;

.button {
  background: $primary;
  &:hover { background: darken($primary, 8%); }
}
```

4. Why use them

- DRY styles: reduce repetition
- Better organization: modular partials and imports
- Easier maintenance: central variables for tokens
- Build-time optimizations: minification, autoprefixing

5. Caveats

- Nested selectors can cause high specificity if overused.
- Adds build complexity and a compile step.
- Modern CSS features (custom properties, calc(), clamp()) have reduced the need for some preprocessor features.

6. Interview-ready summary

Preprocessors like SASS enhance CSS with variables, nesting, mixins, and functions. They’re valuable for large projects to enforce consistency and reduce repetition but introduce a build step and should be used with care to avoid specificity and maintainability issues.
