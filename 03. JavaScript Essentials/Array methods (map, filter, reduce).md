# Array methods (map, filter, reduce)

1. Quick answer

- `map`: transforms each item and returns a new array.
- `filter`: returns a new array containing only items that pass the predicate.
- `reduce`: reduces an array to a single value by accumulating results.

2. Examples

```js
const arr = [1,2,3,4];
const doubled = arr.map(x => x*2); // [2,4,6,8]
const evens = arr.filter(x => x%2 === 0); // [2,4]
const sum = arr.reduce((acc,x) => acc + x, 0); // 10
```

3. Performance tips

- Prefer `for` loops for extremely hot loops but prefer readability and functional approach for most code.
- Chain carefully: `arr.filter(...).map(...)` is readable; in some cases `reduce` can combine multiple steps.

4. Interview-ready summary

Use `map` to transform arrays, `filter` to select elements, and `reduce` to compute aggregated results. Pick the approach that balances readability and performance.
