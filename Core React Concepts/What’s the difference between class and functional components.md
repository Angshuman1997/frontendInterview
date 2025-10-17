# What’s the difference between class and functional components

Class components and functional components are two ways of defining React components. Both can render UI, but they differ in syntax, how they manage state, lifecycle methods, and overall complexity.

1. **Class Components**

* Use ES6 classes and extend `React.Component`.
* Manage state using `this.state` and update it using `this.setState()`.
* Use lifecycle methods such as `componentDidMount()`, `componentDidUpdate()`, and `componentWillUnmount()`.
* Require `this` binding for event handlers.
* Have more boilerplate code and are less concise.

**Example:**
class Counter extends React.Component {
constructor(props) {
super(props);
this.state = { count: 0 };
}

increment = () => {
this.setState({ count: this.state.count + 1 });
};

render() {
return ( <div> <p>Count: {this.state.count}</p> <button onClick={this.increment}>Increment</button> </div>
);
}
}

2. **Functional Components**

* Are plain JavaScript functions that return JSX.
* Use React Hooks (`useState`, `useEffect`, etc.) for state and lifecycle logic.
* No need for constructors or `this` keyword.
* Simpler, cleaner, and easier to read and test.
* Preferred in modern React development.

**Example:**
function Counter() {
const [count, setCount] = useState(0);

return ( <div> <p>Count: {count}</p>
<button onClick={() => setCount(count + 1)}>Increment</button> </div>
);
}

3. **Key Differences**

| Feature           | Class Component              | Functional Component   |
| ----------------- | ---------------------------- | ---------------------- |
| Syntax            | Uses ES6 class               | Uses plain JS function |
| State Management  | this.state & this.setState() | useState() hook        |
| Lifecycle Methods | componentDidMount(), etc.    | useEffect()            |
| "this" Keyword    | Required                     | Not required           |
| Performance       | Slightly heavier             | Lightweight            |
| Logic Reuse       | Hard (HOCs, render props)    | Easy (custom hooks)    |
| Recommended       | Legacy                       | Modern React standard  |

4. **Why React Moved to Functional Components**

* Hooks provide state and lifecycle features without classes.
* Cleaner, more readable, and easier to maintain code.
* Encourages code reuse through custom hooks.
* Eliminates issues with “this” binding.
* Better for optimizing rendering and managing side effects.

**Interview-ready answer:**
Class components use ES6 classes, manage state with `this.state`, and rely on lifecycle methods. Functional components are simpler functions that use React Hooks to manage state and side effects. Hooks made functional components more powerful, eliminating the need for most class components. Today, functional components are preferred because they are cleaner, easier to maintain, and more efficient.
