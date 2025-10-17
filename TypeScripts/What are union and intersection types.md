# What are union and intersection types

Union and intersection types are powerful TypeScript features that allow you to combine types in different ways. Union types (`|`) represent values that can be one of several types, while intersection types (`&`) combine multiple types into one that has all properties of the combined types. These type operations are essential for creating flexible yet type-safe code, especially when dealing with complex data structures and API responses.

## 1. Union Types (`|`) - "OR" Logic

Union types allow a value to be one of several possible types. The value must match at least one of the types in the union.

**Basic Union Types:**
```typescript
// Primitive unions
type Status = 'pending' | 'approved' | 'rejected';
type ID = string | number;
type Theme = 'light' | 'dark' | 'auto';

// Using union types
let userStatus: Status = 'pending';    // ✅ Valid
userStatus = 'approved';               // ✅ Valid
// userStatus = 'invalid';             // ❌ Error

let userId: ID = 123;                  // ✅ Valid (number)
userId = 'user_456';                   // ✅ Valid (string)
// userId = true;                      // ❌ Error
```

**Object Union Types:**
```typescript
interface LoadingState {
  status: 'loading';
  progress?: number;
}

interface SuccessState {
  status: 'success';
  data: any;
  timestamp: number;
}

interface ErrorState {
  status: 'error';
  error: string;
  retryable: boolean;
}

type AppState = LoadingState | SuccessState | ErrorState;

// Usage - must handle all possible states
function handleAppState(state: AppState) {
  switch (state.status) {
    case 'loading':
      console.log('Loading progress:', state.progress); // TypeScript knows this is LoadingState
      break;
    case 'success':
      console.log('Data received:', state.data);       // TypeScript knows this is SuccessState
      break;
    case 'error':
      console.log('Error:', state.error);              // TypeScript knows this is ErrorState
      break;
  }
}
```

**Function Parameter Unions:**
```typescript
function formatValue(value: string | number | Date): string {
  if (typeof value === 'string') {
    return value.toUpperCase();
  }
  if (typeof value === 'number') {
    return value.toFixed(2);
  }
  if (value instanceof Date) {
    return value.toISOString();
  }
  // TypeScript ensures all cases are handled
}
```

## 2. Intersection Types (`&`) - "AND" Logic

Intersection types combine multiple types into one that has all properties of the intersected types.

**Basic Intersection Types:**
```typescript
interface Person {
  name: string;
  age: number;
}

interface Employee {
  employeeId: number;
  department: string;
}

type PersonEmployee = Person & Employee;
// Has all properties: name, age, employeeId, department

const worker: PersonEmployee = {
  name: 'John Doe',
  age: 30,
  employeeId: 12345,
  department: 'Engineering'
};
```

**Mixin Pattern with Intersections:**
```typescript
// Base types
interface Timestamps {
  createdAt: Date;
  updatedAt: Date;
}

interface Trackable {
  id: string;
  version: number;
}

interface User {
  name: string;
  email: string;
}

// Combined types using intersections
type DatabaseUser = User & Timestamps & Trackable;

const dbUser: DatabaseUser = {
  // User properties
  name: 'Alice Smith',
  email: 'alice@example.com',
  // Timestamps properties
  createdAt: new Date(),
  updatedAt: new Date(),
  // Trackable properties
  id: 'user_123',
  version: 1
};
```

## 3. Advanced Union Patterns

**Discriminated Unions (Tagged Unions):**
```typescript
// Each variant has a common discriminator property
interface CircleShape {
  kind: 'circle';
  radius: number;
}

interface RectangleShape {
  kind: 'rectangle';
  width: number;
  height: number;
}

interface TriangleShape {
  kind: 'triangle';
  base: number;
  height: number;
}

type Shape = CircleShape | RectangleShape | TriangleShape;

function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius * shape.radius; // TypeScript knows shape is CircleShape
    case 'rectangle':
      return shape.width * shape.height;            // TypeScript knows shape is RectangleShape
    case 'triangle':
      return (shape.base * shape.height) / 2;       // TypeScript knows shape is TriangleShape
    default:
      // Exhaustive checking - ensures all cases are handled
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape: ${_exhaustive}`);
  }
}
```

**Optional Property Unions:**
```typescript
// Different API response shapes
interface UserWithPosts {
  user: User;
  posts: Post[];
  totalPosts: number;
}

interface UserWithoutPosts {
  user: User;
}

type UserResponse = UserWithPosts | UserWithoutPosts;

function processUserResponse(response: UserResponse) {
  console.log('User:', response.user);
  
  // Type guard to check which variant we have
  if ('posts' in response) {
    console.log(`User has ${response.posts.length} posts`); // TypeScript knows this is UserWithPosts
  }
}
```

## 4. React-Specific Patterns

**Component Props with Unions:**
```typescript
// Button can be either a button or a link
interface ButtonAsButton {
  as: 'button';
  onClick: () => void;
  disabled?: boolean;
}

interface ButtonAsLink {
  as: 'link';
  href: string;
  target?: '_blank' | '_self';
}

interface BaseButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary';
}

type ButtonProps = BaseButtonProps & (ButtonAsButton | ButtonAsLink);

function Button(props: ButtonProps) {
  if (props.as === 'button') {
    return (
      <button 
        onClick={props.onClick}      // TypeScript knows onClick exists
        disabled={props.disabled}    // TypeScript knows disabled exists
        className={`btn ${props.variant}`}
      >
        {props.children}
      </button>
    );
  }

  return (
    <a 
      href={props.href}           // TypeScript knows href exists
      target={props.target}       // TypeScript knows target exists
      className={`btn ${props.variant}`}
    >
      {props.children}
    </a>
  );
}

