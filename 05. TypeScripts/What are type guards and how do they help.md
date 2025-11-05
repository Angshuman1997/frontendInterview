# What are type guards and how do they help

Type guards are TypeScript mechanisms that allow you to narrow down the type of a variable within a specific scope. They help TypeScript understand which specific type a value has when dealing with union types, unknown values, or complex type hierarchies. Type guards make your code more type-safe by providing compile-time guarantees about the shape and properties of your data, reducing runtime errors and improving developer experience.

## 1. Built-in Type Guards

**`typeof` Type Guard:**
```typescript
function processValue(value: string | number | boolean): string {
  if (typeof value === 'string') {
    // TypeScript knows value is string here
    return value.toUpperCase();
  }
  
  if (typeof value === 'number') {
    // TypeScript knows value is number here
    return value.toFixed(2);
  }
  
  if (typeof value === 'boolean') {
    // TypeScript knows value is boolean here
    return value ? 'true' : 'false';
  }
  
  // TypeScript knows this line is unreachable
  throw new Error('Unexpected type');
}
```

**`instanceof` Type Guard:**
```typescript
class User {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
  greet() {
    return `Hello, I'm ${this.name}`;
  }
}

class Admin extends User {
  permissions: string[];
  constructor(name: string, permissions: string[]) {
    super(name);
    this.permissions = permissions;
  }
}

function handleUser(user: User | Admin | string): string {
  if (user instanceof Admin) {
    // TypeScript knows user is Admin
    return `Admin ${user.name} has ${user.permissions.length} permissions`;
  }
  
  if (user instanceof User) {
    // TypeScript knows user is User (but not Admin)
    return user.greet();
  }
  
  if (typeof user === 'string') {
    // TypeScript knows user is string
    return `Username: ${user}`;
  }
  
  throw new Error('Invalid user type');
}
```

**`in` Operator Type Guard:**
```typescript
interface Bird {
  fly(): void;
  layEggs(): void;
}

interface Fish {
  swim(): void;
  layEggs(): void;
}

function moveAnimal(animal: Bird | Fish): void {
  if ('fly' in animal) {
    // TypeScript knows animal is Bird
    animal.fly();
  } else {
    // TypeScript knows animal is Fish
    animal.swim();
  }
  
  // Both have layEggs, so this is always available
  animal.layEggs();
}
```

## 2. Custom Type Guards (User-Defined)

**Basic Custom Type Guards:**
```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

interface Admin {
  id: number;
  name: string;
  email: string;
  permissions: string[];
}

// Custom type guard function
function isAdmin(user: User | Admin): user is Admin {
  return 'permissions' in user;
}

function processUser(user: User | Admin): void {
  if (isAdmin(user)) {
    // TypeScript knows user is Admin
    console.log(`Admin ${user.name} has permissions: ${user.permissions.join(', ')}`);
  } else {
    // TypeScript knows user is User (not Admin)
    console.log(`Regular user: ${user.name}`);
  }
}
```

**Complex Validation Type Guards:**
```typescript
interface ApiUser {
  id: number;
  name: string;
  email: string;
  isActive?: boolean;
}

function isValidApiUser(data: unknown): data is ApiUser {
  return (
    typeof data === 'object' &&
    data !== null &&
    typeof (data as ApiUser).id === 'number' &&
    typeof (data as ApiUser).name === 'string' &&
    typeof (data as ApiUser).email === 'string' &&
    (typeof (data as ApiUser).isActive === 'boolean' || 
     (data as ApiUser).isActive === undefined)
  );
}

// Safe API response handling
async function fetchUser(id: number): Promise<ApiUser> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  
  if (isValidApiUser(data)) {
    return data; // TypeScript knows data is ApiUser
  }
  
  throw new Error('Invalid user data received from API');
}
```

## 3. React-Specific Type Guards

**Event Type Guards:**
```typescript
function handleFormEvent(event: React.FormEvent): void {
  event.preventDefault();
  
  const target = event.target;
  
  // Type guard for HTMLFormElement
  if (target instanceof HTMLFormElement) {
    const formData = new FormData(target);
    // Process form data safely
    console.log('Form submitted:', Object.fromEntries(formData));
  }
}

