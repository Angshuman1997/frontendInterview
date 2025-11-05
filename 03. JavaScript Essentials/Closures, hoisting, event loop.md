# Closures, hoisting, event loop

1. Quick breakdown

- Closures: functions retaining access to lexical scope (see separate file for example).
- Hoisting: variable and function declarations are conceptually moved to the top of their scope. `var` declarations are hoisted and initialized with `undefined`; `let`/`const` are hoisted but in a temporal dead zone until initialized.
- Event loop: see separate file; coordinates async callbacks using microtasks and macrotasks.

2. Hoisting examples

```js
console.log(a); // undefined
var a = 10;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 5;
```

3. Interview-ready summary

Closures, hoisting, and the event loop are fundamental JS concepts: closures enable private state, hoisting affects variable initialization and scoping, and the event loop manages asynchronous execution.
