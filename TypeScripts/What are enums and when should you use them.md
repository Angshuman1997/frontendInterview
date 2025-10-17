# What are enums and when should you use them

Enums (enumerations) in TypeScript allow you to define named constants that represent a set of related values. They provide a way to create meaningful names for numeric or string values, making code more readable and maintainable. Enums are particularly useful for representing states, status codes, configuration options, or any finite set of related constants that won't change during runtime.

## 1. Basic Enum Types

**Numeric Enums (Default):**
```typescript
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right  // 3
}

// Usage
function move(direction: Direction): string {
  switch (direction) {
    case Direction.Up:
      return 'Moving up';
    case Direction.Down:
      return 'Moving down';
    case Direction.Left:
      return 'Moving left';
    case Direction.Right:
      return 'Moving right';
    default:
      return 'Invalid direction';
  }
}

console.log(move(Direction.Up));    // "Moving up"
console.log(Direction.Up);          // 0
console.log(Direction[0]);          // "Up" (reverse mapping)
```

**String Enums:**
```typescript
enum Status {
  Pending = 'PENDING',
  Approved = 'APPROVED',
  Rejected = 'REJECTED',
  Cancelled = 'CANCELLED'
}

// API response handling
interface ApiResponse {
  status: Status;
  data?: any;
  message?: string;
}

function handleResponse(response: ApiResponse): void {
  switch (response.status) {
    case Status.Pending:
      console.log('Request is still pending...');
      break;
    case Status.Approved:
      console.log('Request approved:', response.data);
      break;
    case Status.Rejected:
      console.log('Request rejected:', response.message);
      break;
    case Status.Cancelled:
      console.log('Request was cancelled');
      break;
  }
}
```

**Mixed Enums (Not Recommended):**
```typescript
enum MixedEnum {
  No = 0,
  Yes = 'YES',
  Unknown = 1
}
// Generally avoid mixed enums as they can be confusing
```

## 2. Advanced Enum Patterns

**Computed and Constant Members:**
```typescript
enum FileAccess {
  // Constant members
  None,
  Read = 1 << 1,      // 2
  Write = 1 << 2,     // 4
  ReadWrite = Read | Write,  // 6
  
  // Computed member
  G = "123".length    // 3
}

// Bitwise operations with enums
function hasAccess(userAccess: FileAccess, requiredAccess: FileAccess): boolean {
  return (userAccess & requiredAccess) === requiredAccess;
}

console.log(hasAccess(FileAccess.ReadWrite, FileAccess.Read));  // true
console.log(hasAccess(FileAccess.Read, FileAccess.Write));     // false
```

**Ambient Enums (Declaration Only):**
```typescript
declare enum ExternalEnum {
  Value1,
  Value2,
  Value3
}

// Used when you need to reference enums from external libraries
// without including their implementation
```

## 3. React-Specific Enum Usage

**Component State Management:**
```typescript
enum LoadingState {
  Idle = 'IDLE',
  Loading = 'LOADING',
  Success = 'SUCCESS',
  Error = 'ERROR'
}

interface AppState {
  loadingState: LoadingState;
  data?: any;
  error?: string;
}

function DataComponent() {
  const [state, setState] = useState<AppState>({
    loadingState: LoadingState.Idle
  });

  const fetchData = async () => {
    setState(prev => ({ ...prev, loadingState: LoadingState.Loading }));
    
    try {
      const response = await fetch('/api/data');
      const data = await response.json();
      
      setState({
        loadingState: LoadingState.Success,
        data,
        error: undefined
      });
    } catch (error) {
      setState({
        loadingState: LoadingState.Error,
        data: undefined,
        error: error instanceof Error ? error.message : 'Unknown error'
      });
    }
  };

  const renderContent = () => {
    switch (state.loadingState) {
      case LoadingState.Idle:
        return <button onClick={fetchData}>Load Data</button>;
      case LoadingState.Loading:
        return <div>Loading...</div>;
      case LoadingState.Success:
        return <div>Data: {JSON.stringify(state.data)}</div>;
      case LoadingState.Error:
        return <div>Error: {state.error}</div>;
      default:
        return null;
    }
  };

  return <div>{renderContent()}</div>;
}
```

