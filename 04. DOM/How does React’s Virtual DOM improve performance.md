# How does React’s Virtual DOM improve performance

1. Quick answer

React’s Virtual DOM lets the library compute differences between render outputs and apply only necessary DOM mutations, reducing expensive layout/paint operations.

2. How it helps

- Reconciliation: React diffs old and new VDOM and generates minimal patches.
- Batching: React batches updates and applies them efficiently.
- Componentization: isolated updates at component boundaries reduce unnecessary changes.

3. Examples

- Using keys with lists helps React avoid re-rendering unchanged items.
- `shouldComponentUpdate`/`React.memo` can avoid unnecessary re-render computations.

4. Interview-ready summary

React’s VDOM minimizes direct DOM writes by diffing in-memory representations and applying minimal updates, thus lowering the number of expensive layout and paint operations.
