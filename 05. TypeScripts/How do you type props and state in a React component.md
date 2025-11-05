# How do you type props and state in a React component

Typing props and state in React components with TypeScript ensures type safety, better developer experience, and helps catch errors at compile time. TypeScript provides several approaches for typing React components, from simple interfaces to advanced generic patterns. Proper typing makes components more maintainable, self-documenting, and provides excellent IntelliSense support in IDEs.

## 1. Basic Props Typing

**Functional Component Props:**
```typescript
interface UserCardProps {
  name: string;
  email: string;
  age?: number; // Optional prop
  isActive: boolean;
}

function UserCard({ name, email, age, isActive }: UserCardProps) {
  return (
    <div className={`user-card ${isActive ? 'active' : 'inactive'}`}>
      <h3>{name}</h3>
      <p>{email}</p>
      {age && <p>Age: {age}</p>}
    </div>
  );
}

// Usage with type checking
<UserCard 
  name="John Doe" 
  email="john@example.com" 
  isActive={true} 
/>
```

**Alternative Type Definition Approaches:**
```typescript
// Using type alias instead of interface
type UserCardProps = {
  name: string;
  email: string;
  age?: number;
  isActive: boolean;
};

// Inline typing (for simple components)
function SimpleButton({ label, onClick }: { label: string; onClick: () => void }) {
  return <button onClick={onClick}>{label}</button>;
}

// Using React.FC (not recommended in modern React)
const UserCard: React.FC<UserCardProps> = ({ name, email, age, isActive }) => {
  return (
    <div className={`user-card ${isActive ? 'active' : 'inactive'}`}>
      <h3>{name}</h3>
      <p>{email}</p>
      {age && <p>Age: {age}</p>}
    </div>
  );
};
```

## 2. Advanced Props Patterns

**Props with Event Handlers:**
```typescript
interface ButtonProps {
  children: React.ReactNode;
  onClick: (event: React.MouseEvent<HTMLButtonElement>) => void;
  onHover?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  disabled?: boolean;
  variant?: 'primary' | 'secondary' | 'danger';
}

function Button({ children, onClick, onHover, disabled = false, variant = 'primary' }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant}`}
      onClick={onClick}
      onMouseEnter={onHover}
      disabled={disabled}
    >
      {children}
    </button>
  );
}

// Usage
<Button 
  onClick={(e) => console.log('Clicked!', e.currentTarget)}
  onHover={(e) => console.log('Hovered!')}
  variant="primary"
>
  Click me
</Button>
```

**Generic Props:**
```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  onItemClick?: (item: T) => void;
  keyExtractor?: (item: T) => string | number;
}

