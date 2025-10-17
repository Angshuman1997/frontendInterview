# What's the difference between any, unknown, and never

TypeScript provides three special types that represent different concepts: `any`, `unknown`, and `never`. These types serve different purposes in the type system - `any` disables type checking, `unknown` provides type-safe handling of uncertain values, and `never` represents values that should never occur. Understanding these types is crucial for writing safe and maintainable TypeScript code.

## 1. The `any` Type - Type System Escape Hatch

The `any` type effectively disables TypeScript's type checking for a value. It's compatible with every other type, making it both powerful and dangerous.

**When `any` is used:**
```typescript
let value: any = 42;
value = "hello";     // ✅ No error
value = true;        // ✅ No error
value.foo.bar.baz;   // ✅ No error (but runtime error likely)

// any propagates through operations
const result = value + 1; // result is also any
```

**Practical Example - API Migration:**
```typescript
// During gradual TypeScript adoption
interface User {
  id: number;
  name: string;
  // Legacy data might have unknown structure
  metadata: any;
}

function processUser(user: User) {
  console.log(user.id);        // ✅ Type safe
  console.log(user.name);      // ✅ Type safe  
  console.log(user.metadata.anything); // ⚠️ No type checking
}
```

**Problems with `any`:**
- Disables all type checking
- Loses IntelliSense and autocomplete
- Spreads through your codebase
- Defeats the purpose of using TypeScript

## 2. The `unknown` Type - Type-Safe `any`

The `unknown` type represents values whose type we don't know ahead of time, but it requires type checking before use. It's the type-safe alternative to `any`.

**Basic Usage:**
```typescript
let value: unknown;

value = 42;          // ✅ Can assign anything
value = "hello";     // ✅ Can assign anything
value = { x: 1 };    // ✅ Can assign anything

// But can't use without type checking
console.log(value.toUpperCase()); // ❌ Error: Object is of type 'unknown'
console.log(value + 1);           // ❌ Error: Object is of type 'unknown'
```

**Type Guards with `unknown`:**
```typescript
function processUnknownValue(value: unknown) {
  // Type guards required
  if (typeof value === 'string') {
    console.log(value.toUpperCase()); // ✅ Now TypeScript knows it's string
  }
  
  if (typeof value === 'number') {
    console.log(value.toFixed(2));   // ✅ Now TypeScript knows it's number
  }
  
  if (value && typeof value === 'object' && 'name' in value) {
    console.log((value as { name: string }).name); // ✅ Type assertion after check
  }
}
```

**Real-World Example - API Responses:**
```typescript
async function fetchUserData(id: number): Promise<unknown> {
  const response = await fetch(`/api/users/${id}`);
  return response.json(); // JSON parsing returns unknown
}

async function handleUser() {
  const userData = await fetchUserData(1);
  
  // Must validate before use
  if (isValidUser(userData)) {
    console.log(userData.name); // ✅ Safe after validation
  }
}

function isValidUser(data: unknown): data is User {
  return (
    typeof data === 'object' &&
    data !== null &&
    typeof (data as User).id === 'number' &&
    typeof (data as User).name === 'string'
  );
}

interface User {
  id: number;
  name: string;
}
```

## 3. The `never` Type - The Empty Type

The `never` type represents values that never occur. It's used for functions that never return or values that should be impossible.

**Functions that never return:**
```typescript
// Function that throws error
function throwError(message: string): never {
  throw new Error(message);
}

// Function with infinite loop
function infiniteLoop(): never {
  while (true) {
    console.log("This runs forever");
  }
}

// Function that exits process
function exitProgram(): never {
  process.exit(1);
}
```

**Exhaustive Checking with `never`:**
```typescript
type Status = 'loading' | 'success' | 'error';

function handleStatus(status: Status): string {
  switch (status) {
    case 'loading':
      return 'Loading...';
    case 'success':
      return 'Success!';
    case 'error':
      return 'Error occurred';
    default:
      // This should never be reached
      const exhaustiveCheck: never = status;
      throw new Error(`Unhandled status: ${exhaustiveCheck}`);
  }
}

// If you add a new status type, TypeScript will error at the never assignment
type Status = 'loading' | 'success' | 'error' | 'pending'; // Added 'pending'
// Now the switch above will have a TypeScript error until you handle 'pending'
```

