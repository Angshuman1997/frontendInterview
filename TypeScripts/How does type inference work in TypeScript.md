# How does type inference work in TypeScript

TypeScript's type inference automatically deduces types based on the values, context, and usage patterns in your code without requiring explicit type annotations. The TypeScript compiler analyzes the code flow, variable assignments, function returns, and contextual information to determine the most appropriate types. This reduces the need for manual type annotations while maintaining type safety and catching potential errors.

## 1. Basic Type Inference

**Variable Declaration Inference:**
```typescript
// TypeScript infers the type based on the initial value
let name = "John";           // Inferred as: let name: string
let age = 30;               // Inferred as: let age: number
let isActive = true;        // Inferred as: let isActive: boolean
let items = [1, 2, 3];      // Inferred as: let items: number[]
let mixed = [1, "hello"];   // Inferred as: let mixed: (string | number)[]

// const declarations have literal types
const color = "red";        // Inferred as: const color: "red" (literal type)
const count = 42;           // Inferred as: const count: 42 (literal type)

// Object inference
const user = {
  id: 1,
  name: "Alice",
  email: "alice@example.com"
}; 
// Inferred as: { id: number; name: string; email: string }

// Array with mixed types
const data = ["hello", 42, true];
// Inferred as: (string | number | boolean)[]
```

**Function Return Type Inference:**
```typescript
// Return type inferred from return statements
function add(a: number, b: number) {
  return a + b;  // Inferred return type: number
}

function greet(name: string) {
  return `Hello, ${name}!`;  // Inferred return type: string
}

function processUser(user: { id: number; name: string }) {
  return {
    ...user,
    displayName: user.name.toUpperCase()
  };
  // Inferred return type: { id: number; name: string; displayName: string }
}

// Conditional return types
function getValue(condition: boolean) {
  if (condition) {
    return "yes";
  }
  return 42;
}
// Inferred return type: string | number
```

## 2. Contextual Type Inference

**Function Parameter Inference:**
```typescript
// Array methods provide context for callback parameters
const numbers = [1, 2, 3, 4, 5];

numbers.map(num => num * 2);        // num inferred as number
numbers.filter(num => num > 3);     // num inferred as number
numbers.reduce((acc, num) => acc + num, 0); // acc and num inferred correctly

// Object methods
const users = [
  { id: 1, name: "John" },
  { id: 2, name: "Jane" }
];

users.find(user => user.id === 1);  // user inferred as { id: number; name: string }
users.sort((a, b) => a.name.localeCompare(b.name)); // a, b inferred from array type
```

**Event Handler Inference:**
```typescript
// React event handlers - types inferred from JSX
function Button() {
  const handleClick = (event) => {
    // event inferred as React.MouseEvent<HTMLButtonElement>
    console.log(event.currentTarget.textContent);
  };

  const handleChange = (event) => {
    // event inferred as React.ChangeEvent<HTMLInputElement>
    console.log(event.target.value);
  };

  return (
    <div>
      <button onClick={handleClick}>Click me</button>
      <input onChange={handleChange} />
    </div>
  );
}

// DOM event listeners
document.addEventListener('click', (event) => {
  // event inferred as MouseEvent
  console.log(event.clientX, event.clientY);
});
```

**Promise and Async/Await Inference:**
```typescript
// Promise inference
const fetchUser = async (id: number) => {
  const response = await fetch(`/api/users/${id}`);
  const user = await response.json();
  return user;
};
// Return type inferred as: Promise<any>

// Better with explicit return type for the API call
interface User {
  id: number;
  name: string;
  email: string;
}

const fetchUserTyped = async (id: number): Promise<User> => {
  const response = await fetch(`/api/users/${id}`);
  return response.json(); // TypeScript knows this should return User
};

// Usage inference
const handleUser = async () => {
  const user = await fetchUserTyped(1); // user inferred as User
  console.log(user.name); // TypeScript knows name exists
};
```

## 3. Generic Type Inference

**Function Generic Inference:**
```typescript
// Generic functions can infer types from arguments
function identity<T>(arg: T): T {
  return arg;
}

const stringResult = identity("hello");    // T inferred as string
const numberResult = identity(42);         // T inferred as number
const arrayResult = identity([1, 2, 3]);   // T inferred as number[]

// Multiple type parameters
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const stringNumberPair = pair("hello", 42);     // T: string, U: number
const booleanArrayPair = pair(true, [1, 2, 3]); // T: boolean, U: number[]
```

**Array and Object Generic Inference:**
```typescript
// Array constructor inference
const stringArray = Array.of("a", "b", "c"); // string[]
const numberArray = Array.of(1, 2, 3);       // number[]

// Map and Set inference
const userMap = new Map([
  [1, { name: "John" }],
  [2, { name: "Jane" }]
]);
// Inferred as: Map<number, { name: string }>

const stringSet = new Set(["apple", "banana", "cherry"]);
// Inferred as: Set<string>

// Promise.all inference
const promises = [
  Promise.resolve("hello"),
  Promise.resolve(42),
  Promise.resolve(true)
];

Promise.all(promises).then(results => {
  // results inferred as [string, number, boolean]
  const [str, num, bool] = results;
});
```

## 4. Advanced Inference Patterns

**Conditional Type Inference:**
```typescript
// Inference in conditional types
type ArrayElementType<T> = T extends (infer U)[] ? U : never;

type StringArrayElement = ArrayElementType<string[]>; // string
type NumberArrayElement = ArrayElementType<number[]>; // number

// Function parameter inference
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;

function exampleFunction(a: string, b: number): boolean {
  return true;
}

type ExampleParams = Parameters<typeof exampleFunction>; // [string, number]
type ExampleReturn = ReturnType<typeof exampleFunction>; // boolean
```

