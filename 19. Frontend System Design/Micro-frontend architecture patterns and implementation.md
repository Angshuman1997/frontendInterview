# Micro-frontend architecture patterns and implementation

Micro-frontends extend the microservices concept to frontend development, allowing teams to work independently on different parts of a web application while maintaining a cohesive user experience. This architecture is essential for large-scale applications with multiple teams.

## Core Concepts & Benefits

### What are Micro-frontends?

**Definition:** An architectural approach where a frontend app is decomposed into individual, semi-independent "microapps" that can be developed, tested, and deployed independently.

**Key Characteristics:**
- **Independent Development** - Teams own entire vertical slices
- **Technology Agnostic** - Different micro-frontends can use different frameworks
- **Independent Deployment** - Deploy without coordinating with other teams
- **Failure Isolation** - One micro-frontend failure doesn't crash the entire app
- **Team Autonomy** - Full ownership from database to UI

### Benefits vs Traditional Monolith

```typescript
// Traditional Monolith Structure
src/
├── components/
│   ├── Header/
│   ├── Dashboard/
│   ├── UserProfile/
│   └── Analytics/
├── services/
├── utils/
└── App.tsx

// Micro-frontend Structure
apps/
├── shell/              // Container application
├── dashboard/          // Independent React app
├── user-management/    // Independent Vue app
├── analytics/          // Independent Angular app
└── shared-library/     // Common components
```

**Advantages:**
- Faster development cycles
- Technology diversity
- Better team scaling
- Independent deployments
- Easier maintenance of large codebases

**Challenges:**
- Increased complexity
- Bundle size overhead
- Runtime coupling issues
- Shared state management complexity

## Implementation Patterns

### 1. Module Federation (Webpack 5)

**Host Application Setup:**
```typescript
// webpack.config.js (Shell/Host)
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  mode: 'development',
  devServer: {
    port: 3000,
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        dashboard: 'dashboard@http://localhost:3001/remoteEntry.js',
        userProfile: 'userProfile@http://localhost:3002/remoteEntry.js',
        analytics: 'analytics@http://localhost:3003/remoteEntry.js',
      },
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
      },
    }),
  ],
};

// Remote Application Setup
// webpack.config.js (Dashboard Micro-frontend)
module.exports = {
  mode: 'development',
  devServer: {
    port: 3001,
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'dashboard',
      filename: 'remoteEntry.js',
      exposes: {
        './Dashboard': './src/Dashboard',
        './DashboardWidget': './src/components/DashboardWidget',
      },
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
      },
    }),
  ],
};
```

**Dynamic Remote Loading:**
```typescript
// Dynamic micro-frontend loader
const MicroFrontendLoader: React.FC<{
  remoteName: string;
  moduleName: string;
  fallback?: React.ReactNode;
  errorFallback?: React.ReactNode;
}> = ({ remoteName, moduleName, fallback, errorFallback }) => {
  const [Component, setComponent] = useState<React.ComponentType | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const loadComponent = async () => {
      try {
        setLoading(true);
        setError(null);
        
        // Dynamic import with error handling
        const module = await import(`${remoteName}/${moduleName}`);
        setComponent(() => module.default);
      } catch (err) {
        console.error(`Failed to load ${remoteName}/${moduleName}:`, err);
        setError(err as Error);
      } finally {
        setLoading(false);
      }
    };

    loadComponent();
  }, [remoteName, moduleName]);

  if (loading) return fallback || <MicroFrontendSkeleton />;
  if (error) return errorFallback || <MicroFrontendError error={error} />;
  if (!Component) return null;

  return (
    <ErrorBoundary
      fallback={<MicroFrontendError error={new Error('Runtime error')} />}
    >
      <Component />
    </ErrorBoundary>
  );
};

// Usage in Shell Application
const App: React.FC = () => {
  return (
    <Router>
      <Layout>
        <Routes>
          <Route 
            path="/dashboard" 
            element={
              <MicroFrontendLoader 
                remoteName="dashboard" 
                moduleName="Dashboard" 
              />
            } 
          />
          <Route 
            path="/profile" 
            element={
              <MicroFrontendLoader 
                remoteName="userProfile" 
                moduleName="UserProfile" 
              />
            } 
          />
        </Routes>
      </Layout>
    </Router>
  );
};
```

