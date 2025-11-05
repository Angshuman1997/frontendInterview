# Virtual DOM & Reconciliation algorithm

The Virtual DOM is a lightweight, in-memory representation of the real DOM. It’s a plain JavaScript object that React uses to describe what the UI should look like. Instead of updating the real DOM directly on every change, React builds a Virtual DOM tree, compares it with the previous one (diffing), and updates only the changed parts in the real DOM. This process makes React fast and efficient.

Why not update the real DOM directly:
The real DOM is slow to manipulate. Every change triggers layout recalculations and reflows, which are expensive. The Virtual DOM minimizes direct DOM manipulations by batching and optimizing updates.

How it works:

1. React renders the UI to a Virtual DOM tree.
2. When state or props change, React creates a new Virtual DOM tree.
3. React compares the new tree with the old one using the reconciliation (diffing) algorithm.
4. Only the differences (minimal changes) are updated in the real DOM.

Example:
Old Virtual DOM: <h1>Hello</h1>
New Virtual DOM: <h1>Hi</h1>
React updates only the text, not the entire element.

Reconciliation Algorithm (Diffing):
React’s reconciliation algorithm determines how to efficiently update the UI.

* If element types differ (e.g., <div> → <span>), React destroys and recreates the node.
* If element types are the same, React updates attributes and children.
* React uses the “key” prop in lists to identify which elements have changed, been added, or removed.

Simplified rules:

1. Different element types → replace the node.
2. Same type → update attributes and diff children.
3. Keys → identify stable elements in lists.

Pseudocode:
function reconcile(oldNode, newNode) {
if (oldNode.type !== newNode.type) {
replaceNode(oldNode, newNode);
} else {
updateProps(oldNode, newNode);
reconcileChildren(oldNode.children, newNode.children);
}
}

React Fiber (modern reconciliation engine):
React 16 introduced Fiber, a new reconciliation algorithm that enables:

* Incremental rendering (splitting work into chunks)
* Pausing, resuming, and aborting rendering tasks
* Prioritizing updates (e.g., animations before background tasks)
* Concurrent rendering for smoother performance

Performance benefits:

* React avoids unnecessary DOM operations.
* Batch updates for efficiency.
* Maintains UI responsiveness during heavy updates.

Interview-ready answer:
The Virtual DOM is an in-memory representation of the UI. When state or props change, React creates a new Virtual DOM tree and compares it with the previous one using the reconciliation (diffing) algorithm. React updates only the changed parts in the real DOM, improving performance. The reconciliation process uses rules like comparing element types and keys to determine minimal updates. React Fiber enhances this process by making it incremental and prioritized for smoother performance.
