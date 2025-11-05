# How does React handle re-rendering

React handles re-rendering through its Virtual DOM and reconciliation process. When a component's state or props change, React re-renders that component to generate a new Virtual DOM tree. It then compares the new Virtual DOM with the previous one to determine what actually changed and updates only those specific parts in the real DOM.

This selective updating process ensures that React applications stay performant even as the UI grows large and complex.

Detailed process:

1. **State or props change:**
   React triggers a re-render of the affected component.

2. **Render phase:**
   The component’s render function (or return in a functional component) is called to produce a new Virtual DOM representation.

3. **Diffing (Reconciliation):**
   React compares the new Virtual DOM tree with the previous one to find differences (added, removed, or updated nodes).

4. **Commit phase:**
   React updates only the changed nodes in the real DOM. The rest of the DOM remains untouched.

5. **Batching updates:**
   React groups multiple state updates into a single render cycle to avoid unnecessary renders and improve performance.

Example:
function Counter() {
const [count, setCount] = useState(0);

return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

When the button is clicked:

* setCount triggers a re-render.
* React re-runs the Counter component.
* A new Virtual DOM is created.
* React compares it with the previous version.
* Only the text content of the button is updated in the real DOM.

Performance optimizations during re-rendering:

* React.memo() prevents re-renders of child components when props haven’t changed.
* useMemo() and useCallback() memoize values and functions to avoid unnecessary recalculations.
* React batches multiple state updates together to minimize rendering work.

Interview-ready answer:
React handles re-rendering by re-creating a Virtual DOM when state or props change, comparing it with the previous Virtual DOM, and updating only the parts of the real DOM that have changed. This process is called reconciliation. React also batches updates and uses memoization techniques to avoid unnecessary re-renders, ensuring efficient and fast UI updates.