### 2. Single-SPA Framework

**Root Configuration:**
```typescript
// root-config.js
import { registerApplication, start } from 'single-spa';

// Register micro-frontends
registerApplication({
  name: 'dashboard',
  app: () => import('./micro-frontends/dashboard/main.js'),
  activeWhen: ['/dashboard'],
});

registerApplication({
  name: 'user-profile',
  app: () => import('./micro-frontends/user-profile/main.js'),
  activeWhen: ['/profile'],
});

registerApplication({
  name: 'analytics',
  app: () => import('./micro-frontends/analytics/main.js'),
  activeWhen: ['/analytics'],
});

// Start single-spa
start();

// Micro-frontend implementation
// dashboard/main.js
import React from 'react';
import ReactDOM from 'react-dom';
import singleSpaReact from 'single-spa-react';
import Dashboard from './Dashboard';

const lifecycles = singleSpaReact({
  React,
  ReactDOM,
  rootComponent: Dashboard,
  errorBoundary(err, info, props) {
    return <ErrorFallback error={err} info={info} />;
  },
});

export const { bootstrap, mount, unmount } = lifecycles;
```

### 3. Web Components Approach

**Micro-frontend as Web Component:**
```typescript
// dashboard-micro-frontend.ts
class DashboardMicroFrontend extends HTMLElement {
  private reactRoot: any;
  private shadowRoot: ShadowRoot;

  constructor() {
    super();
    this.shadowRoot = this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    const props = this.getPropsFromAttributes();
    this.render(props);
  }

  disconnectedCallback() {
    if (this.reactRoot) {
      this.reactRoot.unmount();
    }
  }

  attributeChangedCallback() {
    const props = this.getPropsFromAttributes();
    this.render(props);
  }

  static get observedAttributes() {
    return ['user-id', 'theme', 'config'];
  }

  private getPropsFromAttributes() {
    return {
      userId: this.getAttribute('user-id'),
      theme: this.getAttribute('theme'),
      config: JSON.parse(this.getAttribute('config') || '{}'),
    };
  }

  private render(props: any) {
    import('./Dashboard').then(({ Dashboard }) => {
      if (!this.reactRoot) {
        this.reactRoot = ReactDOM.createRoot(this.shadowRoot);
      }
      this.reactRoot.render(<Dashboard {...props} />);
    });
  }
}

// Register custom element
customElements.define('dashboard-micro-frontend', DashboardMicroFrontend);

// Usage in host application
const App: React.FC = () => {
  const [userId, setUserId] = useState('123');
  const [theme, setTheme] = useState('dark');

  return (
    <div>
      <dashboard-micro-frontend
        user-id={userId}
        theme={theme}
        config={JSON.stringify({ showAnalytics: true })}
      />
    </div>
  );
};
```

## Communication Patterns

### 1. Event-Driven Communication

**Custom Event System:**
```typescript
// Event types
interface MicroFrontendEvents {
  'user:login': { user: User };
  'user:logout': {};
  'navigation:change': { route: string };
  'theme:change': { theme: Theme };
  'data:update': { type: string; data: any };
}

// Event bus for micro-frontend communication
class MicroFrontendEventBus {
  private eventTarget = new EventTarget();

  emit<K extends keyof MicroFrontendEvents>(
    eventType: K,
    data: MicroFrontendEvents[K]
  ) {
    const event = new CustomEvent(eventType, { detail: data });
    this.eventTarget.dispatchEvent(event);
  }

  on<K extends keyof MicroFrontendEvents>(
    eventType: K,
    callback: (data: MicroFrontendEvents[K]) => void
  ) {
    const handler = (event: any) => callback(event.detail);
    this.eventTarget.addEventListener(eventType, handler);
    
    return () => this.eventTarget.removeEventListener(eventType, handler);
  }
}

// Global event bus instance
const eventBus = new MicroFrontendEventBus();

// Usage in micro-frontend
const UserProfile: React.FC = () => {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    // Listen for user login events
    const unsubscribe = eventBus.on('user:login', ({ user }) => {
      setUser(user);
    });

    return unsubscribe;
  }, []);

  const handleLogout = () => {
    setUser(null);
    eventBus.emit('user:logout', {});
  };

  return (
    <div>
      {user ? (
        <div>
          <h1>Welcome, {user.name}</h1>
          <button onClick={handleLogout}>Logout</button>
        </div>
      ) : (
        <LoginForm />
      )}
    </div>
  );
};

// React hook for event bus
const useEventBus = () => {
  const emit = useCallback(<K extends keyof MicroFrontendEvents>(
    eventType: K,
    data: MicroFrontendEvents[K]
  ) => {
    eventBus.emit(eventType, data);
  }, []);

  const on = useCallback(<K extends keyof MicroFrontendEvents>(
    eventType: K,
    callback: (data: MicroFrontendEvents[K]) => void
  ) => {
    return eventBus.on(eventType, callback);
  }, []);

  return { emit, on };
};
```

