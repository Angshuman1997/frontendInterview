# What is the DOM, and how does it differ from the Virtual DOM

1. Quick answer

The DOM is the browser’s tree representation of HTML documents; the Virtual DOM (VDOM) is an in-memory representation used by libraries (like React) to compute minimal updates to the real DOM efficiently.

2. Differences

- DOM: actual browser API, expensive to manipulate because changes can trigger layout/paint.
- VDOM: lightweight JS object tree; diffing two VDOM trees produces a patch which is applied to the real DOM to minimize updates.

3. Why use a Virtual DOM

- Batch updates and compute minimal DOM mutations.
- Easier to reason about UI state as a function of data.

4. Trade-offs

- VDOM adds CPU overhead for diffing, but saves expensive layout/paint by reducing DOM writes.
- For simple UIs or highly optimized direct DOM code, VDOM might not always be needed.

5. Interview-ready summary

The DOM is the browser’s document model; a Virtual DOM is a library-managed in-memory representation that helps reduce and batch real DOM updates by diffing two virtual trees and applying minimal changes.
