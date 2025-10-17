# How would you split a large UI into components

This file explains strategies for breaking down complex user interfaces into manageable, reusable React components.
Essential for building maintainable and scalable applications.

---

## Component splitting principles

### Single Responsibility Principle
Each component should have one clear purpose and reason to change.

### Composition over Inheritance
Build complex UIs by combining simpler components rather than creating deep inheritance hierarchies.

### Data Flow Considerations
Split components based on how data flows through the application - separate data fetching from presentation.

### Reusability Boundaries
Identify patterns that could be reused across different parts of the application.

---

## Splitting strategies

### 1. By Visual Hierarchy

```jsx
// Before: Monolithic dashboard component
function MonolithicDashboard() {
  const [user, setUser] = useState(null);
  const [analytics, setAnalytics] = useState(null);
  const [notifications, setNotifications] = useState([]);
  const [projects, setProjects] = useState([]);

  // Lots of useEffects and logic...

  return (
    <div className="dashboard">
      <header className="dashboard-header">
        <div className="logo">
          <img src="/logo.png" alt="Logo" />
          <h1>Dashboard</h1>
        </div>
        <nav className="main-nav">
          <a href="/projects">Projects</a>
          <a href="/analytics">Analytics</a>
          <a href="/settings">Settings</a>
        </nav>
        <div className="user-menu">
          <img src={user?.avatar} alt={user?.name} />
          <span>{user?.name}</span>
          <div className="dropdown">
            <a href="/profile">Profile</a>
            <a href="/logout">Logout</a>
          </div>
        </div>
      </header>

      <aside className="sidebar">
        <div className="notifications">
          <h3>Notifications</h3>
          {notifications.map(notification => (
            <div key={notification.id} className="notification">
              <span className="icon">{notification.icon}</span>
              <div>
                <p>{notification.message}</p>
                <small>{notification.timestamp}</small>
              </div>
            </div>
          ))}
        </div>
      </aside>

      <main className="main-content">
        <section className="analytics-section">
          <h2>Analytics Overview</h2>
          <div className="stats-grid">
            <div className="stat-card">
              <h3>Total Users</h3>
              <p className="stat-number">{analytics?.totalUsers}</p>
            </div>
            <div className="stat-card">
              <h3>Revenue</h3>
              <p className="stat-number">${analytics?.revenue}</p>
            </div>
            <div className="stat-card">
              <h3>Growth</h3>
              <p className="stat-number">{analytics?.growth}%</p>
            </div>
          </div>
        </section>

        <section className="projects-section">
          <h2>Recent Projects</h2>
          <div className="projects-grid">
            {projects.map(project => (
              <div key={project.id} className="project-card">
                <h3>{project.name}</h3>
                <p>{project.description}</p>
                <div className="project-meta">
                  <span>Due: {project.dueDate}</span>
                  <span>Progress: {project.progress}%</span>
                </div>
                <div className="project-actions">
                  <button>Edit</button>
                  <button>View</button>
                </div>
              </div>
            ))}
          </div>
        </section>
      </main>
    </div>
  );
}

// After: Split by visual hierarchy
function Dashboard() {
  return (
    <div className="dashboard">
      <DashboardHeader />
      <div className="dashboard-body">
        <Sidebar />
        <MainContent />
      </div>
    </div>
  );
}

function DashboardHeader() {
  return (
    <header className="dashboard-header">
      <Logo />
      <MainNavigation />
      <UserMenu />
    </header>
  );
}

function Logo() {
  return (
    <div className="logo">
      <img src="/logo.png" alt="Logo" />
      <h1>Dashboard</h1>
    </div>
  );
}

function MainNavigation() {
  const navItems = [
    { href: '/projects', label: 'Projects' },
    { href: '/analytics', label: 'Analytics' },
    { href: '/settings', label: 'Settings' }
  ];

  return (
    <nav className="main-nav">
      {navItems.map(item => (
        <a key={item.href} href={item.href}>
          {item.label}
        </a>
      ))}
    </nav>
  );
}

function UserMenu() {
  const { user } = useAuth();
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className="user-menu">
      <button 
        className="user-trigger"
        onClick={() => setIsOpen(!isOpen)}
      >
        <img src={user?.avatar} alt={user?.name} />
        <span>{user?.name}</span>
      </button>
      
      {isOpen && (
        <div className="dropdown">
          <a href="/profile">Profile</a>
          <a href="/logout">Logout</a>
        </div>
      )}
    </div>
  );
}

function Sidebar() {
  return (
    <aside className="sidebar">
      <NotificationPanel />
    </aside>
  );
}

function MainContent() {
  return (
    <main className="main-content">
      <AnalyticsSection />
      <ProjectsSection />
    </main>
  );
}
```