**Mapped Type Inference:**
```typescript
// Inference with mapped types
type Partial<T> = {
  [P in keyof T]?: T[P];
};

interface User {
  id: number;
  name: string;
  email: string;
}

const partialUser: Partial<User> = {
  name: "John"  // TypeScript infers this matches Partial<User>
};

// Template literal inference
type EventName<T extends string> = `on${Capitalize<T>}`;

type ClickEvent = EventName<"click">; // "onClick"
type HoverEvent = EventName<"hover">; // "onHover"
```

## 5. React-Specific Inference

**Component Props Inference:**
```typescript
// Props inference from usage
function Button({ children, onClick, disabled = false }) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {children}
    </button>
  );
}

// TypeScript infers the props type from usage
<Button onClick={() => console.log('clicked')}>
  Click me
</Button>

// Better with explicit props interface
interface ButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  disabled?: boolean;
}

function TypedButton({ children, onClick, disabled = false }: ButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {children}
    </button>
  );
}
```

**Hook Inference:**
```typescript
// useState inference
const [count, setCount] = useState(0); // count: number, setCount: (value: number | ((prev: number) => number)) => void
const [name, setName] = useState(""); // name: string, setName: (value: string | ((prev: string) => string)) => void
const [user, setUser] = useState(null); // user: null, setUser: (value: null | ((prev: null) => null)) => void

// Better with explicit typing for complex states
const [user, setUser] = useState<User | null>(null); // Now properly typed

// useEffect inference
useEffect(() => {
  // Cleanup function return type inferred
  const timer = setInterval(() => {}, 1000);
  return () => clearInterval(timer); // Return type inferred as void | (() => void)
}, []);

// Custom hook inference
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  
  const increment = () => setCount(prev => prev + 1);
  const decrement = () => setCount(prev => prev - 1);
  
  return { count, increment, decrement };
  // Return type inferred as: { count: number; increment: () => void; decrement: () => void }
}
```

## 6. Control Flow Analysis

**Type Narrowing Through Guards:**
```typescript
function processValue(value: string | number | null) {
  // Type narrowing through truthiness
  if (value) {
    // TypeScript knows value is string | number (not null)
    console.log(value.toString());
  }
  
  // Type narrowing through typeof
  if (typeof value === 'string') {
    // TypeScript knows value is string
    console.log(value.toUpperCase());
  } else if (typeof value === 'number') {
    // TypeScript knows value is number
    console.log(value.toFixed(2));
  } else {
    // TypeScript knows value is null
    console.log('Value is null');
  }
}

// Discriminated union inference
interface Circle {
  kind: 'circle';
  radius: number;
}

interface Rectangle {
  kind: 'rectangle';
  width: number;
  height: number;
}

type Shape = Circle | Rectangle;

function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      // TypeScript knows shape is Circle
      return Math.PI * shape.radius * shape.radius;
    case 'rectangle':
      // TypeScript knows shape is Rectangle
      return shape.width * shape.height;
  }
}
```

## 7. Best Practices for Type Inference

**When to Use Explicit Types:**
```typescript
// Good: Let inference work for simple cases
const name = "John";          // Let TypeScript infer string
const items = [1, 2, 3];      // Let TypeScript infer number[]

// Good: Explicit types for complex cases or public APIs
interface UserService {
  getUser(id: number): Promise<User>;  // Explicit return type for clarity
  updateUser(user: User): Promise<void>;
}

// Good: Explicit types for function parameters
function processUser(user: User): string {  // Explicit parameter type
  return user.name.toUpperCase();           // Return type can be inferred
}

// Avoid: Over-annotating obvious types
const count: number = 42;        // Unnecessary annotation
const isActive: boolean = true;  // Unnecessary annotation
```

**Helping TypeScript Infer Better:**
```typescript
// Use const assertions for better literal type inference
const colors = ['red', 'green', 'blue'] as const;
// Type: readonly ["red", "green", "blue"]

const config = {
  api: 'https://api.example.com',
  timeout: 5000,
  retries: 3
} as const;
// Properties become readonly with literal types

// Use type parameters to guide inference
function createArray<T>(item: T): T[] {
  return [item];
}

const stringArray = createArray("hello");   // string[]
const numberArray = createArray(42);        // number[]

// Provide better context for callbacks
const users: User[] = [];
users.map(user => ({
  id: user.id,
  displayName: user.name.toUpperCase()
}));
// TypeScript can infer the callback parameter and return types correctly
```

**Common Inference Pitfalls:**
```typescript
// Pitfall: Empty arrays lose type information
const items = []; // any[]
items.push("hello");
items.push(42); // No error, but probably not intended

// Solution: Provide explicit type
const items: string[] = [];
// or
const items = [] as string[];

// Pitfall: Null/undefined in unions
const getValue = (condition: boolean) => {
  if (condition) {
    return { id: 1, name: "John" };
  }
  // Implicitly returns undefined
};
// Return type: { id: number; name: string } | undefined

// Solution: Be explicit about all return paths
const getValue = (condition: boolean): User | null => {
  if (condition) {
    return { id: 1, name: "John" };
  }
  return null;
};
```

## Interview-ready answer:
TypeScript's type inference automatically deduces types from values, context, and usage patterns without explicit annotations. It works through variable initialization (inferring from assigned values), contextual typing (like array method callbacks), function return types (from return statements), and control flow analysis (narrowing types through guards). Inference is particularly powerful with generics, where types are deduced from arguments, and in React where event handlers and hook types are automatically inferred. While inference reduces boilerplate, explicit types are still important for function parameters, public APIs, and complex scenarios where inference might be too permissive or unclear.
