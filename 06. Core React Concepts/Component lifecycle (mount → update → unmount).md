# Component lifecycle (mount → update → unmount)

React components go through three main lifecycle phases — mounting, updating, and unmounting. These define how a component is created, updated, and destroyed in the UI. Understanding this helps manage side effects, subscriptions, and resource cleanup properly.

1. Mounting Phase (Component Creation)
   This phase occurs when the component is first added to the DOM.

For class components, the key lifecycle methods are:

* constructor(): Initializes state and binds event handlers.
* static getDerivedStateFromProps(): Syncs state with props before rendering.
* render(): Returns JSX to render the UI.
* componentDidMount(): Called after the component is rendered. Good place for API calls, subscriptions, or DOM manipulations.

Example (Class Component):
class Example extends React.Component {
constructor(props) {
super(props);
this.state = { count: 0 };
}

componentDidMount() {
console.log('Component mounted');
}

render() {
return <h1>{this.state.count}</h1>;
}
}

For function components with Hooks:
The equivalent of componentDidMount is a useEffect hook with an empty dependency array [].

Example:
useEffect(() => {
console.log('Component mounted');
}, []);

2. Updating Phase (State or Props Change)
   This happens when the component re-renders due to changes in props or state.

Key lifecycle methods (Class):

* static getDerivedStateFromProps(): Updates state based on new props.
* shouldComponentUpdate(): Determines if re-render is needed (used for performance optimization).
* render(): Re-renders the UI with new data.
* getSnapshotBeforeUpdate(): Captures some information (like scroll position) before DOM updates.
* componentDidUpdate(): Runs after the DOM is updated. Useful for performing side effects like API calls when data changes.

Function component equivalent:
useEffect(() => {
console.log('Component updated');
}, [dependencies]);

3. Unmounting Phase (Component Removal)
   This phase occurs when the component is removed from the DOM.

Key lifecycle method (Class):

* componentWillUnmount(): Cleanup logic like removing event listeners, canceling timers, or aborting API calls.

Function component equivalent:
useEffect(() => {
const timer = setInterval(() => console.log('Running'), 1000);
return () => clearInterval(timer); // cleanup
}, []);

Example (Function Component):
useEffect(() => {
console.log('Component mounted');
return () => {
console.log('Component unmounted');
};
}, []);

Lifecycle summary table:

| Phase   | Class Component      | Functional (Hooks)          |
| ------- | -------------------- | --------------------------- |
| Mount   | componentDidMount    | useEffect(() => {}, [])     |
| Update  | componentDidUpdate   | useEffect(() => {}, [deps]) |
| Unmount | componentWillUnmount | useEffect cleanup function  |

Interview-ready answer:
React components have three main lifecycle phases — mount, update, and unmount. During mounting, the component is created and inserted into the DOM (useEffect with [] or componentDidMount). During updating, it re-renders when state or props change (useEffect with dependencies or componentDidUpdate). During unmounting, it is removed from the DOM, and cleanup is performed (useEffect cleanup or componentWillUnmount). In modern React, Hooks like useEffect handle all these lifecycle stages in functional components.