// Usage
<Button as="button" onClick={() => console.log('clicked')}>
  Click me
</Button>

<Button as="link" href="https://example.com" target="_blank">
  Visit link
</Button>
```

**Event Handler Unions:**
```typescript
type FormInputElement = HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement;

interface FormFieldProps {
  value: string;
  onChange: (e: React.ChangeEvent<FormInputElement>) => void;
  placeholder?: string;
  type?: 'input' | 'textarea' | 'select';
  options?: string[]; // Only used for select
}

function FormField({ value, onChange, placeholder, type = 'input', options }: FormFieldProps) {
  if (type === 'textarea') {
    return (
      <textarea 
        value={value}
        onChange={onChange}
        placeholder={placeholder}
      />
    );
  }

  if (type === 'select' && options) {
    return (
      <select value={value} onChange={onChange}>
        {options.map(option => (
          <option key={option} value={option}>{option}</option>
        ))}
      </select>
    );
  }

  return (
    <input 
      type="text"
      value={value}
      onChange={onChange}
      placeholder={placeholder}
    />
  );
}
```

## 5. Advanced Intersection Patterns

**Function Type Intersections:**
```typescript
interface Validator<T> {
  validate: (value: T) => boolean;
}

interface Formatter<T> {
  format: (value: T) => string;
}

interface Parser<T> {
  parse: (value: string) => T;
}

// Intersection of multiple function interfaces
type StringProcessor = Validator<string> & Formatter<string> & Parser<string>;

const emailProcessor: StringProcessor = {
  validate: (email: string) => /\S+@\S+\.\S+/.test(email),
  format: (email: string) => email.toLowerCase().trim(),
  parse: (value: string) => value // Simple pass-through for strings
};
```

**Conditional Type Intersections:**
```typescript
type ApiResponse<T> = {
  data: T;
  status: 'success';
} | {
  error: string;
  status: 'error';
};

type WithTimestamp<T> = T & {
  timestamp: number;
};

type TimestampedResponse<T> = WithTimestamp<ApiResponse<T>>;

// Usage
const response: TimestampedResponse<User> = {
  data: { name: 'John', email: 'john@example.com' },
  status: 'success',
  timestamp: Date.now()
};
```

## 6. Type Guards and Narrowing

**Custom Type Guards for Unions:**
```typescript
function isLoadingState(state: AppState): state is LoadingState {
  return state.status === 'loading';
}

function isSuccessState(state: AppState): state is SuccessState {
  return state.status === 'success';
}

function isErrorState(state: AppState): state is ErrorState {
  return state.status === 'error';
}

// Usage with type guards
function displayState(state: AppState) {
  if (isLoadingState(state)) {
    return `Loading... ${state.progress || 0}%`;
  }
  
  if (isSuccessState(state)) {
    return `Success! Data: ${JSON.stringify(state.data)}`;
  }
  
  if (isErrorState(state)) {
    return `Error: ${state.error} (Retryable: ${state.retryable})`;
  }
}
```

**Built-in Type Guards:**
```typescript
function processValue(value: string | number | null | undefined) {
  // typeof type guard
  if (typeof value === 'string') {
    return value.toUpperCase(); // TypeScript knows value is string
  }
  
  if (typeof value === 'number') {
    return value.toFixed(2);    // TypeScript knows value is number
  }
  
  // Truthiness type guard
  if (value) {
    // This won't be reached due to previous checks, but demonstrates the pattern
  }
  
  // Null/undefined check
  if (value == null) {
    return 'No value provided';
  }
}
```

## 7. Common Patterns and Best Practices

**API Response Modeling:**
```typescript
// Base response structure
interface BaseApiResponse {
  requestId: string;
  timestamp: number;
}

// Success response
interface ApiSuccess<T> extends BaseApiResponse {
  success: true;
  data: T;
}

// Error response
interface ApiError extends BaseApiResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
  };
}

type ApiResponse<T> = ApiSuccess<T> | ApiError;

// Generic API handler
async function handleApiCall<T>(
  apiCall: () => Promise<ApiResponse<T>>
): Promise<T> {
  const response = await apiCall();
  
  if (response.success) {
    return response.data; // TypeScript knows this is ApiSuccess<T>
  } else {
    throw new Error(response.error.message); // TypeScript knows this is ApiError
  }
}
```

**State Machine Patterns:**
```typescript
interface IdleState {
  state: 'idle';
}

interface LoadingState {
  state: 'loading';
  startedAt: number;
}

interface SuccessState {
  state: 'success';
  data: any;
  completedAt: number;
}

interface ErrorState {
  state: 'error';
  error: string;
  failedAt: number;
}

type AsyncState = IdleState | LoadingState | SuccessState | ErrorState;

// State transition functions
function startLoading(): LoadingState {
  return { state: 'loading', startedAt: Date.now() };
}

function completeSuccess(data: any): SuccessState {
  return { state: 'success', data, completedAt: Date.now() };
}

function failWithError(error: string): ErrorState {
  return { state: 'error', error, failedAt: Date.now() };
}
```

## Interview-ready answer:
Union types (`|`) represent values that can be one of several types, like `string | number`, and are useful for handling multiple possible data shapes. Intersection types (`&`) combine multiple types into one that has all properties, like `User & Timestamps`. Union types are perfect for discriminated unions (like different API response states), optional configurations, and polymorphic components. Intersection types excel at mixins, combining base interfaces, and adding metadata to existing types. TypeScript provides excellent type narrowing for unions through discriminator properties, type guards, and built-in checks like `typeof`. These patterns are essential in React for flexible component props, state management, and API response handling.
