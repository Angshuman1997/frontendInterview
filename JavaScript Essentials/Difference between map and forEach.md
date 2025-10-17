# Difference between map and forEach

1. Quick answer

- `map` returns a new array built from the return value of the callback.
- `forEach` executes a callback for each item but returns `undefined` (used for side effects).

2. Example

```js
const arr = [1,2,3];
const doubled = arr.map(x => x * 2); // [2,4,6]
arr.forEach(x => console.log(x)); // undefined
```

3. When to use

- Use `map` when you want to transform an array and get a new array.
- Use `forEach` for side effects (logging, mutating external state).

4. Interview-ready summary

`map` is for transformations that produce a new array; `forEach` is for iteration with side effects and no return value.
