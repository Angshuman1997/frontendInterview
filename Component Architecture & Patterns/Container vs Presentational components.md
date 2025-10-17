# Container vs Presentational components

This file explains the separation of concerns pattern that divides components into data-handling containers and UI-focused presentational components.
A foundational concept for building maintainable React applications.

---

## Core concepts

### Presentational Components (UI Components)
* Focus on how things look
* Receive data and callbacks via props
* Rarely have their own state (mostly UI state)
* Don't specify how data is loaded or mutated
* Are written as functional components

### Container Components (Smart Components)
* Focus on how things work
* Provide data and behavior to presentational components
* Call Redux actions and provide data from Redux store
* Are often stateful and serve as data sources
* Are usually generated using higher order components

---

## Basic example

### Presentational Component

```jsx
// UserList.jsx - Presentational component
function UserList({ users, onUserClick, loading, error }) {
  if (loading) return <div>Loading users...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!users.length) return <div>No users found</div>;

  return (
    <div className="user-list">
      <h2>Users</h2>
      {users.map(user => (
        <div 
          key={user.id} 
          className="user-item"
          onClick={() => onUserClick(user.id)}
        >
          <img src={user.avatar} alt={user.name} />
          <div>
            <h3>{user.name}</h3>
            <p>{user.email}</p>
          </div>
        </div>
      ))}
    </div>
  );
}

// UserCard.jsx - Another presentational component
function UserCard({ user, onEdit, onDelete }) {
  return (
    <div className="user-card">
      <img src={user.avatar} alt={user.name} />
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      <p>{user.role}</p>
      <div className="actions">
        <button onClick={() => onEdit(user.id)}>Edit</button>
        <button onClick={() => onDelete(user.id)}>Delete</button>
      </div>
    </div>
  );
}
```

### Container Component

```jsx
// UserListContainer.jsx - Container component
import { useState, useEffect } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { fetchUsers, deleteUser } from '../store/userActions';
import UserList from './UserList';

function UserListContainer() {
  const dispatch = useDispatch();
  const { users, loading, error } = useSelector(state => state.users);
  const [selectedUser, setSelectedUser] = useState(null);

  useEffect(() => {
    dispatch(fetchUsers());
  }, [dispatch]);

  const handleUserClick = (userId) => {
    setSelectedUser(userId);
    // Navigate to user detail page
    navigate(`/users/${userId}`);
  };

  const handleDeleteUser = async (userId) => {
    if (window.confirm('Are you sure?')) {
      await dispatch(deleteUser(userId));
    }
  };

  return (
    <UserList
      users={users}
      loading={loading}
      error={error}
      onUserClick={handleUserClick}
      onUserDelete={handleDeleteUser}
      selectedUser={selectedUser}
    />
  );
}
```

---

## Modern patterns with hooks

### Custom hooks as containers

```jsx
// useUserData.js - Custom hook acting as container logic
function useUserData() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchUsers = async () => {
    try {
      setLoading(true);
      const response = await api.getUsers();
      setUsers(response.data);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  };

  const addUser = async (userData) => {
    try {
      const response = await api.createUser(userData);
      setUsers(prev => [...prev, response.data]);
      return response.data;
    } catch (err) {
      setError(err);
      throw err;
    }
  };

  const deleteUser = async (userId) => {
    try {
      await api.deleteUser(userId);
      setUsers(prev => prev.filter(user => user.id !== userId));
    } catch (err) {
      setError(err);
      throw err;
    }
  };

  useEffect(() => {
    fetchUsers();
  }, []);

  return {
    users,
    loading,
    error,
    addUser,
    deleteUser,
    refetch: fetchUsers,
  };
}

// Component using the custom hook
function UserManagementPage() {
  const { users, loading, error, addUser, deleteUser } = useUserData();

  return (
    <div>
      <UserList 
        users={users}
        loading={loading}
        error={error}
        onUserDelete={deleteUser}
      />
      <UserForm onSubmit={addUser} />
    </div>
  );
}
```

### Compound component pattern