**Theme and Styling:**
```typescript
enum Theme {
  Light = 'light',
  Dark = 'dark',
  Auto = 'auto'
}

enum ButtonVariant {
  Primary = 'primary',
  Secondary = 'secondary',
  Success = 'success',
  Danger = 'danger',
  Warning = 'warning'
}

interface ButtonProps {
  variant?: ButtonVariant;
  children: React.ReactNode;
  onClick?: () => void;
}

function Button({ variant = ButtonVariant.Primary, children, onClick }: ButtonProps) {
  return (
    <button 
      className={`btn btn-${variant}`}
      onClick={onClick}
    >
      {children}
    </button>
  );
}

// Usage
<Button variant={ButtonVariant.Danger} onClick={handleDelete}>
  Delete
</Button>
```

## 4. API Integration with Enums

**HTTP Status Codes:**
```typescript
enum HttpStatusCode {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  Forbidden = 403,
  NotFound = 404,
  InternalServerError = 500
}

enum HttpMethod {
  GET = 'GET',
  POST = 'POST',
  PUT = 'PUT',
  DELETE = 'DELETE',
  PATCH = 'PATCH'
}

interface ApiConfig {
  method: HttpMethod;
  endpoint: string;
  expectedStatus?: HttpStatusCode;
}

class ApiClient {
  async request<T>(config: ApiConfig, data?: any): Promise<T> {
    const response = await fetch(config.endpoint, {
      method: config.method,
      headers: {
        'Content-Type': 'application/json',
      },
      body: data ? JSON.stringify(data) : undefined
    });

    if (response.status !== (config.expectedStatus || HttpStatusCode.OK)) {
      throw new Error(`API Error: ${response.status}`);
    }

    return response.json();
  }
}

// Usage
const client = new ApiClient();

const users = await client.request<User[]>({
  method: HttpMethod.GET,
  endpoint: '/api/users',
  expectedStatus: HttpStatusCode.OK
});
```

**Error Codes and Types:**
```typescript
enum ErrorType {
  Validation = 'VALIDATION_ERROR',
  Authentication = 'AUTH_ERROR',
  Authorization = 'AUTHZ_ERROR',
  NotFound = 'NOT_FOUND',
  ServerError = 'SERVER_ERROR',
  NetworkError = 'NETWORK_ERROR'
}

enum ErrorCode {
  // Validation errors
  InvalidEmail = 'INVALID_EMAIL',
  RequiredField = 'REQUIRED_FIELD',
  PasswordTooWeak = 'PASSWORD_TOO_WEAK',
  
  // Auth errors
  InvalidCredentials = 'INVALID_CREDENTIALS',
  TokenExpired = 'TOKEN_EXPIRED',
  
  // Server errors
  DatabaseError = 'DATABASE_ERROR',
  ServiceUnavailable = 'SERVICE_UNAVAILABLE'
}

interface AppError {
  type: ErrorType;
  code: ErrorCode;
  message: string;
  field?: string;
}

function handleError(error: AppError): string {
  switch (error.type) {
    case ErrorType.Validation:
      return `Validation Error: ${error.message}${error.field ? ` (Field: ${error.field})` : ''}`;
    case ErrorType.Authentication:
      return 'Please log in to continue';
    case ErrorType.Authorization:
      return 'You do not have permission to perform this action';
    case ErrorType.NotFound:
      return 'The requested resource was not found';
    case ErrorType.ServerError:
      return 'A server error occurred. Please try again later';
    case ErrorType.NetworkError:
      return 'Network error. Please check your connection';
    default:
      return 'An unknown error occurred';
  }
}
```

## 5. When to Use Enums vs Alternatives

**Enums vs Union Types:**
```typescript
// Using enum
enum Color {
  Red = 'red',
  Green = 'green',
  Blue = 'blue'
}

// Using union type (often preferred for simple cases)
type Color = 'red' | 'green' | 'blue';

// Enum advantages:
// - Reverse mapping for numeric enums
// - Can add methods and computed properties
// - Better for complex state machines
// - Clearer intent for "this is a finite set"

// Union type advantages:
// - More lightweight (no runtime overhead)
// - Better tree-shaking
// - More TypeScript-idiomatic
// - Easier to extend with template literals
```

