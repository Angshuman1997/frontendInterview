# Why are keys needed in lists

Keys are unique identifiers used by React to track elements in a list. They help React efficiently update and re-render only the items that have changed, been added, or removed, rather than re-rendering the entire list.

When rendering lists using the map() function, each element should have a key prop that uniquely identifies it among its siblings.

Example:
{items.map(item => (

  <li key={item.id}>{item.name}</li>
))}

How React uses keys:

1. React builds a Virtual DOM tree for the list items.
2. When the list changes, React compares the new Virtual DOM with the previous one.
3. Using the keys, React determines which items have changed, been removed, or added.
4. React updates only those specific elements in the real DOM.

Without unique keys:

* React may incorrectly match elements between renders.
* Unnecessary re-renders can happen.
* Component state can be lost or mixed up between list items.

Example without proper keys:
If you use the index as a key (key={index}) and items are reordered, React can mistakenly reuse DOM nodes for different data, causing visual or logical bugs.

Correct usage:

* Use a stable, unique value like an ID.
* Avoid using array indices unless the list is static and never changes order.

Example:
✅ Correct:
{users.map(user => <UserCard key={user.id} data={user} />)}

❌ Incorrect:
{users.map((user, index) => <UserCard key={index} data={user} />)}

Interview-ready answer:
Keys are unique identifiers that help React efficiently identify which list items have changed, been added, or removed during reconciliation. They allow React to update only the affected elements in the real DOM instead of re-rendering the entire list. Without proper keys, React can mismanage element updates, causing performance issues or bugs. Always use stable, unique keys like IDs rather than array indices.
