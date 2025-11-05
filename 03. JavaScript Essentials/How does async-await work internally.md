# How does async-await work internally

1. Quick answer

`async/await` transforms asynchronous code into a state machine built on top of Promises. `await` pauses execution of the async function and yields control to the event loop; the function resumes when the awaited promise settles.

2. Under the hood

- An `async` function always returns a Promise.
- When hitting `await`, the function returns to the caller and the remainder of the function is scheduled as a microtask that runs when the awaited promise resolves.
- Errors thrown inside async functions reject the returned promise.

3. Example transformation (conceptual)

```js
async function foo() {
  const a = await step1();
  const b = await step2(a);
  return b;
}
```
Transforms into roughly:
```js
function foo() {
  return step1().then(a => step2(a)).then(b => b);
}
```

4. Interview-ready summary

`async/await` is syntactic sugar over Promises. `await` yields to the event loop and resumes via microtasks when the awaited Promise resolves. It simplifies control flow and error propagation by using Promise chaining under the hood.
