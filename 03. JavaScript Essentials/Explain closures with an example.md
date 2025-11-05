# Explain closures with an example

1. Quick answer

A closure is a function that retains access to variables from its lexical scope even after the outer function has returned.

2. Example

```js
function makeCounter() {
  let count = 0;
  return function () {
    count += 1;
    return count;
  };
}
const counter = makeCounter();
counter(); // 1
counter(); // 2
```
`counter` closes over `count` and keeps it alive across calls.

3. Use cases

- Data encapsulation (private state)
- Function factories
- Memoization

4. Interview-ready summary

Closures let functions capture and keep references to variables in their lexical environment, enabling patterns like private state and function factories.
