# What are utility types like Partial, Pick, Omit, and Readonly

TypeScript provides built-in utility types that help transform and manipulate existing types. These utilities save you from writing repetitive type definitions and make your code more maintainable. The most commonly used utilities are `Partial`, `Pick`, `Omit`, and `Readonly`, which allow you to create new types by modifying properties of existing types in various ways.

## 1. Partial<T> - Make All Properties Optional

`Partial<T>` creates a new type where all properties of `T` become optional.

**Basic Usage:**
```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type PartialUser = Partial<User>;
// Equivalent to:
// {
//   id?: number;
//   name?: string;
//   email?: string;
//   age?: number;
// }
```

**Real-World Example - Update Functions:**
```typescript
function updateUser(id: number, updates: Partial<User>): User {
  const currentUser = getUserById(id);
  return { ...currentUser, ...updates };
}

// Usage - can update any combination of properties
updateUser(1, { name: "John Doe" });
updateUser(1, { email: "john@example.com", age: 30 });
updateUser(1, {}); // Valid - no updates
```

**React Example - Component Props:**
```typescript
interface UserFormProps {
  user: User;
  onUpdate: (updates: Partial<User>) => void;
}

function UserForm({ user, onUpdate }: UserFormProps) {
  const handleNameChange = (name: string) => {
    onUpdate({ name }); // Only updating name field
  };

  const handleEmailChange = (email: string) => {
    onUpdate({ email }); // Only updating email field
  };

  return (
    <form>
      <input 
        value={user.name} 
        onChange={(e) => handleNameChange(e.target.value)} 
      />
      <input 
        value={user.email} 
        onChange={(e) => handleEmailChange(e.target.value)} 
      />
    </form>
  );
}
```

## 2. Pick<T, K> - Select Specific Properties

`Pick<T, K>` creates a new type by selecting only specific properties from `T`.

**Basic Usage:**
```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
}

type PublicUser = Pick<User, 'id' | 'name' | 'email'>;
// Equivalent to:
// {
//   id: number;
//   name: string;
//   email: string;
// }
```

**API Response Example:**
```typescript
// Different API endpoints return different subsets of user data
type UserSummary = Pick<User, 'id' | 'name'>;
type UserProfile = Pick<User, 'id' | 'name' | 'email' | 'createdAt'>;

interface ApiResponse<T> {
  data: T;
  status: 'success' | 'error';
}

async function getUserSummary(): Promise<ApiResponse<UserSummary>> {
  // Returns only id and name
  const response = await fetch('/api/users/summary');
  return response.json();
}

async function getUserProfile(id: number): Promise<ApiResponse<UserProfile>> {
  // Returns id, name, email, and createdAt
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}
```

**React Props Example:**
```typescript
// Reuse existing type definitions for component props
interface UserCardProps extends Pick<User, 'name' | 'email'> {
  onClick: () => void;
}

function UserCard({ name, email, onClick }: UserCardProps) {
  return (
    <div onClick={onClick}>
      <h3>{name}</h3>
      <p>{email}</p>
    </div>
  );
}
```

## 3. Omit<T, K> - Exclude Specific Properties

`Omit<T, K>` creates a new type by excluding specific properties from `T`.

**Basic Usage:**
```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

type CreateUserInput = Omit<User, 'id' | 'createdAt'>;
// Equivalent to:
// {
//   name: string;
//   email: string;
//   password: string;
// }
```

**Form Handling Example:**
```typescript
interface CreateUserFormProps {
  onSubmit: (user: CreateUserInput) => void;
  loading?: boolean;
}

function CreateUserForm({ onSubmit, loading }: CreateUserFormProps) {
  const [formData, setFormData] = useState<CreateUserInput>({
    name: '',
    email: '',
    password: ''
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSubmit(formData); // TypeScript ensures we don't include id or createdAt
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        placeholder="Name"
        value={formData.name}
        onChange={(e) => setFormData(prev => ({ ...prev, name: e.target.value }))}
      />
      <input
        placeholder="Email"
        value={formData.email}
        onChange={(e) => setFormData(prev => ({ ...prev, email: e.target.value }))}
      />
      <input
        placeholder="Password"
        type="password"
        value={formData.password}
        onChange={(e) => setFormData(prev => ({ ...prev, password: e.target.value }))}
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create User'}
      </button>
    </form>
  );
}
```

**Database Models:**
```typescript
// Database entity
interface UserEntity {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
}

// Types for different operations
type CreateUserData = Omit<UserEntity, 'id' | 'createdAt' | 'updatedAt'>;
type UpdateUserData = Partial<Omit<UserEntity, 'id' | 'createdAt' | 'updatedAt'>>;
type PublicUserData = Omit<UserEntity, 'password'>;
```

## 4. Readonly<T> - Make All Properties Readonly

`Readonly<T>` creates a new type where all properties of `T` become readonly (immutable).

**Basic Usage:**
```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type ReadonlyUser = Readonly<User>;
// Equivalent to:
// {
//   readonly id: number;
//   readonly name: string;
//   readonly email: string;
// }
```

