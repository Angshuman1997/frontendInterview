# Explain HOC vs Render Props vs Custom Hooks

This file compares three React patterns for sharing stateful logic between components.
Essential for understanding React's evolution and choosing the right pattern for code reuse.

---

## Quick comparison

| Pattern | React Era | Pros | Cons | Modern Usage |
|---------|-----------|------|------|--------------|
| **HOC** | Class components | Composition, prop injection | Wrapper hell, naming conflicts | Legacy code, libraries |
| **Render Props** | Class/early hooks | Flexible, explicit | Nesting, complexity | Specialized cases |
| **Custom Hooks** | Modern React | Clean, composable | Function components only | Preferred approach |

---

## Higher-Order Components (HOC)

### What it is
A function that takes a component and returns a new component with enhanced functionality.

### Basic example

```jsx
// withAuth.js - HOC for authentication
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const { user, loading } = useAuth();
    
    if (loading) {
      return <div>Loading...</div>;
    }
    
    if (!user) {
      return <div>Please log in to access this page.</div>;
    }
    
    // Inject auth-related props
    return <WrappedComponent {...props} user={user} />;
  };
}

// Usage
function Dashboard({ user }) {
  return <div>Welcome, {user.name}!</div>;
}

const AuthenticatedDashboard = withAuth(Dashboard);

// In your app
function App() {
  return <AuthenticatedDashboard />;
}
```

### Advanced HOC patterns

```jsx
// withDataFetching.js - Generic data fetching HOC
function withDataFetching(url, propName = 'data') {
  return function(WrappedComponent) {
    return function DataFetchingComponent(props) {
      const [data, setData] = useState(null);
      const [loading, setLoading] = useState(true);
      const [error, setError] = useState(null);

      useEffect(() => {
        fetchData();
      }, []);

      const fetchData = async () => {
        try {
          setLoading(true);
          const response = await fetch(url);
          const result = await response.json();
          setData(result);
        } catch (err) {
          setError(err);
        } finally {
          setLoading(false);
        }
      };

      const enhancedProps = {
        ...props,
        [propName]: data,
        [`${propName}Loading`]: loading,
        [`${propName}Error`]: error,
        [`refresh${propName}`]: fetchData,
      };

      return <WrappedComponent {...enhancedProps} />;
    };
  };
}

// Usage
const UserList = ({ users, usersLoading, usersError, refreshUsers }) => {
  if (usersLoading) return <div>Loading users...</div>;
  if (usersError) return <div>Error: {usersError.message}</div>;
  
  return (
    <div>
      <button onClick={refreshUsers}>Refresh</button>
      {users?.map(user => <div key={user.id}>{user.name}</div>)}
    </div>
  );
};

const EnhancedUserList = withDataFetching('/api/users', 'users')(UserList);
```

### Multiple HOCs composition

```jsx
// Composing multiple HOCs
function withLogging(WrappedComponent) {
  return function LoggingComponent(props) {
    useEffect(() => {
      console.log(`${WrappedComponent.name} mounted`);
      return () => console.log(`${WrappedComponent.name} unmounted`);
    }, []);
    
    return <WrappedComponent {...props} />;
  };
}

function withTheme(WrappedComponent) {
  return function ThemedComponent(props) {
    const theme = useContext(ThemeContext);
    return <WrappedComponent {...props} theme={theme} />;
  };
}

// Composition helper
function compose(...fns) {
  return (arg) => fns.reduceRight((acc, fn) => fn(acc), arg);
}

// Usage
const EnhancedComponent = compose(
  withAuth,
  withLogging,
  withTheme,
  withDataFetching('/api/data', 'data')
)(MyComponent);
```

---

## Render Props

### What it is
A technique where a component accepts a function as a prop and calls it to render content.

### Basic example

```jsx
// DataProvider.jsx - Render props for data fetching
class DataProvider extends React.Component {
  state = {
    data: null,
    loading: true,
    error: null,
  };

  async componentDidMount() {
    try {
      const response = await fetch(this.props.url);
      const data = await response.json();
      this.setState({ data, loading: false });
    } catch (error) {
      this.setState({ error, loading: false });
    }
  }

  render() {
    return this.props.children(this.state);
  }
}

// Usage
function App() {
  return (
    <DataProvider url="/api/users">
      {({ data: users, loading, error }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error: {error.message}</div>;
        
        return (
          <div>
            {users?.map(user => (
              <div key={user.id}>{user.name}</div>
            ))}
          </div>
        );
      }}
    </DataProvider>
  );
}
```

### Modern render props with hooks

