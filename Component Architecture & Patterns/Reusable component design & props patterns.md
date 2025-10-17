# Reusable component design & props patterns

This file covers design principles, prop patterns, and composition techniques for building reusable React components.
Essential for creating maintainable and flexible component libraries.

---

## Component Design Principles

### 1. Single Responsibility Principle
Each component should have one clear purpose and do it well.

```jsx
// ❌ Poor: Component doing too many things
function UserDashboard({ user, posts, notifications, settings }) {
  return (
    <div>
      <div className="user-profile">
        <img src={user.avatar} alt={user.name} />
        <h1>{user.name}</h1>
        <p>{user.email}</p>
      </div>
      
      <div className="posts">
        {posts.map(post => (
          <div key={post.id}>
            <h3>{post.title}</h3>
            <p>{post.content}</p>
            <span>{post.date}</span>
          </div>
        ))}
      </div>
      
      <div className="notifications">
        {notifications.map(notif => (
          <div key={notif.id} className={notif.read ? 'read' : 'unread'}>
            {notif.message}
          </div>
        ))}
      </div>
    </div>
  );
}

// ✅ Good: Separate components with single responsibilities
function UserProfile({ user }) {
  return (
    <div className="user-profile">
      <Avatar src={user.avatar} alt={user.name} size="large" />
      <UserInfo name={user.name} email={user.email} />
    </div>
  );
}

function PostList({ posts }) {
  return (
    <div className="posts">
      {posts.map(post => (
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  );
}

function NotificationList({ notifications }) {
  return (
    <div className="notifications">
      {notifications.map(notification => (
        <NotificationItem 
          key={notification.id} 
          notification={notification} 
        />
      ))}
    </div>
  );
}

function UserDashboard({ user, posts, notifications }) {
  return (
    <div className="dashboard">
      <UserProfile user={user} />
      <PostList posts={posts} />
      <NotificationList notifications={notifications} />
    </div>
  );
}
```

### 2. Composition over Inheritance
Build complex components by combining simpler ones.

```jsx
// Base components
function Card({ children, className = '', ...props }) {
  return (
    <div className={`card ${className}`} {...props}>
      {children}
    </div>
  );
}

function CardHeader({ children, className = '' }) {
  return (
    <div className={`card-header ${className}`}>
      {children}
    </div>
  );
}

function CardBody({ children, className = '' }) {
  return (
    <div className={`card-body ${className}`}>
      {children}
    </div>
  );
}

function CardFooter({ children, className = '' }) {
  return (
    <div className={`card-footer ${className}`}>
      {children}
    </div>
  );
}

// Composed components
function ProductCard({ product, onAddToCart }) {
  return (
    <Card className="product-card">
      <CardHeader>
        <img src={product.image} alt={product.name} />
      </CardHeader>
      <CardBody>
        <h3>{product.name}</h3>
        <p>{product.description}</p>
        <span className="price">${product.price}</span>
      </CardBody>
      <CardFooter>
        <Button onClick={() => onAddToCart(product)}>
          Add to Cart
        </Button>
      </CardFooter>
    </Card>
  );
}

function UserCard({ user, actions }) {
  return (
    <Card className="user-card">
      <CardHeader>
        <Avatar src={user.avatar} alt={user.name} />
        <h3>{user.name}</h3>
      </CardHeader>
      <CardBody>
        <p>{user.bio}</p>
        <div className="stats">
          <span>Posts: {user.postsCount}</span>
          <span>Followers: {user.followersCount}</span>
        </div>
      </CardBody>
      {actions && (
        <CardFooter>
          {actions}
        </CardFooter>
      )}
    </Card>
  );
}
```

---

## Props Patterns

### 1. Props Interface Design

