# How to design a scalable single-page application

Designing a scalable SPA requires careful consideration of architecture, performance, maintainability, and team collaboration. This guide covers the essential patterns and practices for building SPAs that can grow from small projects to enterprise-level applications serving millions of users.

## Core Architecture Principles

### 1. Modular Architecture

**Component-Based Design:**
```typescript
// Feature-based folder structure
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   └── types/
│   ├── dashboard/
│   └── user-profile/
├── shared/
│   ├── components/
│   ├── hooks/
│   ├── utils/
│   └── types/
└── core/
    ├── api/
    ├── routing/
    └── store/
```

**Layered Architecture:**
```typescript
// Presentation Layer
interface UserProfileProps {
  userId: string;
}

const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
  const { user, loading, error } = useUser(userId);
  
  if (loading) return <UserProfileSkeleton />;
  if (error) return <ErrorBoundary error={error} />;
  
  return <UserProfileView user={user} />;
};

// Business Logic Layer (Custom Hooks)
function useUser(userId: string) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => userService.fetchUser(userId),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });

  return {
    user,
    loading: isLoading,
    error,
  };
}

// Data Access Layer
class UserService {
  private apiClient: ApiClient;

  async fetchUser(userId: string): Promise<User> {
    const response = await this.apiClient.get(`/users/${userId}`);
    return UserSchema.parse(response.data);
  }

  async updateUser(userId: string, updates: Partial<User>): Promise<User> {
    const response = await this.apiClient.patch(`/users/${userId}`, updates);
    return UserSchema.parse(response.data);
  }
}
```

### 2. Scalable State Management

**Hierarchical State Strategy:**
```typescript
// Global State (Application-wide)
interface GlobalState {
  auth: AuthState;
  theme: ThemeState;
  notifications: NotificationState;
}

// Feature State (Feature-specific)
interface DashboardState {
  widgets: Widget[];
  layout: LayoutConfig;
  filters: FilterState;
}

// Local State (Component-specific)
const UserForm: React.FC = () => {
  const [formData, setFormData] = useState<UserFormData>(initialData);
  const [validationErrors, setValidationErrors] = useState<ValidationErrors>({});
  // ...
};

// State Management Pattern
// Use Context for theme, auth, etc.
const AuthContext = createContext<AuthContextValue>(null!);

// Use external store for complex state
const useAppStore = create<AppState>((set, get) => ({
  // Global app state
}));

// Use server state libraries for API data
const { data: users } = useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
});
```

## Performance Architecture

### 1. Code Splitting & Lazy Loading

**Route-based Splitting:**
```typescript
// Route-level lazy loading
const Dashboard = lazy(() => import('../features/dashboard/Dashboard'));
const UserProfile = lazy(() => import('../features/user/UserProfile'));
const Analytics = lazy(() => import('../features/analytics/Analytics'));

const AppRouter: React.FC = () => (
  <Router>
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<UserProfile />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  </Router>
);

// Component-level lazy loading
const HeavyChart = lazy(() => import('./HeavyChart'));

const Dashboard: React.FC = () => {
  const [showChart, setShowChart] = useState(false);
  
  return (
    <div>
      <DashboardHeader />
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
};
```

**Dynamic Imports for Heavy Libraries:**
```typescript
// Lazy load heavy libraries
const PDFViewer: React.FC<{ document: Document }> = ({ document }) => {
  const [PDFLib, setPDFLib] = useState<any>(null);
  
  useEffect(() => {
    import('react-pdf').then(setPDFLib);
  }, []);
  
  if (!PDFLib) return <LoadingSkeleton />;
  
  return <PDFLib.Document file={document} />;
};

// Conditional loading based on user interaction
const useConditionalImport = (condition: boolean) => {
  const [module, setModule] = useState(null);
  
  useEffect(() => {
    if (condition) {
      import('./HeavyModule').then(setModule);
    }
  }, [condition]);
  
  return module;
};
```

### 2. Virtual Scrolling & Data Handling

