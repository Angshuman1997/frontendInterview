# React.memo, useMemo, useCallback

This file covers the three main React memoization hooks for preventing unnecessary re-renders and computations.
Essential for React performance optimization interviews and real-world applications.

---

## Why they matter

These hooks help optimize React performance by:

* Preventing unnecessary component re-renders
* Avoiding expensive computations on every render
* Stabilizing object/function references for child components
* Reducing CPU usage and improving responsiveness
* Breaking unnecessary dependency chains in useEffect

---

## Quick comparison

| Hook | Purpose | When to use |
|------|---------|-------------|
| `React.memo` | Memoize component | Prevent re-renders when props unchanged |
| `useMemo` | Memoize computed values | Expensive calculations or stable references |
| `useCallback` | Memoize functions | Stable callbacks for memoized children |

---

## React.memo

Wraps functional components to prevent re-renders when props are shallowly equal.

### Basic usage

```jsx
const ExpensiveComponent = React.memo(function ExpensiveComponent({ name, count }) {
  console.log('ExpensiveComponent rendered');
  return (
    <div>
      <h3>{name}</h3>
      <p>Count: {count}</p>
    </div>
  );
});

// Parent component
function Parent() {
  const [count, setCount] = useState(0);
  const [otherState, setOtherState] = useState('');
  
  return (
    <div>
      <ExpensiveComponent name="John" count={count} />
      {/* ExpensiveComponent won't re-render when otherState changes */}
      <input value={otherState} onChange={e => setOtherState(e.target.value)} />
    </div>
  );
}
```

### Custom comparison function

```jsx
const UserCard = React.memo(function UserCard({ user }) {
  return <div>{user.name} - {user.email}</div>;
}, (prevProps, nextProps) => {
  // Return true if props are equal (skip re-render)
  return prevProps.user.id === nextProps.user.id && 
         prevProps.user.name === nextProps.user.name;
});
```

### When to use React.memo

* Component receives complex props but renders the same output
* Parent re-renders frequently but child props rarely change
* Component is expensive to render (heavy computations, large lists)
* **Don't use** for components that change frequently or simple components

---

## useMemo

Memoizes expensive computations and creates stable object references.

### Expensive computations

```jsx
function ProductList({ products, filters }) {
  // Expensive filtering operation - only recalculate when dependencies change
  const filteredProducts = useMemo(() => {
    console.log('Filtering products...');
    return products.filter(product => {
      return Object.entries(filters).every(([key, value]) => 
        !value || product[key].includes(value)
      );
    });
  }, [products, filters]);

  const averagePrice = useMemo(() => {
    return filteredProducts.reduce((sum, p) => sum + p.price, 0) / filteredProducts.length;
  }, [filteredProducts]);

  return (
    <div>
      <p>Average price: ${averagePrice.toFixed(2)}</p>
      {filteredProducts.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

### Stable object references

```jsx
function SearchComponent({ onSearch }) {
  const [query, setQuery] = useState('');
  const [sortBy, setSortBy] = useState('name');

  // Stable object reference - prevents child re-renders
  const searchConfig = useMemo(() => ({
    query,
    sortBy,
    caseSensitive: false,
    maxResults: 50
  }), [query, sortBy]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <select value={sortBy} onChange={e => setSortBy(e.target.value)}>
        <option value="name">Name</option>
        <option value="date">Date</option>
      </select>
      <SearchResults config={searchConfig} />
    </div>
  );
}

const SearchResults = React.memo(function SearchResults({ config }) {
  // This component won't re-render unnecessarily due to stable config object
  useEffect(() => {
    console.log('Searching with config:', config);
    // Perform search...
  }, [config]);
  
  return <div>Search results...</div>;
});
```

### When to use useMemo

* Expensive calculations that don't need to run every render
* Creating stable references for objects/arrays passed to memoized children
* Breaking infinite loops in useEffect dependencies
* **Don't use** for simple calculations - the memoization overhead isn't worth it

---

## useCallback

Memoizes function definitions to provide stable references.

### Basic usage

```jsx
function TodoList({ todos }) {
  const [filter, setFilter] = useState('all');

  // Stable function reference - prevents TodoItem re-renders
  const handleToggle = useCallback((id) => {
    setTodos(prev => prev.map(todo => 
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  }, []); // Empty deps - function never changes

  const handleDelete = useCallback((id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  }, []);

  const filteredTodos = useMemo(() => {
    return todos.filter(todo => {
      if (filter === 'completed') return todo.completed;
      if (filter === 'active') return !todo.completed;
      return true;
    });
  }, [todos, filter]);

  return (
    <div>
      <FilterButtons filter={filter} onFilterChange={setFilter} />
      {filteredTodos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={handleToggle}
          onDelete={handleDelete}
        />
      ))}
    </div>
  );
}