**Enums vs Constants:**
```typescript
// Using enum
enum ApiEndpoints {
  Users = '/api/users',
  Posts = '/api/posts',
  Comments = '/api/comments'
}

// Using const object (alternative approach)
const API_ENDPOINTS = {
  Users: '/api/users',
  Posts: '/api/posts',
  Comments: '/api/comments'
} as const;

type ApiEndpoint = typeof API_ENDPOINTS[keyof typeof API_ENDPOINTS];

// Const object advantages:
// - No runtime overhead
// - Better tree-shaking
// - More flexible

// Enum advantages:
// - Built-in TypeScript feature
// - Reverse mapping
// - Clear semantic meaning
```

## 6. Enum Best Practices

**Naming Conventions:**
```typescript
// Use PascalCase for enum names
enum UserRole {
  Admin = 'ADMIN',
  Moderator = 'MODERATOR',
  User = 'USER'
}

// Use PascalCase for enum members
enum OrderStatus {
  PendingPayment = 'PENDING_PAYMENT',
  PaymentConfirmed = 'PAYMENT_CONFIRMED',
  InProgress = 'IN_PROGRESS',
  Completed = 'COMPLETED',
  Cancelled = 'CANCELLED'
}
```

**Const Enums for Performance:**
```typescript
const enum Direction {
  Up = 'UP',
  Down = 'DOWN',
  Left = 'LEFT',
  Right = 'RIGHT'
}

// Compiled output will inline the values instead of creating an object
function move(direction: Direction) {
  // This gets compiled to: if (direction === 'UP')
  if (direction === Direction.Up) {
    return 'Moving up';
  }
}

// Note: Const enums don't support reverse mapping
```

**Enum Utilities:**
```typescript
enum Priority {
  Low = 1,
  Medium = 2,
  High = 3,
  Critical = 4
}

// Utility functions for enums
namespace PriorityUtils {
  export function getAllValues(): Priority[] {
    return Object.values(Priority).filter(value => typeof value === 'number') as Priority[];
  }

  export function getAllKeys(): string[] {
    return Object.keys(Priority).filter(key => isNaN(Number(key)));
  }

  export function toString(priority: Priority): string {
    return Priority[priority];
  }

  export function fromString(value: string): Priority | undefined {
    return Priority[value as keyof typeof Priority];
  }

  export function isValid(value: any): value is Priority {
    return Object.values(Priority).includes(value);
  }
}

// Usage
const allPriorities = PriorityUtils.getAllValues(); // [1, 2, 3, 4]
const priorityName = PriorityUtils.toString(Priority.High); // "High"
const isValidPriority = PriorityUtils.isValid(5); // false
```

## 7. Common Pitfalls and Solutions

**Reverse Mapping Issues:**
```typescript
enum Status {
  Active = 1,
  Inactive = 2
}

// Be careful with reverse mapping
console.log(Status.Active);    // 1
console.log(Status[1]);        // "Active"
console.log(Status["1"]);      // "Active" (string key also works)

// This can lead to unexpected behavior:
const keys = Object.keys(Status); // ["1", "2", "Active", "Inactive"]

// Solution: Use string enums to avoid reverse mapping
enum StatusString {
  Active = 'ACTIVE',
  Inactive = 'INACTIVE'
}

const keysString = Object.keys(StatusString); // ["Active", "Inactive"]
```

**Type Safety with Enums:**
```typescript
enum Color {
  Red,
  Green,
  Blue
}

// Potential issue: TypeScript allows any number
function setColor(color: Color) {
  // This works but is dangerous
}

setColor(999); // No error, but probably not intended

// Solution: Use string enums for better safety
enum SafeColor {
  Red = 'RED',
  Green = 'GREEN',
  Blue = 'BLUE'
}

function setSafeColor(color: SafeColor) {
  // Now only valid enum values are allowed
}

// setSafeColor('INVALID'); // Error!
```

## Interview-ready answer:
Enums in TypeScript define named constants representing related values, making code more readable and maintainable. Use numeric enums for simple sequences (auto-incrementing from 0), string enums for better debugging and serialization, and const enums for performance. They're ideal for states (loading, success, error), configuration options, status codes, and finite sets of related constants. Prefer string enums over numeric ones for better type safety and avoid reverse mapping issues. For simple cases, consider union types instead (`'red' | 'green' | 'blue'`) as they're lighter weight. Enums excel when you need a clear, finite set of related constants that won't change at runtime.