### 2. By Data Responsibility

```jsx
// Split by data concerns - Container vs Presentational

// Data containers
function AnalyticsContainer() {
  const { data: analytics, loading, error } = useAnalytics();

  if (loading) return <AnalyticsSkeleton />;
  if (error) return <AnalyticsError onRetry={() => window.location.reload()} />;

  return <AnalyticsPresentation data={analytics} />;
}

function ProjectsContainer() {
  const { 
    projects, 
    loading, 
    error, 
    createProject, 
    updateProject, 
    deleteProject 
  } = useProjects();

  const handleProjectAction = (action, projectId, data) => {
    switch (action) {
      case 'edit':
        updateProject(projectId, data);
        break;
      case 'delete':
        deleteProject(projectId);
        break;
      case 'create':
        createProject(data);
        break;
    }
  };

  return (
    <ProjectsPresentation
      projects={projects}
      loading={loading}
      error={error}
      onProjectAction={handleProjectAction}
    />
  );
}

// Presentational components
function AnalyticsPresentation({ data }) {
  return (
    <section className="analytics-section">
      <h2>Analytics Overview</h2>
      <StatsGrid stats={data} />
    </section>
  );
}

function StatsGrid({ stats }) {
  const statItems = [
    { label: 'Total Users', value: stats.totalUsers, format: 'number' },
    { label: 'Revenue', value: stats.revenue, format: 'currency' },
    { label: 'Growth', value: stats.growth, format: 'percentage' }
  ];

  return (
    <div className="stats-grid">
      {statItems.map(item => (
        <StatCard key={item.label} {...item} />
      ))}
    </div>
  );
}

function StatCard({ label, value, format }) {
  const formatValue = (val, fmt) => {
    switch (fmt) {
      case 'currency': return `$${val}`;
      case 'percentage': return `${val}%`;
      default: return val;
    }
  };

  return (
    <div className="stat-card">
      <h3>{label}</h3>
      <p className="stat-number">{formatValue(value, format)}</p>
    </div>
  );
}
```

### 3. By Feature Boundaries

```jsx
// Feature-based splitting
function Dashboard() {
  return (
    <div className="dashboard">
      <Header />
      <div className="dashboard-content">
        <div className="main-area">
          <AnalyticsFeature />
          <ProjectManagementFeature />
        </div>
        <div className="sidebar-area">
          <NotificationsFeature />
          <QuickActionsFeature />
        </div>
      </div>
    </div>
  );
}

// Each feature manages its own state and UI
function AnalyticsFeature() {
  return (
    <section className="feature-analytics">
      <FeatureHeader title="Analytics" />
      <AnalyticsContainer />
    </section>
  );
}

function ProjectManagementFeature() {
  return (
    <section className="feature-projects">
      <FeatureHeader 
        title="Projects" 
        actions={<NewProjectButton />}
      />
      <ProjectsContainer />
    </section>
  );
}

function NotificationsFeature() {
  return (
    <section className="feature-notifications">
      <FeatureHeader title="Notifications" />
      <NotificationsContainer />
    </section>
  );
}

// Reusable feature components
function FeatureHeader({ title, actions }) {
  return (
    <div className="feature-header">
      <h2>{title}</h2>
      {actions && <div className="feature-actions">{actions}</div>}
    </div>
  );
}
```