function List<T>({ items, renderItem, onItemClick, keyExtractor }: ListProps<T>) {
  return (
    <div className="list">
      {items.map((item, index) => (
        <div 
          key={keyExtractor ? keyExtractor(item) : index}
          onClick={() => onItemClick?.(item)}
          className="list-item"
        >
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

<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  onItemClick={(user) => console.log('Selected user:', user)}
  keyExtractor={(user) => user.id}
/>

<List<string>
  items={['Apple', 'Banana', 'Cherry']}
  renderItem={(fruit) => <span>{fruit}</span>}
/>
```

**Conditional Props (Discriminated Unions):**
```typescript
interface BaseModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
}

interface ConfirmModalProps extends BaseModalProps {
  type: 'confirm';
  onConfirm: () => void;
  confirmText?: string;
  cancelText?: string;
}

interface InfoModalProps extends BaseModalProps {
  type: 'info';
  message: string;
}

interface FormModalProps extends BaseModalProps {
  type: 'form';
  onSubmit: (data: any) => void;
  children: React.ReactNode;
}

type ModalProps = ConfirmModalProps | InfoModalProps | FormModalProps;

function Modal(props: ModalProps) {
  if (!props.isOpen) return null;

  return (
    <div className="modal-overlay">
      <div className="modal">
        <h2>{props.title}</h2>
        
        {props.type === 'confirm' && (
          <div>
            <p>Are you sure you want to continue?</p>
            <button onClick={props.onConfirm}>
              {props.confirmText || 'Confirm'}
            </button>
            <button onClick={props.onClose}>
              {props.cancelText || 'Cancel'}
            </button>
          </div>
        )}
        
        {props.type === 'info' && (
          <div>
            <p>{props.message}</p>
            <button onClick={props.onClose}>Close</button>
          </div>
        )}
        
        {props.type === 'form' && (
          <div>
            {props.children}
          </div>
        )}
      </div>
    </div>
  );
}
```

## 3. Typing State with useState

**Basic State Typing:**
```typescript
function UserProfile() {
  // TypeScript infers string type
  const [name, setName] = useState('');
  
  // Explicit typing for complex types
  const [user, setUser] = useState<User | null>(null);
  
  // Union types
  const [status, setStatus] = useState<'idle' | 'loading' | 'success' | 'error'>('idle');
  
  // Array state
  const [items, setItems] = useState<string[]>([]);
  
  // Object state with interface
  const [formData, setFormData] = useState<{
    name: string;
    email: string;
    age: number;
  }>({
    name: '',
    email: '',
    age: 0
  });

  return (
    <div>
      <input 
        value={name} 
        onChange={(e) => setName(e.target.value)} 
      />
      <p>Status: {status}</p>
      <p>Items count: {items.length}</p>
    </div>
  );
}
```

**Complex State Patterns:**
```typescript
interface AppState {
  user: User | null;
  isAuthenticated: boolean;
  theme: 'light' | 'dark';
  notifications: Notification[];
}

interface Notification {
  id: string;
  type: 'info' | 'success' | 'warning' | 'error';
  message: string;
  timestamp: Date;
}

function App() {
  const [state, setState] = useState<AppState>({
    user: null,
    isAuthenticated: false,
    theme: 'light',
    notifications: []
  });

  // Type-safe state updates
  const addNotification = (notification: Omit<Notification, 'id' | 'timestamp'>) => {
    setState(prevState => ({
      ...prevState,
      notifications: [
        ...prevState.notifications,
        {
          ...notification,
          id: crypto.randomUUID(),
          timestamp: new Date()
        }
      ]
    }));
  };

  const setUser = (user: User) => {
    setState(prevState => ({
      ...prevState,
      user,
      isAuthenticated: true
    }));
  };

  const toggleTheme = () => {
    setState(prevState => ({
      ...prevState,
      theme: prevState.theme === 'light' ? 'dark' : 'light'
    }));
  };

  return (
    <div className={`app theme-${state.theme}`}>
      {state.isAuthenticated ? (
        <p>Welcome, {state.user?.name}</p>
      ) : (
        <p>Please log in</p>
      )}
      
      <button onClick={toggleTheme}>
        Switch to {state.theme === 'light' ? 'dark' : 'light'} mode
      </button>
      
      {state.notifications.map(notification => (
        <div key={notification.id} className={`notification ${notification.type}`}>
          {notification.message}
        </div>
      ))}
    </div>
  );
}
```

## 4. Custom Hooks with TypeScript

**Typed Custom Hooks:**
```typescript
interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

function useCounter(initialValue: number = 0): UseCounterReturn {
  const [count, setCount] = useState<number>(initialValue);

  const increment = useCallback(() => {
    setCount(prev => prev + 1);
  }, []);

  const decrement = useCallback(() => {
    setCount(prev => prev - 1);
  }, []);

  const reset = useCallback(() => {
    setCount(initialValue);
  }, [initialValue]);

  return { count, increment, decrement, reset };
}

// Generic hook
interface UseAsyncState<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
}

function useAsync<T>(
  asyncFunction: () => Promise<T>,
  immediate: boolean = true
): UseAsyncState<T> & {
  execute: () => Promise<void>;
} {
  const [state, setState] = useState<UseAsyncState<T>>({
    data: null,
    loading: false,
    error: null
  });

  const execute = useCallback(async () => {
    setState(prev => ({ ...prev, loading: true, error: null }));
    
    try {
      const data = await asyncFunction();
      setState({ data, loading: false, error: null });
    } catch (error) {
      setState({ data: null, loading: false, error: error instanceof Error ? error.message : 'Unknown error' });
    }
  }, [asyncFunction]);

  useEffect(() => {
    if (immediate) {
      execute();
    }
  }, [execute, immediate]);

  return { ...state, execute };
}

// Usage
function UserProfile({ userId }: { userId: number }) {
  const { data: user, loading, error } = useAsync<User>(
    () => fetch(`/api/users/${userId}`).then(res => res.json()),
    true
  );

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>No user found</div>;

  return <div>User: {user.name}</div>;
}
```

## 5. Class Component Typing (Legacy)

**Class Component with Props and State:**
```typescript
interface CounterProps {
  initialCount?: number;
  step?: number;
  onCountChange?: (count: number) => void;
}

interface CounterState {
  count: number;
  history: number[];
}

class Counter extends React.Component<CounterProps, CounterState> {
  constructor(props: CounterProps) {
    super(props);
    this.state = {
      count: props.initialCount || 0,
      history: [props.initialCount || 0]
    };
  }

  increment = (): void => {
    this.setState(
      prevState => ({
        count: prevState.count + (this.props.step || 1),
        history: [...prevState.history, prevState.count + (this.props.step || 1)]
      }),
      () => {
        // Callback after state update
        this.props.onCountChange?.(this.state.count);
      }
    );
  };

  decrement = (): void => {
    this.setState(prevState => ({
      count: prevState.count - (this.props.step || 1),
      history: [...prevState.history, prevState.count - (this.props.step || 1)]
    }));
  };

  render(): React.ReactNode {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={this.increment}>+</button>
        <button onClick={this.decrement}>-</button>
        <p>History: {this.state.history.join(', ')}</p>
      </div>
    );
  }
}
```

## 6. React Built-in Types

**Common React Types:**
```typescript
interface ComponentProps {
  // Basic types
  children?: React.ReactNode;
  className?: string;
  style?: React.CSSProperties;
  
  // Event handlers
  onClick?: React.MouseEventHandler<HTMLButtonElement>;
  onChange?: React.ChangeEventHandler<HTMLInputElement>;
  onSubmit?: React.FormEventHandler<HTMLFormElement>;
  
  // Refs
  ref?: React.Ref<HTMLDivElement>;
  
  // HTML attributes (extending HTML element props)
  id?: string;
  'data-testid'?: string;
}

// Extending HTML element props
interface CustomButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
  loading?: boolean;
}

function CustomButton({ variant = 'primary', loading, children, ...buttonProps }: CustomButtonProps) {
  return (
    <button 
      {...buttonProps}
      className={`btn btn-${variant} ${buttonProps.className || ''}`}
      disabled={loading || buttonProps.disabled}
    >
      {loading ? 'Loading...' : children}
    </button>
  );
}

// Forward ref typing
const FancyInput = React.forwardRef<
  HTMLInputElement,
  React.InputHTMLAttributes<HTMLInputElement> & { label?: string }
>(({ label, ...props }, ref) => (
  <div>
    {label && <label>{label}</label>}
    <input ref={ref} {...props} />
  </div>
));
```

## 7. Best Practices and Common Patterns

**Props Validation and Default Values:**
```typescript
interface UserCardProps {
  user: User;
  showEmail?: boolean;
  showAge?: boolean;
  onEdit?: () => void;
  className?: string;
}

// Using default parameters
function UserCard({ 
  user, 
  showEmail = true, 
  showAge = false, 
  onEdit,
  className = ''
}: UserCardProps) {
  return (
    <div className={`user-card ${className}`}>
      <h3>{user.name}</h3>
      {showEmail && <p>{user.email}</p>}
      {showAge && user.age && <p>Age: {user.age}</p>}
      {onEdit && <button onClick={onEdit}>Edit</button>}
    </div>
  );
}

// Props with children patterns
interface LayoutProps {
  children: React.ReactNode;
  sidebar?: React.ReactNode;
  header?: React.ReactNode;
  footer?: React.ReactNode;
}

function Layout({ children, sidebar, header, footer }: LayoutProps) {
  return (
    <div className="layout">
      {header && <header>{header}</header>}
      <main className="main-content">
        {sidebar && <aside>{sidebar}</aside>}
        <section>{children}</section>
      </main>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}
```

**Type-safe Form Handling:**
```typescript
interface FormData {
  name: string;
  email: string;
  age: number;
  preferences: {
    newsletter: boolean;
    notifications: boolean;
  };
}

interface FormProps {
  initialData?: Partial<FormData>;
  onSubmit: (data: FormData) => void;
  onCancel?: () => void;
}

function UserForm({ initialData, onSubmit, onCancel }: FormProps) {
  const [formData, setFormData] = useState<FormData>({
    name: initialData?.name || '',
    email: initialData?.email || '',
    age: initialData?.age || 0,
    preferences: {
      newsletter: initialData?.preferences?.newsletter || false,
      notifications: initialData?.preferences?.notifications || false,
    }
  });

  const handleInputChange = (field: keyof Pick<FormData, 'name' | 'email'>) => 
    (e: React.ChangeEvent<HTMLInputElement>) => {
      setFormData(prev => ({
        ...prev,
        [field]: e.target.value
      }));
    };

  const handleNumberChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setFormData(prev => ({
      ...prev,
      age: parseInt(e.target.value) || 0
    }));
  };

  const handlePreferenceChange = (pref: keyof FormData['preferences']) => 
    (e: React.ChangeEvent<HTMLInputElement>) => {
      setFormData(prev => ({
        ...prev,
        preferences: {
          ...prev.preferences,
          [pref]: e.target.checked
        }
      }));
    };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSubmit(formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input 
        type="text" 
        value={formData.name} 
        onChange={handleInputChange('name')} 
        placeholder="Name"
      />
      <input 
        type="email" 
        value={formData.email} 
        onChange={handleInputChange('email')} 
        placeholder="Email"
      />
      <input 
        type="number" 
        value={formData.age} 
        onChange={handleNumberChange} 
        placeholder="Age"
      />
      <label>
        <input 
          type="checkbox" 
          checked={formData.preferences.newsletter} 
          onChange={handlePreferenceChange('newsletter')} 
        />
        Subscribe to newsletter
      </label>
      <label>
        <input 
          type="checkbox" 
          checked={formData.preferences.notifications} 
          onChange={handlePreferenceChange('notifications')} 
        />
        Enable notifications
      </label>
      <button type="submit">Submit</button>
      {onCancel && <button type="button" onClick={onCancel}>Cancel</button>}
    </form>
  );
}
```

## Interview-ready answer:
To type React components in TypeScript, define interfaces for props and use generic types for state. For props, create interfaces that describe the expected shape: `interface ComponentProps { name: string; onClick: () => void; }`. Use `useState<Type>()` for state typing, often with union types or complex objects. Event handlers should use React's built-in types like `React.MouseEvent<HTMLButtonElement>`. For advanced patterns, use generic components `<T>`, discriminated unions for conditional props, and extend HTML attributes when needed. Custom hooks return typed objects, and class components use `React.Component<Props, State>`. Always prefer explicit typing over `any` for better type safety and developer experience.