```jsx
// ✅ Good: Clear, predictable props interface
interface ButtonProps {
  // Core functionality
  onClick?: (event: MouseEvent) => void;
  type?: 'button' | 'submit' | 'reset';
  disabled?: boolean;
  
  // Appearance
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'small' | 'medium' | 'large';
  
  // Content
  children: ReactNode;
  icon?: ReactNode;
  iconPosition?: 'left' | 'right';
  
  // States
  loading?: boolean;
  
  // Accessibility
  'aria-label'?: string;
  'aria-describedby'?: string;
  
  // HTML attributes
  className?: string;
  id?: string;
}

function Button({
  children,
  onClick,
  type = 'button',
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false,
  icon,
  iconPosition = 'left',
  className = '',
  ...htmlProps
}: ButtonProps) {
  const handleClick = (event: MouseEvent) => {
    if (!disabled && !loading && onClick) {
      onClick(event);
    }
  };

  const classes = [
    'btn',
    `btn--${variant}`,
    `btn--${size}`,
    disabled && 'btn--disabled',
    loading && 'btn--loading',
    className
  ].filter(Boolean).join(' ');

  return (
    <button
      className={classes}
      onClick={handleClick}
      disabled={disabled || loading}
      type={type}
      {...htmlProps}
    >
      {loading && <Spinner size="small" />}
      {!loading && icon && iconPosition === 'left' && icon}
      <span className="btn-text">{children}</span>
      {!loading && icon && iconPosition === 'right' && icon}
    </button>
  );
}
```

### 2. Render Props Pattern

```jsx
// Flexible data fetching component
function DataFetcher({ url, children }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    
    fetch(url)
      .then(response => response.json())
      .then(data => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(error => {
        if (!cancelled) {
          setError(error);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true;
    };
  }, [url]);

  return children({ data, loading, error });
}

// Usage examples
function UserProfile({ userId }) {
  return (
    <DataFetcher url={`/api/users/${userId}`}>
      {({ data: user, loading, error }) => {
        if (loading) return <UserSkeleton />;
        if (error) return <ErrorMessage error={error} />;
        if (!user) return <NotFound />;
        
        return (
          <div className="user-profile">
            <h1>{user.name}</h1>
            <p>{user.email}</p>
            <p>{user.bio}</p>
          </div>
        );
      }}
    </DataFetcher>
  );
}

function PostList() {
  return (
    <DataFetcher url="/api/posts">
      {({ data: posts, loading, error }) => (
        <div className="post-list">
          <h2>Recent Posts</h2>
          {loading && <div>Loading posts...</div>}
          {error && <ErrorBanner error={error} />}
          {posts && posts.map(post => (
            <PostCard key={post.id} post={post} />
          ))}
        </div>
      )}
    </DataFetcher>
  );
}
```

### 3. Compound Components Pattern

```jsx
// Accordion compound component
const AccordionContext = createContext();

function Accordion({ children, defaultOpen = null, className = '' }) {
  const [openItems, setOpenItems] = useState(
    defaultOpen ? [defaultOpen] : []
  );

  const toggleItem = (itemId) => {
    setOpenItems(prev => 
      prev.includes(itemId)
        ? prev.filter(id => id !== itemId)
        : [...prev, itemId]
    );
  };

  const isOpen = (itemId) => openItems.includes(itemId);

  return (
    <AccordionContext.Provider value={{ toggleItem, isOpen }}>
      <div className={`accordion ${className}`}>
        {children}
      </div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ children, id, className = '' }) {
  return (
    <div className={`accordion-item ${className}`} data-item-id={id}>
      {children}
    </div>
  );
}

function AccordionHeader({ children, itemId, className = '' }) {
  const { toggleItem, isOpen } = useContext(AccordionContext);
  const open = isOpen(itemId);

  return (
    <button
      className={`accordion-header ${open ? 'open' : ''} ${className}`}
      onClick={() => toggleItem(itemId)}
      aria-expanded={open}
    >
      {children}
      <ChevronIcon direction={open ? 'up' : 'down'} />
    </button>
  );
}

function AccordionContent({ children, itemId, className = '' }) {
  const { isOpen } = useContext(AccordionContext);
  const open = isOpen(itemId);

  return (
    <div 
      className={`accordion-content ${open ? 'open' : ''} ${className}`}
      aria-hidden={!open}
    >
      <div className="accordion-content-inner">
        {children}
      </div>
    </div>
  );
}

// Attach sub-components to main component
Accordion.Item = AccordionItem;
Accordion.Header = AccordionHeader;
Accordion.Content = AccordionContent;

// Usage
function FAQSection() {
  return (
    <Accordion defaultOpen="faq-1">
      <Accordion.Item id="faq-1">
        <Accordion.Header itemId="faq-1">
          What is React?
        </Accordion.Header>
        <Accordion.Content itemId="faq-1">
          React is a JavaScript library for building user interfaces...
        </Accordion.Content>
      </Accordion.Item>
      
      <Accordion.Item id="faq-2">
        <Accordion.Header itemId="faq-2">
          How do hooks work?
        </Accordion.Header>
        <Accordion.Content itemId="faq-2">
          Hooks let you use state and other React features...
        </Accordion.Content>
      </Accordion.Item>
    </Accordion>
  );
}
```

