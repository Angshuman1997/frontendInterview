# Lazy loading, Suspense, code splitting

This file covers techniques for loading code and resources on-demand to improve initial load performance.
Essential for building fast, scalable React applications with efficient resource loading.

---

## Core concepts

### Code splitting
Breaking your application into smaller chunks that can be loaded on-demand instead of bundling everything into one large file.

### Lazy loading
Loading components, images, or other resources only when they're needed (often when they become visible or when user interaction requires them).

### Suspense
React's mechanism for handling asynchronous operations (like code splitting) with declarative loading states.

---

## React.lazy and Suspense

### Basic lazy loading

```jsx
import { Suspense, lazy } from 'react';

// Lazy load components
const HomePage = lazy(() => import('./pages/HomePage'));
const ProfilePage = lazy(() => import('./pages/ProfilePage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function App() {
  return (
    <div>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/profile">Profile</Link>
        <Link to="/settings">Settings</Link>
      </nav>
      
      <Suspense fallback={<div>Loading page...</div>}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/profile" element={<ProfilePage />} />
          <Route path="/settings" element={<SettingsPage />} />
        </Routes>
      </Suspense>
    </div>
  );
}
```

### Component-level lazy loading

```jsx
import { useState, Suspense, lazy } from 'react';

// Lazy load heavy components
const HeavyChart = lazy(() => import('./components/HeavyChart'));
const DataTable = lazy(() => import('./components/DataTable'));

function Dashboard({ data }) {
  const [activeTab, setActiveTab] = useState('overview');

  return (
    <div>
      <div className="tabs">
        <button onClick={() => setActiveTab('overview')}>Overview</button>
        <button onClick={() => setActiveTab('chart')}>Chart</button>
        <button onClick={() => setActiveTab('table')}>Table</button>
      </div>

      <div className="tab-content">
        {activeTab === 'overview' && <Overview data={data} />}
        
        {activeTab === 'chart' && (
          <Suspense fallback={<div>Loading chart...</div>}>
            <HeavyChart data={data} />
          </Suspense>
        )}
        
        {activeTab === 'table' && (
          <Suspense fallback={<div>Loading table...</div>}>
            <DataTable data={data} />
          </Suspense>
        )}
      </div>
    </div>
  );
}
```

---

## Advanced code splitting patterns

### Route-based splitting with preloading

```jsx
import { lazy, Suspense } from 'react';

// Create lazy components with preloading
function lazyWithPreload(componentImport) {
  const Component = lazy(componentImport);
  Component.preload = componentImport;
  return Component;
}

const HomePage = lazyWithPreload(() => import('./pages/HomePage'));
const ProductPage = lazyWithPreload(() => import('./pages/ProductPage'));

function Navigation() {
  // Preload on hover for instant navigation
  const handleMouseEnter = (component) => {
    component.preload();
  };

  return (
    <nav>
      <Link 
        to="/" 
        onMouseEnter={() => handleMouseEnter(HomePage)}
      >
        Home
      </Link>
      <Link 
        to="/products" 
        onMouseEnter={() => handleMouseEnter(ProductPage)}
      >
        Products
      </Link>
    </nav>
  );
}

function App() {
  return (
    <Router>
      <Navigation />
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/products" element={<ProductPage />} />
        </Routes>
      </Suspense>
    </Router>
  );
}
```

### Conditional loading based on user role

```jsx
import { lazy, Suspense } from 'react';

const AdminPanel = lazy(() => import('./components/AdminPanel'));
const UserDashboard = lazy(() => import('./components/UserDashboard'));
const ModeratorTools = lazy(() => import('./components/ModeratorTools'));

function Dashboard({ user }) {
  const renderContent = () => {
    switch (user.role) {
      case 'admin':
        return <AdminPanel user={user} />;
      case 'moderator':
        return <ModeratorTools user={user} />;
      default:
        return <UserDashboard user={user} />;
    }
  };

  return (
    <div>
      <Header user={user} />
      <Suspense fallback={<DashboardSkeleton />}>
        {renderContent()}
      </Suspense>
    </div>
  );
}
```

---

## Image lazy loading

### Intersection Observer approach

