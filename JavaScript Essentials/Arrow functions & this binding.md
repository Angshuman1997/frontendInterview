# Arrow functions & this binding

1. Quick answer

Arrow functions don't have their own `this` binding — they inherit `this` from the surrounding lexical scope. They also don't have `arguments`, `super`, or `new.target`.

2. Examples

```js
const obj = {
  value: 10,
  regular() { return this.value; },
  arrow: () => this.value
};
obj.regular(); // 10
obj.arrow(); // undefined (or global value) because arrow's this is lexical (module/global)
```

3. When to use

- Arrow functions are great for callbacks and preserving the surrounding `this` (e.g., inside methods passing functions to event handlers).
- Don't use arrow functions as object methods when you need a dynamic `this`.

4. Interview-ready summary

Arrow functions inherit `this` from the outer scope; they are concise and ideal for callbacks but unsuitable as object methods that rely on their own `this`.