function handleInputChange(event: React.ChangeEvent<HTMLElement>): void {
  const target = event.target;
  
  if (target instanceof HTMLInputElement) {
    console.log('Input value:', target.value);
    console.log('Input type:', target.type);
  } else if (target instanceof HTMLTextAreaElement) {
    console.log('Textarea value:', target.value);
    console.log('Rows:', target.rows);
  } else if (target instanceof HTMLSelectElement) {
    console.log('Selected value:', target.value);
    console.log('Selected index:', target.selectedIndex);
  }
}
```

**Component Props Type Guards:**
```typescript
interface BaseButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary';
}

interface LinkButtonProps extends BaseButtonProps {
  href: string;
  target?: string;
}

interface ClickButtonProps extends BaseButtonProps {
  onClick: () => void;
  disabled?: boolean;
}

type ButtonProps = LinkButtonProps | ClickButtonProps;

// Type guard to check if props include href
function isLinkButton(props: ButtonProps): props is LinkButtonProps {
  return 'href' in props;
}

function Button(props: ButtonProps) {
  if (isLinkButton(props)) {
    // TypeScript knows props is LinkButtonProps
    return (
      <a 
        href={props.href} 
        target={props.target}
        className={`btn ${props.variant || 'primary'}`}
      >
        {props.children}
      </a>
    );
  }
  
  // TypeScript knows props is ClickButtonProps
  return (
    <button 
      onClick={props.onClick}
      disabled={props.disabled}
      className={`btn ${props.variant || 'primary'}`}
    >
      {props.children}
    </button>
  );
}
```

## 4. Advanced Type Guard Patterns

**Discriminated Union Type Guards:**
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
  message: string;
  retryCount: number;
}

type AsyncState = LoadingState | SuccessState | ErrorState;

// Type guards for each state
function isLoadingState(state: AsyncState): state is LoadingState {
  return state.status === 'loading';
}

function isSuccessState(state: AsyncState): state is SuccessState {
  return state.status === 'success';
}

function isErrorState(state: AsyncState): state is ErrorState {
  return state.status === 'error';
}

// Usage in component
function StateRenderer({ state }: { state: AsyncState }) {
  if (isLoadingState(state)) {
    return (
      <div>
        Loading... {state.progress ? `${state.progress}%` : ''}
      </div>
    );
  }
  
  if (isSuccessState(state)) {
    return (
      <div>
        <h3>Success!</h3>
        <p>Completed at: {new Date(state.timestamp).toLocaleString()}</p>
        <pre>{JSON.stringify(state.data, null, 2)}</pre>
      </div>
    );
  }
  
  if (isErrorState(state)) {
    return (
      <div>
        <h3>Error</h3>
        <p>{state.message}</p>
        <p>Retry count: {state.retryCount}</p>
      </div>
    );
  }
  
  // This should never be reached due to exhaustive checking
  return null;
}
```

**Array and Object Type Guards:**
```typescript
// Type guard for arrays
function isStringArray(value: unknown): value is string[] {
  return Array.isArray(value) && value.every(item => typeof item === 'string');
}

// Type guard for non-empty arrays
function isNonEmptyArray<T>(array: T[]): array is [T, ...T[]] {
  return array.length > 0;
}

// Type guard for object with specific shape
function hasProperty<T extends object, K extends string>(
  obj: T,
  prop: K
): obj is T & Record<K, unknown> {
  return prop in obj;
}

// Usage examples
function processData(data: unknown) {
  if (isStringArray(data)) {
    // TypeScript knows data is string[]
    console.log('String array:', data.join(', '));
  }
  
  if (Array.isArray(data) && isNonEmptyArray(data)) {
    // TypeScript knows data has at least one element
    console.log('First item:', data[0]);
  }
  
  if (typeof data === 'object' && data !== null && hasProperty(data, 'name')) {
    // TypeScript knows data has a 'name' property
    console.log('Name property exists');
  }
}
```

