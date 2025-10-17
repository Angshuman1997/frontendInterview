# Shallow vs deep copy

1. Quick answer

Shallow copy duplicates the top-level properties of an object but keeps nested objects by reference. Deep copy duplicates nested structures recursively so changes don't affect the original.

2. Examples

- Shallow copy using spread:
```js
const a = { x: 1, nested: { y: 2 } };
const b = { ...a };
b.nested.y = 3; // a.nested.y === 3 (shared reference)
```
- Deep copy using structuredClone (modern):
```js
const deep = structuredClone(a);
```
- Deep copy using JSON (limitations with functions, Dates, undefined):
```js
const deep = JSON.parse(JSON.stringify(a));
```

3. Libraries and advanced options

- Lodash `_.cloneDeep` for robust deep cloning.
- `structuredClone` supports many built-ins but check browser compatibility.

4. Interview-ready summary

Use shallow copies for simple, shallow structures and deep clone for nested objects when immutability is required. Prefer `structuredClone` or `_.cloneDeep` for complex types.
