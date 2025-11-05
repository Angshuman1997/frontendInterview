# How does React’s reconciliation differ from direct DOM manipulation

1. Quick answer

Reconciliation is React’s strategy for updating the UI: it creates a new VDOM, diffs it against the previous VDOM, and applies a minimal set of mutations to the real DOM. Direct DOM manipulation updates nodes immediately, which can cause extra reflows/paints.

2. Key differences

- React batches and diffs updates to minimize DOM writes.
- Direct DOM updates are immediate and can trigger layout repeatedly if not batched.

3. Best practices

- Use keys for list items to help reconciliation.
- Avoid direct DOM manipulation except when necessary (e.g., integrating a non-React library).

4. Interview-ready summary

React’s reconciliation diffs virtual representations and applies minimal DOM patches, whereas direct DOM manipulation performs immediate changes that can be more expensive and error-prone when managing complex UI state.