const TodoItem = React.memo(function TodoItem({ todo, onToggle, onDelete }) {
  console.log('TodoItem rendered:', todo.id);
  
  return (
    <div>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      <span>{todo.text}</span>
      <button onClick={() => onDelete(todo.id)}>Delete</button>
    </div>
  );
});
```

### With dependencies

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [preferences, setPreferences] = useState({});

  // Function depends on userId - recreated when userId changes
  const fetchUserData = useCallback(async () => {
    const userData = await api.getUser(userId);
    setUser(userData);
  }, [userId]);

  // Function depends on preferences
  const savePreferences = useCallback(async (newPrefs) => {
    const updated = { ...preferences, ...newPrefs };
    await api.savePreferences(userId, updated);
    setPreferences(updated);
  }, [userId, preferences]);

  useEffect(() => {
    fetchUserData();
  }, [fetchUserData]);

  return (
    <div>
      {user && <UserDetails user={user} />}
      <PreferencesPanel 
        preferences={preferences} 
        onSave={savePreferences} 
      />
    </div>
  );
}
```

### When to use useCallback

* Passing callbacks to memoized child components
* Functions used in useEffect dependencies
* Event handlers passed to many child components
* **Don't use** if the function isn't passed to memoized children or used in hooks

---

## Common patterns and best practices

### 1. Combining all three

```jsx
function DataDashboard({ rawData, filters }) {
  // Memoize expensive data processing
  const processedData = useMemo(() => {
    return rawData
      .filter(item => matchesFilters(item, filters))
      .sort((a, b) => b.timestamp - a.timestamp)
      .map(item => ({ ...item, formatted: formatData(item) }));
  }, [rawData, filters]);

  // Stable callback for child components
  const handleItemClick = useCallback((item) => {
    analytics.track('item_clicked', { id: item.id });
    navigate(`/details/${item.id}`);
  }, [navigate]);

  // Stable config object
  const chartConfig = useMemo(() => ({
    width: 800,
    height: 400,
    theme: 'dark',
    animations: true
  }), []);

  return (
    <div>
      <Chart data={processedData} config={chartConfig} />
      <ItemList 
        items={processedData} 
        onItemClick={handleItemClick} 
      />
    </div>
  );
}

// Memoized child components
const Chart = React.memo(function Chart({ data, config }) {
  return <div>Chart with {data.length} items</div>;
});

const ItemList = React.memo(function ItemList({ items, onItemClick }) {
  return (
    <div>
      {items.map(item => (
        <ItemCard key={item.id} item={item} onClick={onItemClick} />
      ))}
    </div>
  );
});
```

### 2. Avoiding common pitfalls

```jsx
function BadExample() {
  const [count, setCount] = useState(0);

  // ❌ BAD: Creates new object every render - breaks memoization
  const config = { theme: 'dark', size: 'large' };

  // ❌ BAD: Inline function - breaks memoization
  return (
    <MemoizedChild 
      config={config} 
      onClick={() => setCount(c => c + 1)}
    />
  );
}

function GoodExample() {
  const [count, setCount] = useState(0);

  // ✅ GOOD: Stable object reference
  const config = useMemo(() => ({ 
    theme: 'dark', 
    size: 'large' 
  }), []);

  // ✅ GOOD: Stable function reference
  const handleClick = useCallback(() => {
    setCount(c => c + 1);
  }, []);

  return (
    <MemoizedChild 
      config={config} 
      onClick={handleClick}
    />
  );
}
```

---

## Performance considerations

### When NOT to use these hooks

```jsx
// ❌ Don't memoize simple calculations
const doubled = useMemo(() => count * 2, [count]); // Overhead > benefit

// ❌ Don't memoize if dependencies change often
const filtered = useMemo(() => 
  items.filter(item => item.includes(searchTerm)), 
  [items, searchTerm] // searchTerm changes on every keystroke
);

// ❌ Don't useCallback if not passed to memoized children
const handleClick = useCallback(() => console.log('clicked'), []); // Unnecessary
```

### Measuring effectiveness

```jsx
function ProfiledComponent({ data }) {
  const expensiveValue = useMemo(() => {
    console.time('expensive-calculation');
    const result = heavyComputation(data);
    console.timeEnd('expensive-calculation');
    return result;
  }, [data]);

  return <div>{expensiveValue}</div>;
}
```

---

## Interview-ready summary

**React.memo** prevents component re-renders when props are shallowly equal - use for expensive components with stable props. **useMemo** memoizes expensive computations and creates stable object references - use for heavy calculations or to prevent breaking memoization chains. **useCallback** memoizes functions to provide stable references - use when passing callbacks to memoized children or useEffect dependencies. Only optimize when you have a measured performance problem, and remember that these hooks have overhead costs.