---

## Complex example: E-commerce product page

### Before: Monolithic component

```jsx
// 🚫 Bad: Everything in one component
function ProductPageMonolith({ productId }) {
  const [product, setProduct] = useState(null);
  const [reviews, setReviews] = useState([]);
  const [recommendations, setRecommendations] = useState([]);
  const [selectedVariant, setSelectedVariant] = useState(null);
  const [quantity, setQuantity] = useState(1);
  const [cartItems, setCartItems] = useState([]);
  const [wishlistItems, setWishlistItems] = useState([]);
  const [showReviewForm, setShowReviewForm] = useState(false);
  
  // ... massive amounts of logic and JSX
  
  return (
    <div className="product-page">
      {/* Hundreds of lines of JSX */}
    </div>
  );
}
```

### After: Well-structured components

```jsx
// ✅ Good: Split into logical components

// Main page component - orchestrates the layout
function ProductPage({ productId }) {
  return (
    <div className="product-page">
      <Breadcrumbs productId={productId} />
      
      <div className="product-main">
        <ProductGallery productId={productId} />
        <ProductDetails productId={productId} />
      </div>
      
      <div className="product-secondary">
        <ProductReviews productId={productId} />
        <ProductRecommendations productId={productId} />
      </div>
    </div>
  );
}

// Product details section
function ProductDetails({ productId }) {
  const { product, loading, error } = useProduct(productId);
  
  if (loading) return <ProductDetailsSkeleton />;
  if (error) return <ProductDetailsError />;
  
  return (
    <div className="product-details">
      <ProductInfo product={product} />
      <ProductVariants 
        variants={product.variants}
        defaultVariant={product.defaultVariant}
      />
      <ProductActions product={product} />
    </div>
  );
}

// Product information display
function ProductInfo({ product }) {
  return (
    <div className="product-info">
      <h1 className="product-title">{product.name}</h1>
      <ProductRating rating={product.rating} reviewCount={product.reviewCount} />
      <ProductPrice 
        price={product.price} 
        originalPrice={product.originalPrice}
        discount={product.discount}
      />
      <ProductDescription description={product.description} />
      <ProductSpecifications specs={product.specifications} />
    </div>
  );
}

// Product variants (size, color, etc.)
function ProductVariants({ variants, defaultVariant }) {
  const [selectedVariant, setSelectedVariant] = useState(defaultVariant);
  
  return (
    <div className="product-variants">
      {variants.map(variantGroup => (
        <VariantGroup
          key={variantGroup.type}
          type={variantGroup.type}
          options={variantGroup.options}
          selected={selectedVariant[variantGroup.type]}
          onSelect={(value) => 
            setSelectedVariant(prev => ({ 
              ...prev, 
              [variantGroup.type]: value 
            }))
          }
        />
      ))}
    </div>
  );
}

// Individual variant group (e.g., colors)
function VariantGroup({ type, options, selected, onSelect }) {
  return (
    <div className="variant-group">
      <h4 className="variant-label">{type}</h4>
      <div className="variant-options">
        {options.map(option => (
          <VariantOption
            key={option.value}
            option={option}
            selected={selected === option.value}
            onSelect={() => onSelect(option.value)}
          />
        ))}
      </div>
    </div>
  );
}

// Product actions (add to cart, wishlist, etc.)
function ProductActions({ product }) {
  const [quantity, setQuantity] = useState(1);
  const { addToCart, isInCart } = useCart();
  const { addToWishlist, removeFromWishlist, isInWishlist } = useWishlist();
  
  return (
    <div className="product-actions">
      <QuantitySelector 
        value={quantity}
        onChange={setQuantity}
        max={product.stockQuantity}
      />
      
      <div className="action-buttons">
        <AddToCartButton
          product={product}
          quantity={quantity}
          onAdd={addToCart}
          disabled={!product.inStock}
        />
        
        <WishlistButton
          product={product}
          isInWishlist={isInWishlist(product.id)}
          onToggle={() => 
            isInWishlist(product.id) 
              ? removeFromWishlist(product.id)
              : addToWishlist(product.id)
          }
        />
      </div>
      
      <ProductAvailability product={product} />
    </div>
  );
}

// Product gallery component
function ProductGallery({ productId }) {
  const { images, loading } = useProductImages(productId);
  const [selectedImage, setSelectedImage] = useState(0);
  
  if (loading) return <GallerySkeleton />;
  
  return (
    <div className="product-gallery">
      <MainImage 
        image={images[selectedImage]}
        onZoom={(coords) => console.log('Zoom to:', coords)}
      />
      <ThumbnailList 
        images={images}
        selectedIndex={selectedImage}
        onSelect={setSelectedImage}
      />
    </div>
  );
}

// Reviews section
function ProductReviews({ productId }) {
  const { reviews, loading, addReview } = useProductReviews(productId);
  const [showReviewForm, setShowReviewForm] = useState(false);
  
  return (
    <section className="product-reviews">
      <ReviewsHeader 
        reviewCount={reviews.length}
        onWriteReview={() => setShowReviewForm(true)}
      />
      
      <ReviewsSummary reviews={reviews} />
      
      <ReviewsList reviews={reviews} />
      
      {showReviewForm && (
        <ReviewForm
          productId={productId}
          onSubmit={addReview}
          onCancel={() => setShowReviewForm(false)}
        />
      )}
    </section>
  );
}
```