**Immutable Data Example:**
```typescript
function processUser(user: Readonly<User>): User {
  // user.name = "Changed"; // ❌ Error: Cannot assign to 'name' because it is readonly
  
  // Must return a new object for changes
  return {
    ...user,
    name: user.name.toUpperCase()
  };
}

// State management with immutability
interface AppState {
  users: Readonly<User>[];
  loading: boolean;
}

function userReducer(
  state: AppState, 
  action: { type: 'ADD_USER'; payload: User }
): AppState {
  switch (action.type) {
    case 'ADD_USER':
      return {
        ...state,
        users: [...state.users, action.payload] // Create new array
      };
    default:
      return state;
  }
}
```

**React Props with Readonly:**
```typescript
interface UserDisplayProps {
  user: Readonly<User>;
  users: ReadonlyArray<User>; // Built-in readonly array type
}

function UserDisplay({ user, users }: UserDisplayProps) {
  // user.email = "new@email.com"; // ❌ Error
  // users.push(newUser);          // ❌ Error
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>Total users: {users.length}</p>
    </div>
  );
}
```

## 5. Advanced Combinations and Patterns

**Combining Utility Types:**
```typescript
// Create a type for updating specific fields
type UpdatableUserFields = Pick<User, 'name' | 'email'>;
type UserUpdateInput = Partial<UpdatableUserFields>;

// Make picked properties readonly
type ReadonlyUserProfile = Readonly<Pick<User, 'id' | 'name' | 'email'>>;

// Omit sensitive data and make readonly
type SafeUserData = Readonly<Omit<User, 'password'>>;
```

**Custom Utility Type Patterns:**
```typescript
// Make specific properties optional
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type UserWithOptionalAge = PartialBy<User, 'age'>;
// { id: number; name: string; email: string; age?: number }

// Make specific properties required
type RequireBy<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

interface OptionalUser {
  id?: number;
  name?: string;
  email?: string;
}

type UserWithRequiredEmail = RequireBy<OptionalUser, 'email'>;
// { id?: number; name?: string; email: string }
```

## 6. React-Specific Patterns

**Component Prop Inheritance:**
```typescript
interface BaseComponentProps {
  className?: string;
  children?: React.ReactNode;
  onClick?: () => void;
}

// Button inherits base props but makes onClick required
interface ButtonProps extends Omit<BaseComponentProps, 'onClick'> {
  onClick: () => void;
  variant?: 'primary' | 'secondary';
}

// Card only needs certain base props
interface CardProps extends Pick<BaseComponentProps, 'className' | 'children'> {
  title: string;
}
```

**Form State Management:**
```typescript
interface UserForm {
  name: string;
  email: string;
  age: string; // Form values are typically strings
  terms: boolean;
}

// Different states of the form
type FormData = UserForm;
type FormErrors = Partial<Record<keyof UserForm, string>>;
type TouchedFields = Partial<Record<keyof UserForm, boolean>>;

interface UseFormReturn {
  values: FormData;
  errors: FormErrors;
  touched: TouchedFields;
  setFieldValue: <K extends keyof FormData>(field: K, value: FormData[K]) => void;
  setFieldTouched: (field: keyof FormData, touched: boolean) => void;
}
```

## 7. Real-World API Integration

**Type-Safe API Responses:**
```typescript
interface ApiUser {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
  avatar: string;
  role: 'admin' | 'user';
  lastLoginAt: string;
  createdAt: string;
}

// Different endpoints return different data shapes
type UserListItem = Pick<ApiUser, 'id' | 'firstName' | 'lastName' | 'avatar'>;
type UserProfile = Omit<ApiUser, 'lastLoginAt'>;
type CreateUserRequest = Omit<ApiUser, 'id' | 'avatar' | 'lastLoginAt' | 'createdAt'>;
type UpdateUserRequest = Partial<Pick<ApiUser, 'firstName' | 'lastName' | 'email' | 'role'>>;

// API service methods
class UserService {
  static async getUsers(): Promise<UserListItem[]> {
    const response = await fetch('/api/users');
    return response.json();
  }

  static async getUserProfile(id: number): Promise<UserProfile> {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
  }

  static async createUser(userData: CreateUserRequest): Promise<ApiUser> {
    const response = await fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify(userData)
    });
    return response.json();
  }

  static async updateUser(id: number, updates: UpdateUserRequest): Promise<UserProfile> {
    const response = await fetch(`/api/users/${id}`, {
      method: 'PATCH',
      body: JSON.stringify(updates)
    });
    return response.json();
  }
}
```

## Interview-ready answer:
TypeScript utility types transform existing types to create new ones. `Partial<T>` makes all properties optional (useful for update operations), `Pick<T, K>` selects specific properties (great for API responses with different data shapes), `Omit<T, K>` excludes properties (perfect for form inputs that exclude auto-generated fields), and `Readonly<T>` makes properties immutable (enforces immutability patterns). These utilities promote code reuse, type safety, and maintainability by letting you derive new types from existing ones rather than duplicating type definitions. They're essential for React development, especially for component props, form handling, and API integration.