```jsx
// Modern approach: Compound components with context
const UserListContext = createContext();

function UserListProvider({ children }) {
  const { users, loading, error, deleteUser } = useUserData();
  
  return (
    <UserListContext.Provider value={{ users, loading, error, deleteUser }}>
      {children}
    </UserListContext.Provider>
  );
}

function UserListHeader() {
  const { users, loading } = useContext(UserListContext);
  
  return (
    <header>
      <h1>Users {!loading && `(${users.length})`}</h1>
    </header>
  );
}

function UserListItems() {
  const { users, loading, error } = useContext(UserListContext);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <div className="user-items">
      {users.map(user => (
        <UserItem key={user.id} user={user} />
      ))}
    </div>
  );
}

function UserItem({ user }) {
  const { deleteUser } = useContext(UserListContext);
  
  return (
    <div className="user-item">
      <UserCard user={user} onDelete={deleteUser} />
    </div>
  );
}

// Usage
function App() {
  return (
    <UserListProvider>
      <UserListHeader />
      <UserListItems />
    </UserListProvider>
  );
}
```

---

## Complex example: E-commerce product

### Presentational Components

```jsx
// ProductCard.jsx
function ProductCard({ 
  product, 
  onAddToCart, 
  onToggleWishlist, 
  isInWishlist, 
  isAddingToCart 
}) {
  return (
    <div className="product-card">
      <div className="product-image">
        <img src={product.image} alt={product.name} />
        <button 
          className={`wishlist-btn ${isInWishlist ? 'active' : ''}`}
          onClick={() => onToggleWishlist(product.id)}
        >
          ❤️
        </button>
      </div>
      
      <div className="product-info">
        <h3>{product.name}</h3>
        <p className="price">${product.price}</p>
        <p className="description">{product.description}</p>
        
        <button 
          className="add-to-cart-btn"
          onClick={() => onAddToCart(product.id)}
          disabled={isAddingToCart || !product.inStock}
        >
          {isAddingToCart ? 'Adding...' : 'Add to Cart'}
        </button>
        
        {!product.inStock && (
          <p className="out-of-stock">Out of Stock</p>
        )}
      </div>
    </div>
  );
}

// ProductGrid.jsx
function ProductGrid({ 
  products, 
  loading, 
  error, 
  onAddToCart, 
  onToggleWishlist, 
  wishlistItems,
  addingToCartIds 
}) {
  if (loading) return <ProductGridSkeleton />;
  if (error) return <ErrorMessage error={error} />;
  if (!products.length) return <EmptyState message="No products found" />;

  return (
    <div className="product-grid">
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onAddToCart={onAddToCart}
          onToggleWishlist={onToggleWishlist}
          isInWishlist={wishlistItems.includes(product.id)}
          isAddingToCart={addingToCartIds.includes(product.id)}
        />
      ))}
    </div>
  );
}
```

### Container Component

```jsx
// ProductGridContainer.jsx
function ProductGridContainer({ category, filters }) {
  const dispatch = useDispatch();
  const { 
    products, 
    loading, 
    error 
  } = useSelector(state => state.products);
  
  const { 
    items: wishlistItems 
  } = useSelector(state => state.wishlist);
  
  const [addingToCartIds, setAddingToCartIds] = useState([]);

  useEffect(() => {
    dispatch(fetchProducts({ category, filters }));
  }, [dispatch, category, filters]);

  const handleAddToCart = async (productId) => {
    setAddingToCartIds(prev => [...prev, productId]);
    
    try {
      await dispatch(addToCart(productId));
      // Show success notification
      dispatch(showNotification('Product added to cart!'));
    } catch (error) {
      dispatch(showNotification('Failed to add product', 'error'));
    } finally {
      setAddingToCartIds(prev => prev.filter(id => id !== productId));
    }
  };

  const handleToggleWishlist = (productId) => {
    const isInWishlist = wishlistItems.includes(productId);
    
    if (isInWishlist) {
      dispatch(removeFromWishlist(productId));
    } else {
      dispatch(addToWishlist(productId));
    }
  };

  return (
    <ProductGrid
      products={products}
      loading={loading}
      error={error}
      onAddToCart={handleAddToCart}
      onToggleWishlist={handleToggleWishlist}
      wishlistItems={wishlistItems}
      addingToCartIds={addingToCartIds}
    />
  );
}
```

---

## Benefits and trade-offs

### Benefits

