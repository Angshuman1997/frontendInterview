# JSX → how it compiles to React.createElement

JSX (JavaScript XML) is a syntax extension used in React that lets developers write HTML-like code within JavaScript. It makes UI code more readable and declarative. However, browsers do not understand JSX directly. Tools like Babel compile JSX into plain JavaScript using React.createElement() calls.

When you write:

const element = <h1>Hello React</h1>;

Babel compiles it into:

const element = React.createElement(
'h1', 
{classname: 'main'}, 
'Hello React'
);

React.createElement() returns a plain JavaScript object called a “React element,” which describes what should appear on the screen. React uses these React elements to construct and compare Virtual DOM trees before updating the real DOM efficiently.

React.createElement(type, props, ...children)

type: The type of element (e.g., 'div', 'h1', or a component)

props: An object of attributes or properties (e.g., className, id)

children: Any child elements or text

Example:
React.createElement('button', { className: 'btn' }, 'Click');

produces:
{
type: 'button',
props: { className: 'btn', children: 'Click' }
}

This object is not a DOM node but a lightweight description of one. React compares old and new React elements (via reconciliation) to update only the parts that changed.

Why use JSX:

Easier to read and maintain than nested createElement() calls.

Helps visualize the UI structure.

JSX supports embedding JavaScript expressions using curly braces {}.

Key points:

JSX compiles to React.createElement() via Babel.

React elements are plain JS objects, not DOM nodes.

JSX must have a single parent element.

Before React 17, importing React was required for JSX; React 17+ uses the new JSX transform and doesn’t need it.

Interview-ready answer:
JSX is syntactic sugar for React.createElement() calls. Babel compiles JSX into React.createElement(type, props, children), which returns a React element (a plain JS object). React uses these elements to build and update the Virtual DOM efficiently. JSX improves readability but ultimately compiles to pure JavaScript.