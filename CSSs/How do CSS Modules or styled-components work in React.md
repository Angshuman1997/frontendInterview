# How do CSS Modules or styled-components work in React

1. Quick answer

CSS Modules: scope CSS locally by generating unique class names during build, avoiding collisions. Styled-components: CSS-in-JS library that generates scoped styles at runtime/build time using tagged template literals.

2. CSS Modules

- File: `Button.module.css`
- Import: `import styles from './Button.module.css'`
- Usage: `<button className={styles.primary}>` — class names are hashed at build time.
- Works well with bundlers (Webpack, Vite) and supports local scoping while keeping regular CSS tooling.

3. styled-components

- Usage:
```js
import styled from 'styled-components';
const Button = styled.button`
  background: ${props => props.primary ? 'blue' : 'gray'};
`;
```
- Supports theming, dynamic props, and automatic critical CSS extraction in server-side rendering.

4. Trade-offs

- CSS Modules: compile-time, works with existing CSS tools and PostCSS. No runtime cost.
- styled-components: runtime styling (though SSR optimizes it), better dynamic styling ergonomics, but larger bundle size.

5. When to use which

- CSS Modules: when you want a CSS-first approach with local scoping and no runtime library.
- styled-components: when dynamic styles, theming, or co-locating styles with components is a priority.

6. Interview-ready summary

Both CSS Modules and styled-components solve CSS scoping issues in React. CSS Modules generate hashed class names at build time for local scoping; styled-components provides a CSS-in-JS API for dynamic, component-scoped styles and theming. Choose based on whether you prefer compile-time CSS tooling (Modules) or runtime dynamic styling and theming (styled-components).