### 2. Shared State Management

**Global State Provider:**
```typescript
// Shared state for micro-frontends
interface SharedState {
  user: User | null;
  theme: Theme;
  notifications: Notification[];
  permissions: Permission[];
}

class SharedStateManager {
  private state: SharedState;
  private subscribers = new Set<(state: SharedState) => void>();

  constructor(initialState: SharedState) {
    this.state = initialState;
  }

  getState(): SharedState {
    return { ...this.state };
  }

  setState(updater: Partial<SharedState> | ((prev: SharedState) => SharedState)) {
    const newState = typeof updater === 'function' 
      ? updater(this.state)
      : { ...this.state, ...updater };

    this.state = newState;
    this.notifySubscribers();
  }

  subscribe(callback: (state: SharedState) => void) {
    this.subscribers.add(callback);
    return () => this.subscribers.delete(callback);
  }

  private notifySubscribers() {
    this.subscribers.forEach(callback => callback(this.state));
  }
}

// Global shared state instance
const sharedStateManager = new SharedStateManager({
  user: null,
  theme: 'light',
  notifications: [],
  permissions: [],
});

// React hook for shared state
const useSharedState = <K extends keyof SharedState>(
  key: K
): [SharedState[K], (value: SharedState[K]) => void] => {
  const [value, setValue] = useState(sharedStateManager.getState()[key]);

  useEffect(() => {
    const unsubscribe = sharedStateManager.subscribe((state) => {
      setValue(state[key]);
    });

    return unsubscribe;
  }, [key]);

  const updateValue = useCallback((newValue: SharedState[K]) => {
    sharedStateManager.setState({ [key]: newValue } as Partial<SharedState>);
  }, [key]);

  return [value, updateValue];
};

// Usage in micro-frontends
const Header: React.FC = () => {
  const [user, setUser] = useSharedState('user');
  const [theme, setTheme] = useSharedState('theme');

  return (
    <header>
      <h1>My App</h1>
      {user && <span>Welcome, {user.name}</span>}
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </header>
  );
};
```

## Routing & Navigation

### 1. Coordinated Routing

**Shell Application Router:**
```typescript
// Shell router that coordinates micro-frontend routing
const ShellRouter: React.FC = () => {
  const [currentRoute, setCurrentRoute] = useState(window.location.pathname);

  useEffect(() => {
    const handleRouteChange = () => {
      setCurrentRoute(window.location.pathname);
      
      // Notify micro-frontends of route changes
      eventBus.emit('navigation:change', { route: window.location.pathname });
    };

    window.addEventListener('popstate', handleRouteChange);
    return () => window.removeEventListener('popstate', handleRouteChange);
  }, []);

  const navigate = (path: string) => {
    window.history.pushState({}, '', path);
    setCurrentRoute(path);
    eventBus.emit('navigation:change', { route: path });
  };

  return (
    <NavigationContext.Provider value={{ currentRoute, navigate }}>
      <Layout>
        {currentRoute.startsWith('/dashboard') && (
          <MicroFrontendLoader remoteName="dashboard" moduleName="Dashboard" />
        )}
        {currentRoute.startsWith('/profile') && (
          <MicroFrontendLoader remoteName="userProfile" moduleName="UserProfile" />
        )}
        {currentRoute.startsWith('/analytics') && (
          <MicroFrontendLoader remoteName="analytics" moduleName="Analytics" />
        )}
      </Layout>
    </NavigationContext.Provider>
  );
};

// Hook for navigation in micro-frontends
const useNavigation = () => {
  const context = useContext(NavigationContext);
  
  if (!context) {
    throw new Error('useNavigation must be used within NavigationProvider');
  }
  
  return context;
};
```