---

## Atomic Design Methodology

### Atoms (Basic UI elements)

```jsx
// Atoms - Basic building blocks
function Button({ variant = 'primary', size = 'medium', children, ...props }) {
  return (
    <button 
      className={`btn btn--${variant} btn--${size}`}
      {...props}
    >
      {children}
    </button>
  );
}

function Input({ label, error, ...props }) {
  return (
    <div className="input-group">
      {label && <label className="input-label">{label}</label>}
      <input 
        className={`input ${error ? 'input--error' : ''}`}
        {...props}
      />
      {error && <span className="input-error">{error}</span>}
    </div>
  );
}

function Badge({ variant = 'default', children }) {
  return (
    <span className={`badge badge--${variant}`}>
      {children}
    </span>
  );
}
```

### Molecules (Simple component combinations)

```jsx
// Molecules - Combinations of atoms
function SearchBox({ onSearch, placeholder = "Search..." }) {
  const [query, setQuery] = useState('');
  
  const handleSubmit = (e) => {
    e.preventDefault();
    onSearch(query);
  };
  
  return (
    <form className="search-box" onSubmit={handleSubmit}>
      <Input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder={placeholder}
      />
      <Button type="submit" variant="primary">
        Search
      </Button>
    </form>
  );
}

function ProductPrice({ price, originalPrice, discount }) {
  return (
    <div className="product-price">
      <span className="current-price">${price}</span>
      {originalPrice && originalPrice > price && (
        <>
          <span className="original-price">${originalPrice}</span>
          <Badge variant="sale">{discount}% OFF</Badge>
        </>
      )}
    </div>
  );
}

function UserAvatar({ user, size = 'medium', showName = false }) {
  return (
    <div className={`user-avatar user-avatar--${size}`}>
      <img 
        src={user.avatar} 
        alt={user.name}
        className="avatar-image"
      />
      {showName && (
        <span className="avatar-name">{user.name}</span>
      )}
    </div>
  );
}
```

### Organisms (Complex component groups)