```jsx
import { useState, useEffect, useRef } from 'react';

function LazyImage({ src, alt, placeholder, className }) {
  const [isLoaded, setIsLoaded] = useState(false);
  const [isInView, setIsInView] = useState(false);
  const imgRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsInView(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, []);

  return (
    <div ref={imgRef} className={className}>
      {isInView && (
        <img
          src={src}
          alt={alt}
          onLoad={() => setIsLoaded(true)}
          style={{
            opacity: isLoaded ? 1 : 0,
            transition: 'opacity 0.3s ease',
          }}
        />
      )}
      {!isLoaded && isInView && (
        <div className="image-placeholder">{placeholder}</div>
      )}
    </div>
  );
}

// Usage
function Gallery({ images }) {
  return (
    <div className="gallery">
      {images.map((image) => (
        <LazyImage
          key={image.id}
          src={image.url}
          alt={image.alt}
          placeholder={<div>Loading...</div>}
          className="gallery-item"
        />
      ))}
    </div>
  );
}
```

### Progressive image loading

```jsx
function ProgressiveImage({ src, placeholder, alt }) {
  const [imageLoaded, setImageLoaded] = useState(false);
  const [imageSrc, setImageSrc] = useState(placeholder);

  useEffect(() => {
    const img = new Image();
    img.onload = () => {
      setImageSrc(src);
      setImageLoaded(true);
    };
    img.src = src;
  }, [src]);

  return (
    <div className="progressive-image">
      <img
        src={imageSrc}
        alt={alt}
        className={`image ${imageLoaded ? 'loaded' : 'loading'}`}
      />
      {!imageLoaded && <div className="image-overlay">Loading...</div>}
    </div>
  );
}
```

---

## Webpack code splitting configuration

### Dynamic imports with chunk naming

```js
// Webpack automatically creates chunks for dynamic imports
async function loadModule() {
  // Creates a separate chunk named 'chart-library'
  const { Chart } = await import(
    /* webpackChunkName: "chart-library" */ 
    './chartLibrary'
  );
  return Chart;
}

// Preload chunks
import(
  /* webpackChunkName: "admin-panel" */
  /* webpackPreload: true */
  './AdminPanel'
);

// Prefetch chunks (lower priority)
import(
  /* webpackChunkName: "user-settings" */
  /* webpackPrefetch: true */
  './UserSettings'
);
```

### Webpack splitChunks configuration

```js
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // Vendor libraries
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 10,
        },
        // Common components used across multiple chunks
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all',
          priority: 5,
          reuseExistingChunk: true,
        },
        // Large libraries
        lodash: {
          test: /[\\/]node_modules[\\/]lodash[\\/]/,
          name: 'lodash',
          chunks: 'all',
        },
        // React and related
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          chunks: 'all',
        },
      },
    },
  },
};
```

---

## Custom lazy loading hooks

### useIntersectionObserver hook

```jsx
import { useState, useEffect, useRef } from 'react';

function useIntersectionObserver(options = {}) {
  const [isIntersecting, setIsIntersecting] = useState(false);
  const ref = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      setIsIntersecting(entry.isIntersecting);
    }, options);

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => {
      if (ref.current) {
        observer.unobserve(ref.current);
      }
    };
  }, [options]);

  return [ref, isIntersecting];
}

// Usage
function LazyComponent({ children }) {
  const [ref, isIntersecting] = useIntersectionObserver({
    threshold: 0.1,
    rootMargin: '100px',
  });

  return (
    <div ref={ref}>
      {isIntersecting ? children : <div>Loading...</div>}
    </div>
  );
}
```

### useLazyLoad hook

```jsx
function useLazyLoad(importFunction, deps = []) {
  const [component, setComponent] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const loadComponent = useCallback(async () => {
    if (component) return; // Already loaded
    
    setLoading(true);
    setError(null);
    
    try {
      const module = await importFunction();
      setComponent(() => module.default || module);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, deps);

  return {
    component,
    loading,
    error,
    load: loadComponent,
  };
}

// Usage
function TabPanel({ activeTab }) {
  const { component: ChartComponent, loading, load } = useLazyLoad(
    () => import('./HeavyChart')
  );

  useEffect(() => {
    if (activeTab === 'chart') {
      load();
    }
  }, [activeTab, load]);

  if (activeTab !== 'chart') return null;
  if (loading) return <div>Loading chart...</div>;
  if (ChartComponent) return <ChartComponent />;
  
  return null;
}
```

---

## Suspense patterns

### Nested Suspense boundaries

```jsx
function App() {
  return (
    <div>
      <Header />
      
      {/* Page-level Suspense */}
      <Suspense fallback={<PageSkeleton />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </div>
  );
}

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      
      {/* Component-level Suspense */}
      <Suspense fallback={<ChartSkeleton />}>
        <LazyChart />
      </Suspense>
      
      <Suspense fallback={<TableSkeleton />}>
        <LazyDataTable />
      </Suspense>
    </div>
  );
}
```

### Error boundaries with Suspense