### 4. Higher-Order Component (HOC) Pattern

```jsx
// withLoading HOC
function withLoading(WrappedComponent, LoadingComponent = DefaultLoader) {
  return function WithLoadingComponent(props) {
    const { loading, ...restProps } = props;
    
    if (loading) {
      return <LoadingComponent />;
    }
    
    return <WrappedComponent {...restProps} />;
  };
}

// withError HOC
function withError(WrappedComponent, ErrorComponent = DefaultError) {
  return function WithErrorComponent(props) {
    const { error, ...restProps } = props;
    
    if (error) {
      return <ErrorComponent error={error} />;
    }
    
    return <WrappedComponent {...restProps} />;
  };
}

// Combine HOCs
function withAsyncState(WrappedComponent) {
  return withError(withLoading(WrappedComponent));
}

// Usage
const UserProfileWithStates = withAsyncState(function UserProfile({ user }) {
  return (
    <div className="user-profile">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
});

// In parent component
function App() {
  const { user, loading, error } = useUser(userId);
  
  return (
    <UserProfileWithStates 
      user={user} 
      loading={loading} 
      error={error} 
    />
  );
}
```

---

## Advanced Props Patterns

### 1. Polymorphic Components

```jsx
// Polymorphic component that can render as different elements
function Box({ as: Component = 'div', children, className = '', ...props }) {
  return (
    <Component className={`box ${className}`} {...props}>
      {children}
    </Component>
  );
}

// Usage examples
function Examples() {
  return (
    <div>
      {/* Renders as div */}
      <Box>Default box</Box>
      
      {/* Renders as section */}
      <Box as="section" className="hero">
        Hero section
      </Box>
      
      {/* Renders as button */}
      <Box as="button" onClick={() => alert('Clicked!')}>
        Clickable box
      </Box>
      
      {/* Renders as Link component */}
      <Box as={Link} to="/profile">
        Navigation box
      </Box>
    </div>
  );
}

// Advanced polymorphic with TypeScript
type PolymorphicRef<C extends React.ElementType> = 
  React.ComponentPropsWithRef<C>['ref'];

type AsProp<C extends React.ElementType> = {
  as?: C;
};

type PropsToOmit<C extends React.ElementType, P> = keyof (
  AsProp<C> & P
);

type PolymorphicComponentProp<
  C extends React.ElementType,
  Props = {}
> = React.PropsWithChildren<Props & AsProp<C>> &
  Omit<React.ComponentPropsWithoutRef<C>, PropsToOmit<C, Props>>;

type PolymorphicComponentPropWithRef<
  C extends React.ElementType,
  Props = {}
> = PolymorphicComponentProp<C, Props> & {
  ref?: PolymorphicRef<C>;
};

interface ButtonProps {
  variant?: 'primary' | 'secondary';
  size?: 'small' | 'medium' | 'large';
}

const Button = React.forwardRef<any, PolymorphicComponentPropWithRef<
  React.ElementType,
  ButtonProps
>>(({ as, variant = 'primary', size = 'medium', className = '', children, ...props }, ref) => {
  const Component = as || 'button';
  
  const classes = [
    'btn',
    `btn--${variant}`,
    `btn--${size}`,
    className
  ].filter(Boolean).join(' ');
  
  return (
    <Component ref={ref} className={classes} {...props}>
      {children}
    </Component>
  );
});
```

