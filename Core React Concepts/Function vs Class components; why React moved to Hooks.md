# Function vs Class components; why React moved to Hooks

React originally used class components to manage state and lifecycle methods. Functional components were stateless and used only for rendering UI. However, React introduced Hooks in version 16.8, which allowed functional components to manage state, side effects, and other React features without writing classes. Hooks made components simpler, more readable, and more reusable.

1. Difference Between Class and Functional Components

Class Components:

* Use ES6 classes and extend React.Component.
* Manage state using this.state and this.setState().
* Use lifecycle methods like componentDidMount(), componentDidUpdate(), and componentWillUnmount().
* Have more boilerplate (constructor, binding methods, etc.).
* "this" keyword must be used and managed carefully.

Example:
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

Functional Components:

* Plain JavaScript functions that return JSX.
* Use Hooks like useState() and useEffect() for state and lifecycle logic.
* Simpler, cleaner, and easier to test.
* No need for constructors or the “this” keyword.
* Support better logic reuse through custom hooks.

Example:
function Counter() {
const [count, setCount] = useState(0);

return ( <div> <p>Count: {count}</p>
<button onClick={() => setCount(count + 1)}>Increment</button> </div>
);
}

2. Why React Moved to Hooks

Simpler and cleaner code:
Hooks reduce boilerplate by removing class syntax and the need to bind “this”.

Better code reuse:
Hooks allow you to extract and reuse logic using custom hooks, instead of relying on higher-order components (HOCs) or render props.

Easier to understand lifecycle:
Hooks like useEffect() unify lifecycle behavior for mount, update, and unmount phases.

Eliminate “this” confusion:
Functional components don’t require “this”, which reduces bugs caused by incorrect bindings.

Smaller and faster components:
Functional components are easier to optimize and have smaller memory footprints.

Improved composition:
Hooks encourage splitting logic by purpose rather than lifecycle, leading to more modular and maintainable code.

Example of logic reuse with a custom hook:
function useWindowWidth() {
const [width, setWidth] = useState(window.innerWidth);

useEffect(() => {
const handleResize = () => setWidth(window.innerWidth);
window.addEventListener('resize', handleResize);
return () => window.removeEventListener('resize', handleResize);
}, []);

return width;
}

function App() {
const width = useWindowWidth();
return <p>Window width: {width}</p>;
}

Interview-ready answer:
Class components were used to manage state and lifecycle methods, but they involved complex syntax and “this” bindings. Functional components were simpler but lacked features. React introduced Hooks to bring state and lifecycle management to functional components. Hooks make code simpler, improve reusability through custom hooks, remove “this” confusion, and unify component logic handling. Now, functional components with Hooks are preferred for all new React development.