```jsx
// MouseTracker.jsx - Mouse position tracking
function MouseTracker({ children }) {
  const [mousePosition, setMousePosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handleMouseMove = (event) => {
      setMousePosition({
        x: event.clientX,
        y: event.clientY,
      });
    };

    document.addEventListener('mousemove', handleMouseMove);
    return () => document.removeEventListener('mousemove', handleMouseMove);
  }, []);

  return children(mousePosition);
}

// Usage
function App() {
  return (
    <div>
      <MouseTracker>
        {({ x, y }) => (
          <div>
            Mouse position: {x}, {y}
            <div
              style={{
                position: 'absolute',
                left: x,
                top: y,
                width: 10,
                height: 10,
                backgroundColor: 'red',
                borderRadius: '50%',
                pointerEvents: 'none',
              }}
            />
          </div>
        )}
      </MouseTracker>
    </div>
  );
}
```

### Complex render props example

```jsx
// FormProvider.jsx - Form state management with render props
function FormProvider({ initialValues, validationSchema, onSubmit, children }) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = (name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
    
    // Clear error when user starts typing
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: null }));
    }
  };

  const handleBlur = (name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
    validateField(name, values[name]);
  };

  const validateField = (name, value) => {
    if (validationSchema[name]) {
      const error = validationSchema[name](value);
      setErrors(prev => ({ ...prev, [name]: error }));
      return !error;
    }
    return true;
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    // Validate all fields
    const isValid = Object.keys(validationSchema).every(name =>
      validateField(name, values[name])
    );
    
    if (!isValid) return;
    
    setIsSubmitting(true);
    try {
      await onSubmit(values);
    } catch (error) {
      console.error('Submit error:', error);
    } finally {
      setIsSubmitting(false);
    }
  };

  return children({
    values,
    errors,
    touched,
    isSubmitting,
    handleChange,
    handleBlur,
    handleSubmit,
  });
}

// Usage
function LoginForm() {
  const validationSchema = {
    email: (value) => !value ? 'Email is required' : null,
    password: (value) => value.length < 6 ? 'Password must be at least 6 characters' : null,
  };

  const handleSubmit = async (values) => {
    await api.login(values);
  };

  return (
    <FormProvider
      initialValues={{ email: '', password: '' }}
      validationSchema={validationSchema}
      onSubmit={handleSubmit}
    >
      {({ values, errors, handleChange, handleBlur, handleSubmit, isSubmitting }) => (
        <form onSubmit={handleSubmit}>
          <input
            type="email"
            value={values.email}
            onChange={(e) => handleChange('email', e.target.value)}
            onBlur={() => handleBlur('email')}
            placeholder="Email"
          />
          {errors.email && <span className="error">{errors.email}</span>}
          
          <input
            type="password"
            value={values.password}
            onChange={(e) => handleChange('password', e.target.value)}
            onBlur={() => handleBlur('password')}
            placeholder="Password"
          />
          {errors.password && <span className="error">{errors.password}</span>}
          
          <button type="submit" disabled={isSubmitting}>
            {isSubmitting ? 'Logging in...' : 'Login'}
          </button>
        </form>
      )}
    </FormProvider>
  );
}
```

---

## Custom Hooks

### What it is
A JavaScript function that starts with "use" and can call other hooks. The modern React way to share stateful logic.

### Basic example

```jsx
// useAuth.js - Authentication hook
function useAuth() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const checkAuth = async () => {
      try {
        const token = localStorage.getItem('token');
        if (token) {
          const userData = await api.getCurrentUser();
          setUser(userData);
        }
      } catch (error) {
        localStorage.removeItem('token');
      } finally {
        setLoading(false);
      }
    };

    checkAuth();
  }, []);

  const login = async (credentials) => {
    const { user, token } = await api.login(credentials);
    localStorage.setItem('token', token);
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return { user, loading, login, logout };
}

// Usage
function Dashboard() {
  const { user, loading, logout } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <div>Please log in</div>;

  return (
    <div>
      <h1>Welcome, {user.name}!</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### Advanced custom hooks

```jsx
// useDataFetching.js - Generic data fetching hook
function useDataFetching(url, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchData = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);
      
      const response = await fetch(url, options);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const result = await response.json();
      setData(result);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, [url, JSON.stringify(options)]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  const refetch = useCallback(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch };
}