**Virtual Scrolling for Large Lists:**
```typescript
import { FixedSizeList as List } from 'react-window';

interface VirtualTableProps {
  items: TableItem[];
  itemHeight: number;
}

const VirtualTable: React.FC<VirtualTableProps> = ({ items, itemHeight }) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <TableRow item={items[index]} />
    </div>
  );

  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={itemHeight}
      width="100%"
    >
      {Row}
    </List>
  );
};

// Infinite scrolling with virtual scrolling
const useInfiniteVirtualScroll = <T,>(
  fetchData: (page: number) => Promise<T[]>,
  pageSize: number = 50
) => {
  const [items, setItems] = useState<T[]>([]);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);

  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;
    
    setLoading(true);
    try {
      const page = Math.floor(items.length / pageSize);
      const newItems = await fetchData(page);
      
      if (newItems.length < pageSize) {
        setHasMore(false);
      }
      
      setItems(prev => [...prev, ...newItems]);
    } catch (error) {
      console.error('Failed to load more items:', error);
    } finally {
      setLoading(false);
    }
  }, [fetchData, items.length, loading, hasMore, pageSize]);

  return { items, loadMore, hasMore, loading };
};
```

## Scalable Architecture Patterns

### 1. Micro-Frontend Integration

**Module Federation Setup:**
```typescript
// webpack.config.js for host application
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        dashboard: 'dashboard@http://localhost:3001/remoteEntry.js',
        analytics: 'analytics@http://localhost:3002/remoteEntry.js',
      },
    }),
  ],
};

// Micro-frontend loader component
const MicroFrontendLoader: React.FC<{
  name: string;
  fallback?: React.ReactNode;
}> = ({ name, fallback = <LoadingSkeleton /> }) => {
  const Component = lazy(() => import(name));
  
  return (
    <ErrorBoundary fallback={<MicroFrontendError name={name} />}>
      <Suspense fallback={fallback}>
        <Component />
      </Suspense>
    </ErrorBoundary>
  );
};

// Usage
const Dashboard: React.FC = () => (
  <div>
    <MicroFrontendLoader name="dashboard/Dashboard" />
    <MicroFrontendLoader name="analytics/Charts" />
  </div>
);
```

### 2. Event-Driven Architecture

**Global Event System:**
```typescript
// Event system for cross-component communication
class EventBus {
  private listeners: Map<string, Set<Function>> = new Map();

  subscribe<T>(event: string, callback: (data: T) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    
    this.listeners.get(event)!.add(callback);
    
    // Return unsubscribe function
    return () => {
      this.listeners.get(event)?.delete(callback);
    };
  }

  emit<T>(event: string, data: T): void {
    this.listeners.get(event)?.forEach(callback => callback(data));
  }
}

// React hook for event bus
const useEventBus = () => {
  const eventBus = useRef(new EventBus()).current;
  
  const subscribe = useCallback(<T>(
    event: string, 
    callback: (data: T) => void
  ) => {
    return eventBus.subscribe(event, callback);
  }, [eventBus]);

  const emit = useCallback(<T>(event: string, data: T) => {
    eventBus.emit(event, data);
  }, [eventBus]);

  return { subscribe, emit };
};

// Usage in components
const UserProfile: React.FC = () => {
  const { emit } = useEventBus();
  
  const handleUserUpdate = (user: User) => {
    emit('user:updated', user);
  };
};

const Notification: React.FC = () => {
  const { subscribe } = useEventBus();
  
  useEffect(() => {
    const unsubscribe = subscribe('user:updated', (user: User) => {
      showNotification(`User ${user.name} updated successfully`);
    });
    
    return unsubscribe;
  }, [subscribe]);
};
```

## Monitoring & Performance

### 1. Performance Monitoring

**Core Web Vitals Tracking:**
```typescript
// Performance monitoring service
class PerformanceMonitor {
  private observer: PerformanceObserver;

  constructor() {
    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        this.handlePerformanceEntry(entry);
      }
    });

    this.observer.observe({ entryTypes: ['measure', 'navigation', 'paint'] });
  }

  measureComponentRender(componentName: string) {
    return {
      start: () => performance.mark(`${componentName}-start`),
      end: () => {
        performance.mark(`${componentName}-end`);
        performance.measure(
          `${componentName}-render`,
          `${componentName}-start`,
          `${componentName}-end`
        );
      },
    };
  }

  private handlePerformanceEntry(entry: PerformanceEntry) {
    // Send to analytics service
    analytics.track('performance', {
      name: entry.name,
      duration: entry.duration,
      type: entry.entryType,
    });
  }
}

// React hook for component performance tracking
const usePerformanceTracking = (componentName: string) => {
  const monitor = useRef(new PerformanceMonitor()).current;
  
  useEffect(() => {
    const measure = monitor.measureComponentRender(componentName);
    measure.start();
    
    return () => {
      measure.end();
    };
  }, [monitor, componentName]);
};

// Usage
const HeavyComponent: React.FC = () => {
  usePerformanceTracking('HeavyComponent');
  
  return <div>{/* Component content */}</div>;
};
```

