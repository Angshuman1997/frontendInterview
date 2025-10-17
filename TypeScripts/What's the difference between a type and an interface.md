# What's the difference between a type and an interface

Both `type` aliases and `interface` declarations in TypeScript are used to define custom types, but they have different capabilities and use cases. While they overlap in many scenarios, understanding their differences helps you choose the right tool for the job. Interfaces are primarily for object shapes and support declaration merging, while types are more flexible and can represent any type including unions, primitives, and computed types.

## 1. Basic Syntax and Usage

**Interface Declaration:**
```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

interface UserActions {
  create(user: User): Promise<User>;
  update(id: number, updates: Partial<User>): Promise<User>;
  delete(id: number): Promise<void>;
}
```

**Type Alias:**
```typescript
type User = {
  id: number;
  name: string;
  email: string;
};

type UserActions = {
  create(user: User): Promise<User>;
  update(id: number, updates: Partial<User>): Promise<User>;
  delete(id: number): Promise<void>;
};
```

## 2. Key Differences

### Declaration Merging
**Interfaces support declaration merging (can be extended across multiple declarations):**
```typescript
interface User {
  id: number;
  name: string;
}

// Later in the same scope - this merges with the above
interface User {
  email: string;
  age?: number;
}

// Final User interface has: id, name, email, age
const user: User = {
  id: 1,
  name: "John",
  email: "john@example.com",
  age: 30
};
```

**Types cannot be reopened or merged:**
```typescript
type User = {
  id: number;
  name: string;
};

// Error: Duplicate identifier 'User'
type User = {
  email: string;
};
```

### Extending/Inheritance
**Interface extending:**
```typescript
interface Animal {
  name: string;
  age: number;
}

interface Dog extends Animal {
  breed: string;
  bark(): void;
}

// Multiple inheritance
interface Pet {
  owner: string;
}

interface DomesticDog extends Animal, Pet {
  breed: string;
}
```

**Type intersection:**
```typescript
type Animal = {
  name: string;
  age: number;
};

type Dog = Animal & {
  breed: string;
  bark(): void;
};

// Multiple intersection
type Pet = {
  owner: string;
};

type DomesticDog = Animal & Pet & {
  breed: string;
};
```

## 3. Unique Capabilities

### What Only Types Can Do

**Union Types:**
```typescript
type Status = 'loading' | 'success' | 'error';
type ID = string | number;

// Discriminated unions
type Shape = 
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'square'; size: number };
```

**Primitive Types and Computed Types:**
```typescript
// Primitive aliases
type UserID = string;
type Count = number;

// Computed types
type UserKeys = keyof User; // 'id' | 'name' | 'email'
type UserValues = User[keyof User]; // string | number

// Conditional types
type NonNullable<T> = T extends null | undefined ? never : T;

// Mapped types
type Optional<T> = {
  [K in keyof T]?: T[K];
};
```

**Function Types:**
```typescript
type EventHandler<T> = (event: T) => void;
type APICall<T, R> = (params: T) => Promise<R>;

// Function overloads in types
type Formatter = {
  (value: string): string;
  (value: number): string;
  (value: Date): string;
};
```

### What Only Interfaces Can Do (Practically)

**Class Implementation:**
```typescript
interface Flyable {
  fly(): void;
  altitude: number;
}

interface Swimmable {
  swim(): void;
  depth: number;
}

// Classes can implement multiple interfaces
class Duck implements Flyable, Swimmable {
  altitude = 0;
  depth = 0;
  
  fly() {
    this.altitude = 100;
  }
  
  swim() {
    this.depth = 5;
  }
}
```

## 4. React-Specific Examples

**Component Props:**
```typescript
// Both work for React props
interface ButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
}

type ButtonProps = {
  children: React.ReactNode;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
};

// But types are more flexible for complex prop patterns
type ConditionalProps<T extends boolean> = T extends true
  ? { required: string; optional?: never }
  : { required?: never; optional: string };

type MyComponentProps<T extends boolean = false> = {
  hasRequired: T;
} & ConditionalProps<T>;
```

**Event Handlers:**
```typescript
// Interface approach
interface FormHandlers {
  onSubmit: (event: React.FormEvent) => void;
  onChange: (event: React.ChangeEvent<HTMLInputElement>) => void;
}

// Type approach - more flexible
type EventHandlers<T extends HTMLElement = HTMLElement> = {
  onClick?: (event: React.MouseEvent<T>) => void;
  onFocus?: (event: React.FocusEvent<T>) => void;
};

type ButtonHandlers = EventHandlers<HTMLButtonElement>;
type InputHandlers = EventHandlers<HTMLInputElement>;
```

## 5. Performance Considerations

**Interfaces are generally faster for:**
- Type checking (especially with declaration merging)
- IDE performance in large codebases
- Inheritance hierarchies

**Types are necessary for:**
- Complex type computations
- Union types
- Advanced type manipulations

## 6. Best Practices and When to Use Each

**Use Interfaces When:**
- Defining object shapes that might need extension
- Creating contracts that classes will implement
- Building public APIs that might evolve (declaration merging benefit)
- Working with object-oriented patterns

```typescript
// Good interface use case
interface ApiClient {
  baseURL: string;
  get<T>(endpoint: string): Promise<T>;
  post<T>(endpoint: string, data: unknown): Promise<T>;
}

// Can be extended later
interface ApiClient {
  timeout: number;
}
```

**Use Types When:**
- Creating union types or complex type compositions
- Working with primitives, functions, or computed types
- Building utility types or type transformations
- Need conditional or mapped types

```typescript
// Good type use case
type APIResponse<T> = 
  | { status: 'success'; data: T }
  | { status: 'error'; error: string }
  | { status: 'loading' };

type ComponentState<T> = {
  data: T | null;
  loading: boolean;
  error: string | null;
};
```

## Interview-ready answer:
Both interfaces and types can define object shapes, but they have key differences. Interfaces support declaration merging (can be reopened and extended in the same scope), are optimized for inheritance with the `extends` keyword, and are ideal for defining contracts that classes implement. Types are more flexible - they can represent unions, primitives, computed types, and complex type manipulations that interfaces cannot handle. Use interfaces for object shapes that might evolve and class contracts; use types for unions, utility types, and complex type compositions. In React, both work for props, but types offer more flexibility for advanced patterns.