### 2. Slot Pattern

```jsx
// Layout component with slots
function PageLayout({ 
  header, 
  sidebar, 
  children, 
  footer, 
  className = '' 
}) {
  return (
    <div className={`page-layout ${className}`}>
      {header && (
        <header className="page-header">
          {header}
        </header>
      )}
      
      <div className="page-content">
        {sidebar && (
          <aside className="page-sidebar">
            {sidebar}
          </aside>
        )}
        
        <main className="page-main">
          {children}
        </main>
      </div>
      
      {footer && (
        <footer className="page-footer">
          {footer}
        </footer>
      )}
    </div>
  );
}

// Modal component with slots
function Modal({ 
  isOpen, 
  onClose, 
  title, 
  children, 
  actions,
  size = 'medium',
  className = ''
}) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay">
      <div className={`modal modal--${size} ${className}`}>
        <div className="modal-header">
          <h2 className="modal-title">{title}</h2>
          <button 
            className="modal-close"
            onClick={onClose}
            aria-label="Close modal"
          >
            ×
          </button>
        </div>
        
        <div className="modal-body">
          {children}
        </div>
        
        {actions && (
          <div className="modal-footer">
            {actions}
          </div>
        )}
      </div>
    </div>,
    document.body
  );
}

// Usage
function App() {
  const [showModal, setShowModal] = useState(false);

  return (
    <PageLayout
      header={
        <nav>
          <h1>My App</h1>
          <button onClick={() => setShowModal(true)}>
            Open Modal
          </button>
        </nav>
      }
      sidebar={
        <nav>
          <ul>
            <li><a href="/home">Home</a></li>
            <li><a href="/profile">Profile</a></li>
            <li><a href="/settings">Settings</a></li>
          </ul>
        </nav>
      }
      footer={
        <p>&copy; 2024 My Company</p>
      }
    >
      <h2>Main Content</h2>
      <p>This is the main content area.</p>
      
      <Modal
        isOpen={showModal}
        onClose={() => setShowModal(false)}
        title="Confirm Action"
        actions={
          <>
            <button onClick={() => setShowModal(false)}>
              Cancel
            </button>
            <button 
              className="btn-primary"
              onClick={() => {
                // Handle confirm
                setShowModal(false);
              }}
            >
              Confirm
            </button>
          </>
        }
      >
        <p>Are you sure you want to perform this action?</p>
      </Modal>
    </PageLayout>
  );
}
```

### 3. Control Props Pattern

