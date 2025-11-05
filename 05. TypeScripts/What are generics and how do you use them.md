# What are generics and how do you use them

Generics in TypeScript allow you to create reusable components, functions, and types that work with multiple data types while maintaining type safety. Instead of using `any` or specific types, generics let you define type parameters that are determined when the generic is used. This provides flexibility without sacrificing type checking, making your code more maintainable and less prone to errors.

## 1. Basic Generic Syntax

**Generic Functions:**
```typescript
// Without generics - limited to specific types
function getFirstString(items: string[]): string {
  return items[0];
}

function getFirstNumber(items: number[]): number {
  return items[0];
}

// With generics - works with any type
function getFirst<T>(items: T[]): T {
  return items[0];
}

// Usage
const firstString = getFirst<string>(["hello", "world"]); // Type: string
const firstNumber = getFirst<number>([1, 2, 3]);         // Type: number
const firstBoolean = getFirst([true, false]);           // Type inferred as boolean
```

**Generic Interfaces:**
```typescript
interface Container<T> {
  value: T;
  getValue(): T;
  setValue(value: T): void;
}

// Implementation for different types
const stringContainer: Container<string> = {
  value: "hello",
  getValue() { return this.value; },
  setValue(value: string) { this.value = value; }
};

const numberContainer: Container<number> = {
  value: 42,
  getValue() { return this.value; },
  setValue(value: number) { this.value = value; }
};
```

## 2. Multiple Type Parameters

**Functions with Multiple Generics:**
```typescript
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const stringNumberPair = pair("hello", 42);     // Type: [string, number]
const booleanArrayPair = pair(true, [1, 2, 3]); // Type: [boolean, number[]]

// Key-value mapping function
function createMap<K extends string | number, V>(
  keys: K[], 
  values: V[]
): Map<K, V> {
  const map = new Map<K, V>();
  keys.forEach((key, index) => {
    map.set(key, values[index]);
  });
  return map;
}
```

## 3. Generic Constraints

**Constraining Generic Types:**
```typescript
// Constraint: T must have a length property
interface HasLength {
  length: number;
}

function getLength<T extends HasLength>(item: T): number {
  return item.length;
}

getLength("hello");        // ✅ string has length
getLength([1, 2, 3]);      // ✅ array has length
getLength({ length: 5 });  // ✅ object with length property
// getLength(42);          // ❌ Error: number doesn't have length

// Constraint: T must be a key of U
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person = { name: "John", age: 30, city: "NYC" };
const name = getProperty(person, "name");  // Type: string
const age = getProperty(person, "age");    // Type: number
// getProperty(person, "invalid"); // ❌ Error: invalid key
```

## 4. React-Specific Generic Patterns

**Generic React Components:**
```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor?: (item: T, index: number) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <div>
      {items.map((item, index) => (
        <div key={keyExtractor ? keyExtractor(item, index) : index}>
          {renderItem(item, index)}
        </div>
      ))}
    </div>
  );
}

// Usage with different types
interface User {
  id: number;
  name: string;
}

function UserList() {
  const users: User[] = [
    { id: 1, name: "John" },
    { id: 2, name: "Jane" }
  ];

  return (
    <List<User>
      items={users}
      renderItem={(user) => <span>{user.name}</span>}
      keyExtractor={(user) => user.id}
    />
  );
}

function NumberList() {
  const numbers = [1, 2, 3, 4, 5];
  
  return (
    <List<number>
      items={numbers}
      renderItem={(num) => <span>{num * 2}</span>}
    />
  );
}
```

**Generic Hooks:**
```typescript
// Generic useState hook pattern
function useLocalStorage<T>(
  key: string, 
  initialValue: T
): [T, (value: T) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  const setValue = (value: T) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue];
}

// Usage
function UserProfile() {
  const [user, setUser] = useLocalStorage<User | null>('user', null);
  const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light');
  
  return (
    <div>
      <p>User: {user?.name || 'Not logged in'}</p>
      <p>Theme: {theme}</p>
    </div>
  );
}
```

## 5. Advanced Generic Patterns