```jsx
class LazyLoadErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Lazy loading error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h3>Failed to load component</h3>
          <button onClick={() => this.setState({ hasError: false })}>
            Retry
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <LazyLoadErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <LazyComponent />
      </Suspense>
    </LazyLoadErrorBoundary>
  );
}
```

---

## Advanced patterns

### Preloading strategies

```jsx
// Preload on route change intention
function RoutePreloader() {
  const location = useLocation();
  
  useEffect(() => {
    // Preload likely next routes based on current route
    const preloadMap = {
      '/': ['/products', '/about'],
      '/products': ['/product/:id', '/cart'],
      '/cart': ['/checkout'],
    };
    
    const routesToPreload = preloadMap[location.pathname] || [];
    
    routesToPreload.forEach(route => {
      // Preload route components
      import(`./pages${route}`).catch(() => {
        // Ignore preload failures
      });
    });
  }, [location.pathname]);

  return null;
}

// Preload on user interaction
function ProductCard({ product }) {
  const preloadProductDetails = () => {
    import('./ProductDetails').catch(() => {});
  };

  return (
    <div 
      onMouseEnter={preloadProductDetails}
      onFocus={preloadProductDetails}
    >
      <h3>{product.name}</h3>
      <Link to={`/product/${product.id}`}>View Details</Link>
    </div>
  );
}
```

### Progressive enhancement

```jsx
function ProgressiveFeature({ children, fallback }) {
  const [isSupported, setIsSupported] = useState(false);
  const [isLoaded, setIsLoaded] = useState(false);

  useEffect(() => {
    // Check if browser supports advanced features
    const hasIntersectionObserver = 'IntersectionObserver' in window;
    const hasWebGL = !!document.createElement('canvas').getContext('webgl');
    
    if (hasIntersectionObserver && hasWebGL) {
      setIsSupported(true);
      // Load advanced component only if supported
      import('./AdvancedFeature').then(() => {
        setIsLoaded(true);
      });
    }
  }, []);

  if (!isSupported) {
    return fallback;
  }

  if (!isLoaded) {
    return <div>Loading enhanced features...</div>;
  }

  return children;
}

// Usage
function App() {
  return (
    <ProgressiveFeature 
      fallback={<BasicChart />}
    >
      <AdvancedInteractiveChart />
    </ProgressiveFeature>
  );
}
```

---

## Performance considerations

### Bundle size analysis

```js
// Monitor chunk loading performance
function trackChunkPerformance() {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (entry.name.includes('chunk')) {
        console.log(`Chunk ${entry.name} loaded in ${entry.duration}ms`);
        
        // Track slow chunk loading
        if (entry.duration > 1000) {
          console.warn('Slow chunk loading detected:', entry.name);
        }
      }
    }
  });
  
  observer.observe({ entryTypes: ['navigation', 'resource'] });
}
```

### Optimal loading strategies

```jsx
// Viewport-based loading priority
function SmartLoader({ children, priority = 'normal' }) {
  const [shouldLoad, setShouldLoad] = useState(priority === 'high');
  const [ref, isInView] = useIntersectionObserver({
    threshold: priority === 'high' ? 0 : 0.1,
    rootMargin: priority === 'high' ? '200px' : '50px',
  });

  useEffect(() => {
    if (isInView || priority === 'high') {
      setShouldLoad(true);
    }
  }, [isInView, priority]);

  return (
    <div ref={ref}>
      {shouldLoad ? children : <div>Content will load when visible</div>}
    </div>
  );
}
```

---

## Best practices checklist

* [ ] Implement route-based code splitting for major pages
* [ ] Use component-level lazy loading for heavy features
* [ ] Set up preloading for likely user interactions
* [ ] Implement proper loading states and error boundaries
* [ ] Use Intersection Observer for viewport-based loading
* [ ] Configure webpack splitChunks for optimal bundle splitting
* [ ] Monitor chunk loading performance in production
* [ ] Implement progressive enhancement for advanced features
* [ ] Use nested Suspense boundaries for granular loading states
* [ ] Test lazy loading on slow networks and devices

---

## Interview-ready summary

Code splitting breaks apps into smaller chunks loaded on-demand, improving initial load performance. React.lazy and Suspense provide declarative lazy loading with loading states. Key patterns include route-based splitting, component-level lazy loading, and preloading strategies. Image lazy loading uses Intersection Observer for viewport-based loading. Webpack splitChunks optimizes bundle composition. Advanced techniques include nested Suspense boundaries, error handling, and progressive enhancement. Monitor chunk performance and implement appropriate loading priorities based on user experience needs.