### 2. Sub-routing in Micro-frontends

**Micro-frontend Internal Routing:**
```typescript
// Dashboard micro-frontend with internal routing
const Dashboard: React.FC = () => {
  const { currentRoute } = useNavigation();
  const [internalRoute, setInternalRoute] = useState('/overview');

  useEffect(() => {
    // Extract internal route from global route
    const dashboardRoute = currentRoute.replace('/dashboard', '') || '/overview';
    setInternalRoute(dashboardRoute);
  }, [currentRoute]);

  return (
    <div className="dashboard">
      <DashboardNav currentRoute={internalRoute} />
      
      {internalRoute === '/overview' && <DashboardOverview />}
      {internalRoute === '/widgets' && <DashboardWidgets />}
      {internalRoute === '/settings' && <DashboardSettings />}
    </div>
  );
};

// Navigation component
const DashboardNav: React.FC<{ currentRoute: string }> = ({ currentRoute }) => {
  const { navigate } = useNavigation();

  const handleNavigation = (route: string) => {
    navigate(`/dashboard${route}`);
  };

  return (
    <nav>
      <button 
        className={currentRoute === '/overview' ? 'active' : ''}
        onClick={() => handleNavigation('/overview')}
      >
        Overview
      </button>
      <button 
        className={currentRoute === '/widgets' ? 'active' : ''}
        onClick={() => handleNavigation('/widgets')}
      >
        Widgets
      </button>
    </nav>
  );
};
```

## Deployment & CI/CD

### 1. Independent Deployment Pipeline

**GitHub Actions for Micro-frontend:**
```yaml
# .github/workflows/dashboard-deployment.yml
name: Dashboard Micro-frontend Deployment

on:
  push:
    branches: [main]
    paths: ['apps/dashboard/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./apps/dashboard
      
      - name: Run tests
        run: npm test
        working-directory: ./apps/dashboard
      
      - name: Type check
        run: npm run type-check
        working-directory: ./apps/dashboard

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./apps/dashboard
      
      - name: Build
        run: npm run build
        working-directory: ./apps/dashboard
        env:
          PUBLIC_URL: https://cdn.example.com/dashboard
      
      - name: Deploy to CDN
        run: |
          aws s3 sync ./apps/dashboard/dist s3://micro-frontends-cdn/dashboard/
          aws cloudfront create-invalidation --distribution-id $CDN_DISTRIBUTION_ID --paths "/dashboard/*"
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          CDN_DISTRIBUTION_ID: ${{ secrets.CDN_DISTRIBUTION_ID }}
```

### 2. Version Management

**Runtime Version Resolution:**
```typescript
// Version management for micro-frontends
interface MicroFrontendManifest {
  name: string;
  version: string;
  url: string;
  dependencies: Record<string, string>;
}

class MicroFrontendRegistry {
  private manifests = new Map<string, MicroFrontendManifest>();

  async loadManifest(name: string): Promise<MicroFrontendManifest> {
    if (this.manifests.has(name)) {
      return this.manifests.get(name)!;
    }

    const response = await fetch(`/api/micro-frontends/${name}/manifest`);
    const manifest: MicroFrontendManifest = await response.json();
    
    this.manifests.set(name, manifest);
    return manifest;
  }

  async loadMicroFrontend(name: string, version?: string) {
    const manifest = await this.loadManifest(name);
    
    // Use specific version or latest
    const targetVersion = version || manifest.version;
    const url = manifest.url.replace('{version}', targetVersion);
    
    return import(url);
  }

  checkCompatibility(
    microFrontend: string, 
    requiredDependencies: Record<string, string>
  ): boolean {
    const manifest = this.manifests.get(microFrontend);
    if (!manifest) return false;

    return Object.entries(requiredDependencies).every(([dep, version]) => {
      const actualVersion = manifest.dependencies[dep];
      return actualVersion && this.isVersionCompatible(actualVersion, version);
    });
  }

  private isVersionCompatible(actual: string, required: string): boolean {
    // Implement semver compatibility check
    return true; // Simplified
  }
}

// Usage
const registry = new MicroFrontendRegistry();

const DynamicMicroFrontend: React.FC<{
  name: string;
  version?: string;
}> = ({ name, version }) => {
  const [Component, setComponent] = useState<React.ComponentType | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    registry.loadMicroFrontend(name, version)
      .then(module => setComponent(() => module.default))
      .catch(err => setError(err.message));
  }, [name, version]);

  if (error) return <div>Error loading {name}: {error}</div>;
  if (!Component) return <div>Loading {name}...</div>;

  return <Component />;
};
```

