# What is the event loop

1. Quick answer

The event loop is the mechanism that allows JavaScript (single-threaded) to perform asynchronous operations by processing a task queue and microtask queue, coordinating callbacks, promises, and I/O without blocking the main thread.

2. Key concepts

- Call stack: where functions are executed.
- Web APIs: timers, DOM events, network requests provided by the environment.
- Callback queue (task queue): holds callbacks from `setTimeout`, `setInterval`, I/O, and events.
- Microtask queue: holds Promise callbacks and `queueMicrotask`; microtasks run after the current task but before the next task.

3. Execution order

1. Execute current call stack.
2. Run all microtasks (Promise resolutions, mutation observer callbacks).
3. Render if needed.
4. Take the next task from the task queue and repeat.

4. Example

```js
console.log('start');
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('end');
// Output: start, end, promise, timeout
```

5. Interview-ready summary

The event loop orchestrates asynchronous behavior by using a call stack, microtask queue (promises), and task queue (timers, events). Microtasks run before the next macrotask, ensuring Promise callbacks execute quickly after the current synchronous code.