```jsx
// ✅ Better separation of concerns
// UI logic separated from business logic
function UserList({ users, onUserClick }) { /* Pure UI */ }
function UserListContainer() { /* Data & business logic */ }

// ✅ Easier testing
// Test presentational components with simple props
const wrapper = shallow(<UserList users={mockUsers} onUserClick={jest.fn()} />);

// Test container logic separately
const mockDispatch = jest.fn();
const wrapper = mount(<UserListContainer />);

// ✅ Better reusability
// Same presentational component, different containers
<AdminUserList /> // Different data source
<PublicUserList /> // Different data source
// Both use same UserList presentational component
```

### Modern considerations

```jsx
// ⚠️ Modern React: Hooks reduce the need for strict separation
function UserList() {
  // Data fetching and UI in same component
  const { users, loading, error } = useUsers();
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <div>
      {users.map(user => <UserCard key={user.id} user={user} />)}
    </div>
  );
}

// ✅ But separation still valuable for complex scenarios
function useUserManagement() {
  // Complex business logic
  const [users, setUsers] = useState([]);
  const [filters, setFilters] = useState({});
  const [pagination, setPagination] = useState({ page: 1, limit: 10 });
  
  // Complex data fetching, filtering, sorting logic
  // ...
  
  return { users, filters, pagination, /* ... */ };
}

function UserManagementPage() {
  const userManagement = useUserManagement();
  
  return <UserList {...userManagement} />;
}
```

---

## When to use each pattern

### Use Container/Presentational when:

* Complex business logic that needs testing in isolation
* Multiple data sources for similar UI
* Large teams where UI and data developers work separately
* Component library development
* Redux/complex state management

### Use unified components when:

* Simple components with straightforward logic
* Rapid prototyping
* Small teams
* One-off components unlikely to be reused

---

## Testing strategies

### Testing presentational components

```jsx
// UserCard.test.jsx
describe('UserCard', () => {
  const mockUser = {
    id: 1,
    name: 'John Doe',
    email: 'john@example.com',
    avatar: 'avatar.jpg'
  };

  test('renders user information correctly', () => {
    const onEdit = jest.fn();
    const onDelete = jest.fn();
    
    render(
      <UserCard 
        user={mockUser} 
        onEdit={onEdit} 
        onDelete={onDelete} 
      />
    );
    
    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  test('calls onEdit when edit button clicked', () => {
    const onEdit = jest.fn();
    const onDelete = jest.fn();
    
    render(
      <UserCard 
        user={mockUser} 
        onEdit={onEdit} 
        onDelete={onDelete} 
      />
    );
    
    fireEvent.click(screen.getByText('Edit'));
    expect(onEdit).toHaveBeenCalledWith(1);
  });
});
```

### Testing container components

```jsx
// UserListContainer.test.jsx
describe('UserListContainer', () => {
  test('fetches users on mount', () => {
    const mockDispatch = jest.fn();
    useDispatch.mockReturnValue(mockDispatch);
    
    render(<UserListContainer />);
    
    expect(mockDispatch).toHaveBeenCalledWith(fetchUsers());
  });
  
  test('handles user deletion', async () => {
    const mockDispatch = jest.fn();
    useDispatch.mockReturnValue(mockDispatch);
    
    // Mock window.confirm
    window.confirm = jest.fn(() => true);
    
    render(<UserListContainer />);
    
    // Simulate delete action
    // ...
    
    expect(mockDispatch).toHaveBeenCalledWith(deleteUser(1));
  });
});
```

---

## Best practices

* Keep presentational components pure and predictable
* Pass primitive values when possible, avoid complex objects
* Use TypeScript to define clear prop interfaces
* Implement proper loading and error states
* Use custom hooks to extract and reuse container logic
* Consider compound components for complex UI patterns
* Test presentational and container logic separately
* Document component APIs clearly

---

## Interview-ready summary

Container components handle data fetching, state management, and business logic, while presentational components focus purely on rendering UI based on props. This separation improves testability, reusability, and maintainability. Modern React with hooks reduces the strict need for this pattern, but it's still valuable for complex scenarios. Custom hooks can serve as modern "containers" by extracting stateful logic. Test presentational components with mock props and container logic separately. Consider compound components with context for complex component APIs.
