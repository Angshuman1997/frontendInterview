# Promise - async-await flow

1. Quick answer

Promises represent eventual completion/failure of async operations. `async/await` is syntax sugar on top of promises that makes asynchronous code look synchronous.

2. Promise lifecycle

- pending → fulfilled or rejected.
- `then` handles fulfillment; `catch` handles rejection.

3. async/await

- `async` function returns a Promise.
- `await` pauses execution in the async function until the Promise resolves or rejects.

4. Error handling

- Use try/catch inside `async` functions or attach `.catch()` on the returned promise.

5. Example

```js
async function fetchUser() {
  try {
    const res = await fetch('/user');
    const user = await res.json();
    return user;
  } catch (err) {
    console.error(err);
    throw err;
  }
}
```

6. Interview-ready summary

Promises are the primitive for asynchronous operations; `async/await` simplifies chaining and error handling. Under the hood, `await` schedules a microtask to resume execution when the promise resolves.