// useLocalStorage.js - Local storage hook
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  const setValue = useCallback((value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(`Error setting localStorage key "${key}":`, error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue];
}

// useForm.js - Form management hook
function useForm(initialValues, validationSchema = {}) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});

  const handleChange = useCallback((name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
    
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: null }));
    }
  }, [errors]);

  const handleBlur = useCallback((name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
    
    if (validationSchema[name]) {
      const error = validationSchema[name](values[name]);
      setErrors(prev => ({ ...prev, [name]: error }));
    }
  }, [validationSchema, values]);

  const validate = useCallback(() => {
    const newErrors = {};
    let isValid = true;

    Object.keys(validationSchema).forEach(name => {
      const error = validationSchema[name](values[name]);
      if (error) {
        newErrors[name] = error;
        isValid = false;
      }
    });

    setErrors(newErrors);
    return isValid;
  }, [validationSchema, values]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  }, [initialValues]);

  return {
    values,
    errors,
    touched,
    handleChange,
    handleBlur,
    validate,
    reset,
  };
}

// Usage
function UserProfile() {
  const { data: user, loading, error, refetch } = useDataFetching('/api/user');
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  
  const { values, errors, handleChange, handleBlur, validate } = useForm(
    { name: user?.name || '', email: user?.email || '' },
    {
      name: (value) => value.length < 2 ? 'Name must be at least 2 characters' : null,
      email: (value) => !/\S+@\S+\.\S+/.test(value) ? 'Invalid email' : null,
    }
  );

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div className={`profile ${theme}`}>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
      
      <form>
        <input
          value={values.name}
          onChange={(e) => handleChange('name', e.target.value)}
          onBlur={() => handleBlur('name')}
          placeholder="Name"
        />
        {errors.name && <span>{errors.name}</span>}
        
        <input
          value={values.email}
          onChange={(e) => handleChange('email', e.target.value)}
          onBlur={() => handleBlur('email')}
          placeholder="Email"
        />
        {errors.email && <span>{errors.email}</span>}
      </form>
    </div>
  );
}
```

---

## Pattern comparison in practice

### Same functionality, different patterns

```jsx
// 1. HOC approach
const withCounter = (WrappedComponent) => {
  return function CounterHOC(props) {
    const [count, setCount] = useState(0);
    const increment = () => setCount(c => c + 1);
    const decrement = () => setCount(c => c - 1);
    
    return (
      <WrappedComponent 
        {...props} 
        count={count} 
        increment={increment} 
        decrement={decrement} 
      />
    );
  };
};

const CounterWithHOC = withCounter(({ count, increment, decrement }) => (
  <div>
    <p>Count: {count}</p>
    <button onClick={increment}>+</button>
    <button onClick={decrement}>-</button>
  </div>
));

// 2. Render Props approach
function CounterRenderProps({ children }) {
  const [count, setCount] = useState(0);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  
  return children({ count, increment, decrement });
}

const CounterWithRenderProps = () => (
  <CounterRenderProps>
    {({ count, increment, decrement }) => (
      <div>
        <p>Count: {count}</p>
        <button onClick={increment}>+</button>
        <button onClick={decrement}>-</button>
      </div>
    )}
  </CounterRenderProps>
);

// 3. Custom Hook approach
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  const increment = useCallback(() => setCount(c => c + 1), []);
  const decrement = useCallback(() => setCount(c => c - 1), []);
  const reset = useCallback(() => setCount(initialValue), [initialValue]);
  
  return { count, increment, decrement, reset };
}

const CounterWithHook = () => {
  const { count, increment, decrement } = useCounter();
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
};
```

---

## When to use each pattern

### Use HOCs when:
* Working with legacy class components
* Building component libraries that need to work with any component
* Complex prop manipulation or injection
* Wrapping third-party components

### Use Render Props when:
* Need maximum flexibility in how data is rendered
* Complex component composition scenarios
* Building headless UI components
* When the "what" to render depends on the data

### Use Custom Hooks when:
* Modern React applications (React 16.8+)
* Sharing stateful logic between function components
* Complex state management that can be abstracted
* Building reusable business logic

---

## Best practices

### HOCs
* Use `forwardRef` to pass refs through
* Hoist non-React statics with `hoist-non-react-statics`
* Use display names for better debugging
* Don't mutate the original component

### Render Props
* Use consistent naming for render prop functions
* Consider performance implications of inline functions
* Provide good TypeScript definitions

### Custom Hooks
* Follow the "use" naming convention
* Extract related state and effects together
* Return objects for multiple values (easier to extend)
* Use `useCallback` and `useMemo` appropriately

---

## Interview-ready summary

**HOCs** wrap components to inject props and functionality - good for cross-cutting concerns but can cause wrapper hell. **Render Props** pass functions as props for flexible rendering - excellent for complex UI composition but can lead to nesting. **Custom Hooks** extract stateful logic into reusable functions - the modern React approach, cleaner and more composable. Custom hooks are now preferred for new React applications, while HOCs remain useful for legacy code and component libraries. Each pattern solves the same problem of logic reuse but with different trade-offs in complexity and flexibility.
