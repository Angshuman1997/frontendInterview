# What are document fragments and why use them

1. Quick answer

A `DocumentFragment` is a lightweight container for DOM nodes. It’s not part of the main DOM tree; appending it to the DOM inserts its children in one operation, which is more efficient for bulk updates.

2. Example

```js
const frag = document.createDocumentFragment();
for (let i=0;i<100;i++){
  const li = document.createElement('li');
  li.textContent = `Item ${i}`;
  frag.appendChild(li);
}
document.querySelector('#list').appendChild(frag);
```
This approach reduces reflows because the fragment's children are inserted in a single operation.

3. Interview-ready summary

Use `DocumentFragment` to build DOM nodes off-DOM and then append them once to reduce layout thrashing and improve performance for large DOM insertions.