```jsx
// Controlled/Uncontrolled input component
function useControlledState(controlledValue, defaultValue, onChange) {
  const isControlled = controlledValue !== undefined;
  const [internalValue, setInternalValue] = useState(defaultValue);
  
  const value = isControlled ? controlledValue : internalValue;
  
  const setValue = useCallback((newValue) => {
    if (!isControlled) {
      setInternalValue(newValue);
    }
    onChange?.(newValue);
  }, [isControlled, onChange]);
  
  return [value, setValue, isControlled];
}

function SearchInput({
  value: controlledValue,
  defaultValue = '',
  onChange,
  onSearch,
  placeholder = 'Search...',
  debounceMs = 300,
  className = ''
}) {
  const [value, setValue, isControlled] = useControlledState(
    controlledValue, 
    defaultValue, 
    onChange
  );
  
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  // Debounce the search
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, debounceMs);
    
    return () => clearTimeout(timer);
  }, [value, debounceMs]);
  
  // Call onSearch when debounced value changes
  useEffect(() => {
    if (debouncedValue !== defaultValue) {
      onSearch?.(debouncedValue);
    }
  }, [debouncedValue, onSearch, defaultValue]);
  
  const handleChange = (event) => {
    setValue(event.target.value);
  };
  
  const handleSubmit = (event) => {
    event.preventDefault();
    onSearch?.(value);
  };
  
  return (
    <form onSubmit={handleSubmit} className={`search-form ${className}`}>
      <input
        type="text"
        value={value}
        onChange={handleChange}
        placeholder={placeholder}
        className="search-input"
      />
      <button type="submit" className="search-button">
        🔍
      </button>
    </form>
  );
}

// Usage examples
function UncontrolledExample() {
  return (
    <SearchInput
      defaultValue=""
      onSearch={(query) => console.log('Searching for:', query)}
      placeholder="Uncontrolled search..."
    />
  );
}

function ControlledExample() {
  const [searchQuery, setSearchQuery] = useState('');
  
  return (
    <SearchInput
      value={searchQuery}
      onChange={setSearchQuery}
      onSearch={(query) => {
        console.log('Searching for:', query);
        // Perform search
      }}
      placeholder="Controlled search..."
    />
  );
}
```

---

## Component Composition Patterns

### 1. Provider Pattern

```jsx
// Theme provider with context
const ThemeContext = createContext();

function ThemeProvider({ children, defaultTheme = 'light' }) {
  const [theme, setTheme] = useState(defaultTheme);
  const [customColors, setCustomColors] = useState({});
  
  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };
  
  const updateColor = (colorName, colorValue) => {
    setCustomColors(prev => ({
      ...prev,
      [colorName]: colorValue
    }));
  };
  
  const themeValue = {
    theme,
    setTheme,
    toggleTheme,
    customColors,
    updateColor,
    colors: {
      ...defaultColors[theme],
      ...customColors
    }
  };
  
  return (
    <ThemeContext.Provider value={themeValue}>
      <div className={`app-theme app-theme--${theme}`}>
        {children}
      </div>
    </ThemeContext.Provider>
  );
}

function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}

// Components that use theme
function ThemedButton({ children, variant = 'primary', ...props }) {
  const { colors } = useTheme();
  
  const style = {
    backgroundColor: colors[`${variant}Background`],
    color: colors[`${variant}Text`],
    borderColor: colors[`${variant}Border`]
  };
  
  return (
    <button className={`btn btn--${variant}`} style={style} {...props}>
      {children}
    </button>
  );
}

function ThemeControls() {
  const { theme, toggleTheme, updateColor } = useTheme();
  
  return (
    <div className="theme-controls">
      <button onClick={toggleTheme}>
        Switch to {theme === 'light' ? 'dark' : 'light'} theme
      </button>
      
      <div className="color-controls">
        <label>
          Primary Color:
          <input
            type="color"
            onChange={(e) => updateColor('primaryBackground', e.target.value)}
          />
        </label>
      </div>
    </div>
  );
}
```

### 2. Factory Pattern