## 5. Type Guards with Generics

**Generic Type Guards:**
```typescript
function isOfType<T>(
  value: unknown,
  validator: (val: unknown) => val is T
): value is T {
  return validator(value);
}

// Validator functions
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

function isNumber(value: unknown): value is number {
  return typeof value === 'number';
}

// Generic usage
function processGenericValue<T>(
  value: unknown,
  validator: (val: unknown) => val is T
): T | null {
  if (isOfType(value, validator)) {
    return value; // TypeScript knows value is T
  }
  return null;
}

// Usage
const stringValue = processGenericValue(someUnknownValue, isString);
const numberValue = processGenericValue(someUnknownValue, isNumber);
```

**API Response Type Guards:**
```typescript
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
}

function isSuccessResponse<T>(
  response: ApiResponse<T>
): response is ApiResponse<T> & { success: true; data: T } {
  return response.success && response.data !== undefined;
}

function isErrorResponse<T>(
  response: ApiResponse<T>
): response is ApiResponse<T> & { success: false; error: string } {
  return !response.success && response.error !== undefined;
}

// Usage
async function handleApiCall<T>(apiCall: () => Promise<ApiResponse<T>>): Promise<T> {
  const response = await apiCall();
  
  if (isSuccessResponse(response)) {
    return response.data; // TypeScript knows data exists
  }
  
  if (isErrorResponse(response)) {
    throw new Error(response.error); // TypeScript knows error exists
  }
  
  throw new Error('Invalid response format');
}
```

## 6. Performance and Best Practices

**Efficient Type Guards:**
```typescript
// Good - early return pattern
function processUser(user: User | Admin | null): string {
  if (!user) return 'No user';
  if (isAdmin(user)) return `Admin: ${user.name}`;
  return `User: ${user.name}`;
}

// Good - cached type guard results
function complexProcessing(items: (string | number | object)[]): void {
  const strings = items.filter((item): item is string => typeof item === 'string');
  const numbers = items.filter((item): item is number => typeof item === 'number');
  
  // Process each type separately
  strings.forEach(str => console.log(str.toUpperCase()));
  numbers.forEach(num => console.log(num.toFixed(2)));
}
```

**Reusable Type Guard Library:**
```typescript
// Create a library of common type guards
export const TypeGuards = {
  isString: (value: unknown): value is string => typeof value === 'string',
  isNumber: (value: unknown): value is number => typeof value === 'number',
  isBoolean: (value: unknown): value is boolean => typeof value === 'boolean',
  isArray: <T>(value: unknown, guard?: (item: unknown) => item is T): value is T[] => {
    if (!Array.isArray(value)) return false;
    if (!guard) return true;
    return value.every(guard);
  },
  isObject: (value: unknown): value is Record<string, unknown> => {
    return typeof value === 'object' && value !== null && !Array.isArray(value);
  },
  hasProperty: <T extends object, K extends string>(
    obj: T,
    key: K
  ): obj is T & Record<K, unknown> => key in obj,
  isNonNullable: <T>(value: T | null | undefined): value is T => {
    return value !== null && value !== undefined;
  }
};

// Usage
function safeProcessing(data: unknown) {
  if (TypeGuards.isString(data)) {
    console.log(data.toUpperCase());
  }
  
  if (TypeGuards.isArray(data, TypeGuards.isNumber)) {
    console.log(data.reduce((sum, num) => sum + num, 0));
  }
}
```

## Interview-ready answer:
Type guards are TypeScript mechanisms that narrow down types within specific scopes, helping you safely work with union types and unknown values. Built-in guards include `typeof`, `instanceof`, and `in` operator checks. Custom type guards use functions with `value is Type` return signatures to validate data shapes. They're essential for handling API responses safely, discriminated unions, and React event handling. Type guards provide compile-time type safety while handling runtime type checking, making code more robust by ensuring TypeScript knows the exact type at each point. Common patterns include validation functions for API data, discriminated union handlers, and reusable guard libraries for complex applications.