**Conditional Types with Generics:**
```typescript
type ApiResponse<T> = T extends string 
  ? { message: T }
  : T extends number 
    ? { count: T }
    : { data: T };

type StringResponse = ApiResponse<string>; // { message: string }
type NumberResponse = ApiResponse<number>; // { count: number }
type ObjectResponse = ApiResponse<User>;   // { data: User }
```

**Mapped Types with Generics:**
```typescript
// Make all properties optional
type Optional<T> = {
  [K in keyof T]?: T[K];
};

// Make all properties readonly
type ReadOnly<T> = {
  readonly [K in keyof T]: T[K];
};

// Pick specific properties
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type PartialUser = Optional<User>;           // All properties optional
type ReadOnlyUser = ReadOnly<User>;          // All properties readonly
type PublicUser = Pick<User, 'id' | 'name'>; // Only id and name
```

**Generic Classes:**
```typescript
class Repository<T> {
  private items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  findById<K extends keyof T>(id: T[K], key: K): T | undefined {
    return this.items.find(item => item[key] === id);
  }

  getAll(): T[] {
    return [...this.items];
  }

  filter(predicate: (item: T) => boolean): T[] {
    return this.items.filter(predicate);
  }
}

// Usage
const userRepository = new Repository<User>();
userRepository.add({ id: 1, name: "John", email: "john@example.com", password: "secret" });

const user = userRepository.findById(1, 'id'); // Type: User | undefined
const johns = userRepository.filter(user => user.name === "John"); // Type: User[]
```

## 6. Generic Utility Types

**Built-in Generic Utilities:**
```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial - makes all properties optional
type PartialUser = Partial<User>;

// Required - makes all properties required (removes ?)
type RequiredUser = Required<PartialUser>;

// Pick - selects specific properties
type PublicUser = Pick<User, 'id' | 'name' | 'email'>;

// Omit - excludes specific properties
type CreateUserInput = Omit<User, 'id' | 'createdAt'>;

// Record - creates object type with specific key-value types
type UserRoles = Record<string, 'admin' | 'user' | 'guest'>;

// Example usage in React
interface UpdateUserProps {
  user: User;
  onUpdate: (updates: Partial<User>) => void;
}

function UpdateUserForm({ user, onUpdate }: UpdateUserProps) {
  const handleSubmit = (formData: CreateUserInput) => {
    onUpdate(formData);
  };
  
  // Implementation...
}
```

## 7. Best Practices and Common Patterns

**Naming Conventions:**
```typescript
// Use descriptive single letters or full names
function map<T, U>(items: T[], transform: (item: T) => U): U[] {
  return items.map(transform);
}

// For more complex scenarios, use descriptive names
interface DataFetcher<TData, TParams, TError = Error> {
  fetch(params: TParams): Promise<TData>;
  handleError(error: TError): void;
}
```

**Default Generic Parameters:**
```typescript
interface ApiResponse<TData, TError = string> {
  data?: TData;
  error?: TError;
  loading: boolean;
}

// Usage - TError defaults to string
type UserResponse = ApiResponse<User>;           // TError is string
type CustomUserResponse = ApiResponse<User, CustomError>; // TError is CustomError
```

**Generic Function Overloads:**
```typescript
function process<T>(input: T[]): T[];
function process<T>(input: T): T;
function process<T>(input: T | T[]): T | T[] {
  if (Array.isArray(input)) {
    return input.map(item => item); // Process array
  }
  return input; // Process single item
}
```

## Interview-ready answer:
Generics in TypeScript allow you to write reusable, type-safe code that works with multiple types. You define type parameters using angle brackets `<T>` that act as placeholders for actual types. Generics are essential for creating flexible components - like a `List<T>` component that can render any type of data, or utility functions like `getProperty<T, K extends keyof T>()` that safely access object properties. They provide better type safety than `any`, enable code reuse without duplication, and power built-in utilities like `Partial<T>`, `Pick<T, K>`, and `Omit<T, K>`. In React, generics are commonly used for component props, custom hooks, and API response typing.