```jsx
// Form field factory
function createFormField(FieldComponent) {
  return function FormField({
    name,
    label,
    error,
    required = false,
    helpText,
    className = '',
    ...fieldProps
  }) {
    const fieldId = `field-${name}`;
    const errorId = error ? `${fieldId}-error` : undefined;
    const helpId = helpText ? `${fieldId}-help` : undefined;
    
    return (
      <div className={`form-field ${error ? 'has-error' : ''} ${className}`}>
        {label && (
          <label htmlFor={fieldId} className="form-label">
            {label}
            {required && <span className="required" aria-label="required">*</span>}
          </label>
        )}
        
        <FieldComponent
          id={fieldId}
          name={name}
          aria-invalid={!!error}
          aria-describedby={[errorId, helpId].filter(Boolean).join(' ') || undefined}
          className="form-control"
          {...fieldProps}
        />
        
        {helpText && (
          <div id={helpId} className="form-help">
            {helpText}
          </div>
        )}
        
        {error && (
          <div id={errorId} className="form-error" role="alert">
            {error}
          </div>
        )}
      </div>
    );
  };
}

// Create specific field components
const TextField = createFormField((props) => (
  <input type="text" {...props} />
));

const PasswordField = createFormField((props) => (
  <input type="password" {...props} />
));

const TextAreaField = createFormField((props) => (
  <textarea {...props} />
));

const SelectField = createFormField(({ options, ...props }) => (
  <select {...props}>
    {options.map(option => (
      <option key={option.value} value={option.value}>
        {option.label}
      </option>
    ))}
  </select>
));

// Usage
function LoginForm() {
  const [values, setValues] = useState({ email: '', password: '' });
  const [errors, setErrors] = useState({});
  
  return (
    <form>
      <TextField
        name="email"
        label="Email"
        type="email"
        value={values.email}
        onChange={(e) => setValues(prev => ({ ...prev, email: e.target.value }))}
        error={errors.email}
        required
      />
      
      <PasswordField
        name="password"
        label="Password"
        value={values.password}
        onChange={(e) => setValues(prev => ({ ...prev, password: e.target.value }))}
        error={errors.password}
        helpText="Password must be at least 8 characters"
        required
      />
      
      <button type="submit">Login</button>
    </form>
  );
}
```

---

## Best Practices

### 1. Component API Design
* Keep props interface minimal and focused
* Use TypeScript for better prop validation
* Provide sensible defaults
* Support both controlled and uncontrolled modes when appropriate
* Forward refs when the component wraps a DOM element

### 2. Performance Optimization
```jsx
// Memoize expensive computations
const ExpensiveComponent = memo(function ExpensiveComponent({ data, config }) {
  const processedData = useMemo(() => {
    return expensiveDataProcessing(data, config);
  }, [data, config]);
  
  return <div>{processedData}</div>;
});

// Split props to optimize re-renders
function OptimizedComponent({ staticProps, dynamicProps }) {
  return (
    <div>
      <StaticSection {...staticProps} />
      <DynamicSection {...dynamicProps} />
    </div>
  );
}

const StaticSection = memo(function StaticSection(props) {
  // This won't re-render when dynamicProps change
  return <div>{props.content}</div>;
});
```

### 3. Error Handling
```jsx
// Prop validation and error boundaries
function SafeComponent({ data, onError }) {
  if (!data) {
    onError?.(new Error('Data is required'));
    return <div>No data provided</div>;
  }
  
  try {
    return <div>{processData(data)}</div>;
  } catch (error) {
    onError?.(error);
    return <div>Error processing data</div>;
  }
}

// Use error boundaries for graceful degradation
function ComponentWithErrorBoundary(props) {
  return (
    <ErrorBoundary fallback={<ComponentErrorFallback />}>
      <MainComponent {...props} />
    </ErrorBoundary>
  );
}
```

---

## Interview-ready summary

**Reusable component design** focuses on creating flexible, maintainable components through clear props interfaces, composition patterns, and separation of concerns. Key patterns include **render props** for flexible data sharing, **compound components** for related component groups, **HOCs** for cross-cutting concerns, and **polymorphic components** for element flexibility. Props patterns include controlled/uncontrolled state, prop spreading, and prop validation. Design principles emphasize single responsibility, composition over inheritance, and predictable APIs. Advanced patterns include provider pattern for context sharing, factory pattern for similar components, and slot pattern for layout flexibility. Performance optimization through memoization, prop splitting, and error boundaries ensures robust, scalable component libraries.