### 2. Error Boundary & Monitoring

**Comprehensive Error Handling:**
```typescript
interface ErrorInfo {
  componentStack: string;
  errorBoundary?: string;
  eventId?: string;
}

class GlobalErrorBoundary extends React.Component<
  React.PropsWithChildren<{}>,
  { hasError: boolean; error?: Error }
> {
  constructor(props: React.PropsWithChildren<{}>) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Log to monitoring service
    errorMonitor.captureException(error, {
      tags: {
        section: 'react',
      },
      extra: errorInfo,
    });

    // Log to analytics
    analytics.track('error', {
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
    });
  }

  render() {
    if (this.state.hasError) {
      return (
        <ErrorFallback 
          error={this.state.error}
          onRetry={() => this.setState({ hasError: false, error: undefined })}
        />
      );
    }

    return this.props.children;
  }
}

// Feature-specific error boundaries
const FeatureErrorBoundary: React.FC<{
  featureName: string;
  fallback?: React.ComponentType<{ error: Error; onRetry: () => void }>;
  children: React.ReactNode;
}> = ({ featureName, fallback: Fallback = ErrorFallback, children }) => {
  return (
    <ErrorBoundary
      FallbackComponent={Fallback}
      onError={(error, errorInfo) => {
        errorMonitor.captureException(error, {
          tags: { feature: featureName },
          extra: errorInfo,
        });
      }}
    >
      {children}
    </ErrorBoundary>
  );
};
```

## Team Collaboration & Maintainability

### 1. Design System Integration

**Component Library Architecture:**
```typescript
// Base component with design system props
interface BaseComponentProps {
  variant?: 'primary' | 'secondary' | 'tertiary';
  size?: 'small' | 'medium' | 'large';
  className?: string;
  'data-testid'?: string;
}

// Standardized component API
const Button: React.FC<BaseComponentProps & ButtonHTMLAttributes<HTMLButtonElement>> = ({
  variant = 'primary',
  size = 'medium',
  className,
  children,
  ...props
}) => {
  const classes = cn(
    'btn',
    `btn--${variant}`,
    `btn--${size}`,
    className
  );

  return (
    <button className={classes} {...props}>
      {children}
    </button>
  );
};

// Theme provider for consistent styling
const ThemeProvider: React.FC<{ theme: Theme; children: React.ReactNode }> = ({
  theme,
  children,
}) => {
  useEffect(() => {
    // Apply CSS custom properties
    Object.entries(theme.tokens).forEach(([key, value]) => {
      document.documentElement.style.setProperty(`--${key}`, value);
    });
  }, [theme]);

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
};
```

### 2. Development Workflow

**Automated Quality Checks:**
```typescript
// Pre-commit hooks configuration
// .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Type checking
npm run type-check

# Linting
npm run lint

# Testing
npm run test:changed

# Bundle size check
npm run bundle-analyzer

// Performance budgets
// webpack.config.js
module.exports = {
  performance: {
    maxAssetSize: 250000,
    maxEntrypointSize: 250000,
    hints: 'error',
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
  },
};
```

## Interview-Ready Summary

**Scalable SPA design requires:**

1. **Modular Architecture** - Feature-based organization, layered design, clear separation of concerns
2. **Performance Optimization** - Code splitting, lazy loading, virtual scrolling, efficient state management
3. **Micro-Frontend Ready** - Module federation, event-driven communication, independent deployments
4. **Monitoring & Observability** - Performance tracking, error boundaries, analytics integration
5. **Team Collaboration** - Design system integration, automated quality checks, consistent development workflow

**Key principles:** Start simple, scale incrementally, measure everything, optimize based on data, maintain team productivity through good architecture and tooling.

The most critical aspect is **progressive enhancement** - building a solid foundation that can grow with your application's needs while maintaining performance and developer experience.