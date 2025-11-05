# How do you access and manipulate DOM elements in vanilla JS

1. Quick answer

Use APIs like `querySelector`, `getElementById`, and `getElementsByClassName` to select elements. Manipulate them with properties and methods like `textContent`, `innerHTML`, `classList`, `setAttribute`, and DOM methods (`appendChild`, `removeChild`).

2. Examples

```js
const btn = document.querySelector('.btn');
btn.addEventListener('click', () => {
  const el = document.createElement('div');
  el.textContent = 'Hello';
  document.body.appendChild(el);
});
```

3. Performance tips

- Minimize layout reads and writes; batch DOM changes using `DocumentFragment`.
- Cache selectors if used frequently.

4. Interview-ready summary

Select elements with DOM query methods, mutate with property setters and DOM APIs, and keep performance in mind by batching updates and reducing layout thrashing.