```jsx
// Organisms - Complex functional units
function ProductCard({ product, onAddToCart, onAddToWishlist }) {
  return (
    <div className="product-card">
      <div className="product-image">
        <img src={product.image} alt={product.name} />
        {product.discount && (
          <Badge variant="sale" className="product-badge">
            {product.discount}% OFF
          </Badge>
        )}
      </div>
      
      <div className="product-info">
        <h3 className="product-name">{product.name}</h3>
        <ProductPrice 
          price={product.price}
          originalPrice={product.originalPrice}
          discount={product.discount}
        />
      </div>
      
      <div className="product-actions">
        <Button 
          variant="primary" 
          onClick={() => onAddToCart(product.id)}
          disabled={!product.inStock}
        >
          Add to Cart
        </Button>
        <Button 
          variant="secondary" 
          onClick={() => onAddToWishlist(product.id)}
        >
          ♡
        </Button>
      </div>
    </div>
  );
}

function NavigationHeader({ user, cartItemCount }) {
  return (
    <header className="navigation-header">
      <div className="nav-brand">
        <img src="/logo.png" alt="Logo" />
      </div>
      
      <SearchBox onSearch={(query) => navigate(`/search?q=${query}`)} />
      
      <nav className="main-nav">
        <a href="/products">Products</a>
        <a href="/categories">Categories</a>
        <a href="/deals">Deals</a>
      </nav>
      
      <div className="nav-actions">
        <Button variant="ghost" className="cart-button">
          Cart ({cartItemCount})
        </Button>
        <UserAvatar user={user} showName />
      </div>
    </header>
  );
}
```

---

## Component splitting decision tree

### Questions to ask:

1. **Is this component doing too much?**
   - More than 150-200 lines?
   - Multiple responsibilities?
   - Hard to understand at a glance?

2. **Can parts be reused elsewhere?**
   - Similar UI patterns in other places?
   - Generic functionality that could be abstracted?

3. **Does data flow suggest natural boundaries?**
   - Different data sources?
   - Independent state management?
   - Different update frequencies?

4. **Are there visual/functional boundaries?**
   - Distinct sections of the UI?
   - Different user interactions?
   - Separate concerns (display vs. actions)?

### Splitting strategies:

```jsx
// Strategy 1: Extract by function
function ProductPage() {
  // Split by what each part does
  return (
    <div>
      <ProductDisplay />     {/* Shows product info */}
      <ProductActions />     {/* Handles user actions */}
      <ProductReviews />     {/* Manages reviews */}
    </div>
  );
}

// Strategy 2: Extract by data
function Dashboard() {
  // Split by data sources
  return (
    <div>
      <UserDataSection />      {/* Uses user API */}
      <AnalyticsSection />     {/* Uses analytics API */}
      <NotificationsSection /> {/* Uses notifications API */}
    </div>
  );
}

// Strategy 3: Extract by reusability
function App() {
  // Split by reusable patterns
  return (
    <div>
      <Header />         {/* Reused across pages */}
      <Sidebar />        {/* Reused across pages */}
      <MainContent />    {/* Page-specific */}
      <Footer />         {/* Reused across pages */}
    </div>
  );
}
```

---

## Best practices

### Component size guidelines
* Keep components under 200 lines when possible
* If a component needs scrolling to see it all, consider splitting
* Each component should fit on one screen for easy reading

### Naming conventions
* Use descriptive, intention-revealing names
* Follow consistent naming patterns within your team
* Use composition naming (UserProfile, UserAvatar, UserMenu)

### State management
* Keep state as local as possible
* Split components when state becomes unrelated
* Use custom hooks to extract stateful logic

### Testing considerations
* Smaller components are easier to test
* Split at natural testing boundaries
* Each component should have a clear testing purpose

---

## Interview-ready summary

Split large UIs by visual hierarchy, data responsibility, and feature boundaries. Use the Single Responsibility Principle - each component should have one clear purpose. Consider the Atomic Design methodology: atoms (basic elements), molecules (simple combinations), organisms (complex groups). Split when components exceed 150-200 lines, have multiple responsibilities, or contain reusable patterns. Natural boundaries include different data sources, visual sections, and functional areas. Extract custom hooks for stateful logic, separate containers from presentation, and maintain clear prop interfaces between components.