**React Example - Exhaustive Props Handling:**
```typescript
interface LoadingProps {
  state: 'loading';
  message?: string;
}

interface SuccessProps {
  state: 'success';
  data: any;
}

interface ErrorProps {
  state: 'error';
  error: string;
}

type ComponentProps = LoadingProps | SuccessProps | ErrorProps;

function StatusComponent(props: ComponentProps) {
  switch (props.state) {
    case 'loading':
      return <div>{props.message || 'Loading...'}</div>;
    case 'success':
      return <div>Data: {JSON.stringify(props.data)}</div>;
    case 'error':
      return <div>Error: {props.error}</div>;
    default:
      // Ensures all cases are handled
      const exhaustive: never = props;
      return null;
  }
}
```

## 4. Comparison and When to Use Each

### Type Hierarchy
```typescript
// Type assignability (top to bottom)
any          // Top type - accepts everything
unknown      // Top type - but safe
string       // Specific types
number
boolean
never        // Bottom type - accepts nothing
```

### Assignment Rules
```typescript
let anyValue: any;
let unknownValue: unknown;
let neverValue: never;
let stringValue: string;

// any accepts everything
anyValue = "hello";     // ✅
anyValue = 42;          // ✅
anyValue = unknownValue; // ✅

// unknown accepts everything  
unknownValue = "hello"; // ✅
unknownValue = 42;      // ✅
unknownValue = anyValue; // ✅

// Specific types accept their type and any
stringValue = "hello";   // ✅
stringValue = anyValue;  // ✅
stringValue = unknownValue; // ❌ Error
stringValue = neverValue;   // ✅ (never is assignable to everything)

// never accepts nothing
neverValue = "hello";    // ❌ Error
neverValue = anyValue;   // ❌ Error
neverValue = unknownValue; // ❌ Error
```

### Practical Guidelines

**Use `any` when:**
- Migrating JavaScript to TypeScript gradually
- Working with dynamic content where typing is impossible
- Prototyping (temporarily)
- Interfacing with untyped third-party libraries

```typescript
// Migration scenario
const legacyData: any = JSON.parse(localStorage.getItem('data') || '{}');
```

**Use `unknown` when:**
- Receiving data from external sources (APIs, user input)
- You genuinely don't know the type but want type safety
- Building generic utilities that work with any type

```typescript
// Generic utility function
function stringify(value: unknown): string {
  if (typeof value === 'string') {
    return value;
  }
  return JSON.stringify(value);
}
```

**Use `never` when:**
- Functions that never return normally
- Exhaustive checking in discriminated unions
- Representing impossible states
- Bottom types in conditional type logic

```typescript
// Utility type that extracts never-returning functions
type NonReturning<T> = T extends (...args: any[]) => never ? T : never;
```

## 5. React-Specific Patterns

**Error Boundaries with `never`:**
```typescript
interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

class ErrorBoundary extends React.Component<React.PropsWithChildren<{}>, ErrorBoundaryState> {
  constructor(props: React.PropsWithChildren<{}>) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): never {
    console.error('Error caught by boundary:', error, errorInfo);
    throw error; // Re-throw to trigger never type
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }
    return this.props.children;
  }
}
```

**Props Validation with `unknown`:**
```typescript
function validateProps(props: unknown): props is { name: string; age: number } {
  return (
    typeof props === 'object' &&
    props !== null &&
    'name' in props &&
    'age' in props &&
    typeof (props as any).name === 'string' &&
    typeof (props as any).age === 'number'
  );
}

function MyComponent(props: unknown) {
  if (!validateProps(props)) {
    throw new Error('Invalid props');
  }
  
  // Now props is properly typed
  return <div>{props.name} is {props.age} years old</div>;
}
```

## Interview-ready answer:
`any`, `unknown`, and `never` serve different purposes in TypeScript's type system. `any` disables type checking entirely - it accepts any value and allows any operation, but loses all type safety. `unknown` is the type-safe alternative to `any` - it can hold any value but requires type guards or assertions before use. `never` represents values that never occur, used for functions that never return or exhaustive checking in switch statements. Use `any` sparingly for migration or prototyping, prefer `unknown` when you genuinely don't know a type but want safety, and use `never` for impossible states and exhaustive type checking.