## Best Practices & Patterns

### 1. Shared Dependencies

**Dependency Management:**
```typescript
// webpack.config.js - Shared dependencies configuration
const sharedDependencies = {
  react: {
    singleton: true,
    strictVersion: true,
    requiredVersion: '^18.0.0',
  },
  'react-dom': {
    singleton: true,
    strictVersion: true,
    requiredVersion: '^18.0.0',
  },
  '@company/design-system': {
    singleton: true,
    strictVersion: true,
    requiredVersion: '^2.0.0',
  },
  'lodash': {
    singleton: false, // Allow multiple versions
  },
};

// Runtime dependency resolution
const useDynamicImport = (moduleName: string) => {
  const [module, setModule] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    import(moduleName)
      .then(setModule)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [moduleName]);

  return { module, loading, error };
};
```

### 2. Testing Strategies

**Integration Testing for Micro-frontends:**
```typescript
// Integration test for micro-frontend communication
describe('Micro-frontend Integration', () => {
  beforeEach(() => {
    // Setup test environment
    global.eventBus = new MicroFrontendEventBus();
    global.sharedState = new SharedStateManager(initialState);
  });

  test('should communicate between micro-frontends', async () => {
    const { container: shellContainer } = render(<ShellApp />);
    const { container: dashboardContainer } = render(<Dashboard />);

    // Simulate user login in shell
    const loginButton = getByTestId(shellContainer, 'login-button');
    fireEvent.click(loginButton);

    // Check if dashboard receives the event
    await waitFor(() => {
      const userInfo = getByTestId(dashboardContainer, 'user-info');
      expect(userInfo).toHaveTextContent('Welcome, John');
    });
  });

  test('should handle micro-frontend failures gracefully', async () => {
    // Mock failed micro-frontend load
    jest.spyOn(console, 'error').mockImplementation(() => {});
    
    const { container } = render(
      <MicroFrontendLoader 
        remoteName="broken-micro-frontend" 
        moduleName="Component" 
      />
    );

    await waitFor(() => {
      const errorFallback = getByTestId(container, 'error-fallback');
      expect(errorFallback).toBeInTheDocument();
    });
  });
});

// End-to-end testing with Playwright
test('micro-frontend navigation', async ({ page }) => {
  await page.goto('/');
  
  // Navigate to dashboard micro-frontend
  await page.click('[data-testid="dashboard-link"]');
  await page.waitForSelector('[data-testid="dashboard-loaded"]');
  
  // Verify dashboard content
  await expect(page.locator('h1')).toContainText('Dashboard');
  
  // Navigate to profile micro-frontend
  await page.click('[data-testid="profile-link"]');
  await page.waitForSelector('[data-testid="profile-loaded"]');
  
  // Verify profile content
  await expect(page.locator('h1')).toContainText('User Profile');
});
```

## Interview-Ready Summary

**Micro-frontends enable:**

1. **Independent Development** - Teams can work autonomously with different tech stacks
2. **Scalable Architecture** - Add new features without touching existing code
3. **Deployment Independence** - Deploy micro-frontends separately without coordination
4. **Technology Diversity** - Use the best tool for each feature
5. **Failure Isolation** - One micro-frontend failure doesn't break the entire app

**Key implementation patterns:**
- **Module Federation** for runtime composition
- **Event-driven communication** for loose coupling
- **Shared state management** for global data
- **Coordinated routing** for seamless navigation
- **Independent CI/CD** for team autonomy

**Common challenges:** Bundle size overhead, runtime complexity, shared dependency management, testing integration, and maintaining consistent UX across micro-frontends.

**Best practices:** Start with a monolith, extract micro-frontends when teams grow, establish clear contracts between micro-frontends, invest in shared design systems, and implement comprehensive monitoring.