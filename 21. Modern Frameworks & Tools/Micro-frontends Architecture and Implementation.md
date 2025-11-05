# Micro-frontends Architecture and Implementation

Micro-frontends represent a paradigm shift in frontend architecture, extending microservices concepts to the client-side. This comprehensive guide covers architectural patterns, implementation strategies, communication mechanisms, and enterprise-grade deployment patterns for building scalable, maintainable micro-frontend applications.

## Micro-frontends Architecture Fundamentals

### Core Concepts and Design Principles

```typescript
// types/microfrontend.ts
export interface MicrofrontendConfig {
  name: string;
  entry: string;
  container: string;
  routes: string[];
  exposed: Record<string, string>;
  dependencies: {
    shared: Record<string, string>;
    federated: string[];
  };
  lifecycle: {
    bootstrap?: () => Promise<void>;
    mount?: (element: HTMLElement) => Promise<void>;
    unmount?: (element: HTMLElement) => Promise<void>;
    update?: (props: any) => Promise<void>;
  };
}

export interface SharedState {
  user: {
    id: string;
    profile: UserProfile;
    permissions: string[];
  };
  theme: {
    mode: 'light' | 'dark';
    colors: Record<string, string>;
  };
  navigation: {
    currentRoute: string;
    breadcrumbs: Breadcrumb[];
  };
  notifications: Notification[];
}

export interface MicrofrontendRegistry {
  applications: Map<string, MicrofrontendConfig>;
  routes: Map<string, string>;
  sharedDependencies: Map<string, any>;
}

// architecture/MicrofrontendOrchestrator.ts
class MicrofrontendOrchestrator {
  private registry: MicrofrontendRegistry;
  private eventBus: EventBus;
  private sharedState: SharedStateManager;
  private routeResolver: RouteResolver;

  constructor() {
    this.registry = {
      applications: new Map(),
      routes: new Map(),
      sharedDependencies: new Map(),
    };
    this.eventBus = new EventBus();
    this.sharedState = new SharedStateManager();
    this.routeResolver = new RouteResolver();
  }

  async registerApplication(config: MicrofrontendConfig): Promise<void> {
    try {
      // Validate configuration
      this.validateConfig(config);

      // Load remote module
      const remoteModule = await this.loadRemoteModule(config.entry);
      
      // Register routes
      config.routes.forEach(route => {
        this.registry.routes.set(route, config.name);
      });

      // Setup shared dependencies
      await this.setupSharedDependencies(config.dependencies.shared);

      // Register lifecycle hooks
      this.registerLifecycleHooks(config);

      // Store configuration
      this.registry.applications.set(config.name, config);

      console.log(`Microfrontend ${config.name} registered successfully`);
    } catch (error) {
      console.error(`Failed to register microfrontend ${config.name}:`, error);
      throw error;
    }
  }

  async routeToApplication(path: string): Promise<void> {
    const appName = this.routeResolver.resolveRoute(path);
    if (!appName) {
      throw new Error(`No application found for route: ${path}`);
    }

    const config = this.registry.applications.get(appName);
    if (!config) {
      throw new Error(`Application ${appName} not found in registry`);
    }

    // Unmount current application
    await this.unmountCurrentApplication();

    // Mount new application
    await this.mountApplication(config, path);

    // Update shared state
    this.sharedState.updateNavigation(path);

    // Emit navigation event
    this.eventBus.emit('navigation:changed', { path, application: appName });
  }

  private async loadRemoteModule(entry: string): Promise<any> {
    return new Promise((resolve, reject) => {
      const script = document.createElement('script');
      script.src = entry;
      script.type = 'module';
      
      script.onload = () => {
        resolve(window[`__MICROFRONTEND_${entry}__`]);
      };
      
      script.onerror = () => {
        reject(new Error(`Failed to load remote module: ${entry}`));
      };
      
      document.head.appendChild(script);
    });
  }

  private validateConfig(config: MicrofrontendConfig): void {
    const required = ['name', 'entry', 'container', 'routes'];
    for (const field of required) {
      if (!config[field]) {
        throw new Error(`Missing required field: ${field}`);
      }
    }

    if (this.registry.applications.has(config.name)) {
      throw new Error(`Application ${config.name} already registered`);
    }
  }

  private async setupSharedDependencies(dependencies: Record<string, string>): Promise<void> {
    for (const [name, version] of Object.entries(dependencies)) {
      if (!this.registry.sharedDependencies.has(name)) {
        const dependency = await this.loadSharedDependency(name, version);
        this.registry.sharedDependencies.set(name, dependency);
      }
    }
  }

  private async loadSharedDependency(name: string, version: string): Promise<any> {
    // Implementation for loading shared dependencies
    // This could integrate with CDN, local registry, etc.
    return import(`https://cdn.jsdelivr.net/npm/${name}@${version}`);
  }

  private registerLifecycleHooks(config: MicrofrontendConfig): void {
    if (config.lifecycle.bootstrap) {
      this.eventBus.on(`${config.name}:bootstrap`, config.lifecycle.bootstrap);
    }
    if (config.lifecycle.mount) {
      this.eventBus.on(`${config.name}:mount`, config.lifecycle.mount);
    }
    if (config.lifecycle.unmount) {
      this.eventBus.on(`${config.name}:unmount`, config.lifecycle.unmount);
    }
    if (config.lifecycle.update) {
      this.eventBus.on(`${config.name}:update`, config.lifecycle.update);
    }
  }

  private async unmountCurrentApplication(): Promise<void> {
    const currentApp = this.sharedState.getCurrentApplication();
    if (currentApp) {
      const config = this.registry.applications.get(currentApp);
      if (config?.lifecycle.unmount) {
        const container = document.querySelector(config.container);
        if (container) {
          await config.lifecycle.unmount(container as HTMLElement);
        }
      }
    }
  }

  private async mountApplication(config: MicrofrontendConfig, path: string): Promise<void> {
    const container = document.querySelector(config.container);
    if (!container) {
      throw new Error(`Container ${config.container} not found`);
    }

    if (config.lifecycle.mount) {
      await config.lifecycle.mount(container as HTMLElement);
    }

    this.sharedState.setCurrentApplication(config.name);
  }
}
```

### Communication Mechanisms

```typescript
// communication/EventBus.ts
export class EventBus {
  private listeners: Map<string, Set<Function>> = new Map();
  private onceListeners: Map<string, Set<Function>> = new Map();

  on(event: string, callback: Function): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(callback);

    // Return unsubscribe function
    return () => this.off(event, callback);
  }

  once(event: string, callback: Function): () => void {
    if (!this.onceListeners.has(event)) {
      this.onceListeners.set(event, new Set());
    }
    this.onceListeners.get(event)!.add(callback);

    return () => this.onceListeners.get(event)?.delete(callback);
  }

  off(event: string, callback: Function): void {
    this.listeners.get(event)?.delete(callback);
    this.onceListeners.get(event)?.delete(callback);
  }

  emit(event: string, data?: any): void {
    // Regular listeners
    const listeners = this.listeners.get(event);
    if (listeners) {
      listeners.forEach(callback => {
        try {
          callback(data);
        } catch (error) {
          console.error(`Error in event listener for ${event}:`, error);
        }
      });
    }

    // Once listeners
    const onceListeners = this.onceListeners.get(event);
    if (onceListeners) {
      onceListeners.forEach(callback => {
        try {
          callback(data);
        } catch (error) {
          console.error(`Error in once listener for ${event}:`, error);
        }
      });
      this.onceListeners.delete(event);
    }
  }

  clear(): void {
    this.listeners.clear();
    this.onceListeners.clear();
  }

  getListenerCount(event: string): number {
    const regularCount = this.listeners.get(event)?.size || 0;
    const onceCount = this.onceListeners.get(event)?.size || 0;
    return regularCount + onceCount;
  }
}

// communication/SharedStateManager.ts
export class SharedStateManager {
  private state: SharedState;
  private eventBus: EventBus;
  private subscribers: Map<string, Set<Function>> = new Map();

  constructor(eventBus: EventBus) {
    this.eventBus = eventBus;
    this.state = {
      user: {
        id: '',
        profile: {} as UserProfile,
        permissions: [],
      },
      theme: {
        mode: 'light',
        colors: {},
      },
      navigation: {
        currentRoute: '/',
        breadcrumbs: [],
      },
      notifications: [],
    };
  }

  getState(): SharedState {
    return { ...this.state };
  }

  updateUser(user: Partial<SharedState['user']>): void {
    this.state.user = { ...this.state.user, ...user };
    this.notifySubscribers('user', this.state.user);
    this.eventBus.emit('state:user:updated', this.state.user);
  }

  updateTheme(theme: Partial<SharedState['theme']>): void {
    this.state.theme = { ...this.state.theme, ...theme };
    this.notifySubscribers('theme', this.state.theme);
    this.eventBus.emit('state:theme:updated', this.state.theme);
  }

  updateNavigation(currentRoute: string, breadcrumbs?: Breadcrumb[]): void {
    this.state.navigation = {
      currentRoute,
      breadcrumbs: breadcrumbs || this.generateBreadcrumbs(currentRoute),
    };
    this.notifySubscribers('navigation', this.state.navigation);
    this.eventBus.emit('state:navigation:updated', this.state.navigation);
  }

  addNotification(notification: Omit<Notification, 'id' | 'timestamp'>): void {
    const newNotification: Notification = {
      ...notification,
      id: crypto.randomUUID(),
      timestamp: Date.now(),
    };
    
    this.state.notifications.push(newNotification);
    this.notifySubscribers('notifications', this.state.notifications);
    this.eventBus.emit('state:notifications:added', newNotification);
  }

  removeNotification(id: string): void {
    this.state.notifications = this.state.notifications.filter(n => n.id !== id);
    this.notifySubscribers('notifications', this.state.notifications);
    this.eventBus.emit('state:notifications:removed', id);
  }

  subscribe(key: keyof SharedState, callback: Function): () => void {
    if (!this.subscribers.has(key)) {
      this.subscribers.set(key, new Set());
    }
    this.subscribers.get(key)!.add(callback);

    // Return unsubscribe function
    return () => this.subscribers.get(key)?.delete(callback);
  }

  private notifySubscribers(key: keyof SharedState, value: any): void {
    const callbacks = this.subscribers.get(key);
    if (callbacks) {
      callbacks.forEach(callback => {
        try {
          callback(value);
        } catch (error) {
          console.error(`Error in state subscriber for ${key}:`, error);
        }
      });
    }
  }

  private generateBreadcrumbs(route: string): Breadcrumb[] {
    const segments = route.split('/').filter(Boolean);
    const breadcrumbs: Breadcrumb[] = [
      { label: 'Home', path: '/' },
    ];

    let currentPath = '';
    segments.forEach(segment => {
      currentPath += `/${segment}`;
      breadcrumbs.push({
        label: this.formatSegmentLabel(segment),
        path: currentPath,
      });
    });

    return breadcrumbs;
  }

  private formatSegmentLabel(segment: string): string {
    return segment
      .split('-')
      .map(word => word.charAt(0).toUpperCase() + word.slice(1))
      .join(' ');
  }

  getCurrentApplication(): string | null {
    return this.state.navigation.currentApplication || null;
  }

  setCurrentApplication(appName: string): void {
    (this.state.navigation as any).currentApplication = appName;
  }
}

// communication/CrossAppCommunication.ts
export class CrossAppCommunication {
  private eventBus: EventBus;
  private messageQueue: Map<string, any[]> = new Map();
  private pendingRequests: Map<string, { resolve: Function; reject: Function; timeout: NodeJS.Timeout }> = new Map();

  constructor(eventBus: EventBus) {
    this.eventBus = eventBus;
    this.setupMessageHandling();
  }

  // Request-Response pattern
  async request(target: string, action: string, payload?: any, timeout = 5000): Promise<any> {
    const requestId = crypto.randomUUID();
    const channel = `${target}:${action}`;

    return new Promise((resolve, reject) => {
      const timeoutHandle = setTimeout(() => {
        this.pendingRequests.delete(requestId);
        reject(new Error(`Request timeout: ${channel}`));
      }, timeout);

      this.pendingRequests.set(requestId, {
        resolve,
        reject,
        timeout: timeoutHandle,
      });

      this.eventBus.emit(channel, {
        requestId,
        payload,
        responseChannel: `response:${requestId}`,
      });
    });
  }

  // Publish-Subscribe pattern
  publish(channel: string, data: any): void {
    this.eventBus.emit(channel, data);
  }

  subscribe(channel: string, callback: Function): () => void {
    return this.eventBus.on(channel, callback);
  }

  // Message queueing for offline scenarios
  queueMessage(target: string, message: any): void {
    if (!this.messageQueue.has(target)) {
      this.messageQueue.set(target, []);
    }
    this.messageQueue.get(target)!.push(message);
  }

  processQueuedMessages(target: string): void {
    const messages = this.messageQueue.get(target) || [];
    messages.forEach(message => {
      this.eventBus.emit(`${target}:queued`, message);
    });
    this.messageQueue.delete(target);
  }

  private setupMessageHandling(): void {
    this.eventBus.on(/^response:/, (data: any) => {
      const requestId = data.requestId;
      const pending = this.pendingRequests.get(requestId);
      
      if (pending) {
        clearTimeout(pending.timeout);
        this.pendingRequests.delete(requestId);
        
        if (data.error) {
          pending.reject(new Error(data.error));
        } else {
          pending.resolve(data.payload);
        }
      }
    });
  }
}
```

## Module Federation Implementation

### Webpack Module Federation Setup

```javascript
// shell-app/webpack.config.js
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  mode: 'development',
  devServer: {
    port: 3000,
    historyApiFallback: true,
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        'user-management': 'userManagement@http://localhost:3001/remoteEntry.js',
        'product-catalog': 'productCatalog@http://localhost:3002/remoteEntry.js',
        'order-processing': 'orderProcessing@http://localhost:3003/remoteEntry.js',
        'analytics-dashboard': 'analyticsDashboard@http://localhost:3004/remoteEntry.js',
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
        '@emotion/react': {
          singleton: true,
        },
        '@emotion/styled': {
          singleton: true,
        },
        'react-router-dom': {
          singleton: true,
        },
      },
    }),
  ],
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
};

// user-management/webpack.config.js
module.exports = {
  mode: 'development',
  devServer: {
    port: 3001,
    historyApiFallback: true,
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'userManagement',
      filename: 'remoteEntry.js',
      exposes: {
        './UserApp': './src/App',
        './UserProfile': './src/components/UserProfile',
        './UserList': './src/components/UserList',
        './AuthProvider': './src/providers/AuthProvider',
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
        'react-router-dom': {
          singleton: true,
        },
      },
    }),
  ],
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
};
```

### Dynamic Remote Loading

```typescript
// shell-app/src/services/RemoteLoader.ts
export class RemoteLoader {
  private loadedRemotes: Map<string, any> = new Map();
  private loadingPromises: Map<string, Promise<any>> = new Map();

  async loadRemote(remoteName: string, exposedModule: string): Promise<any> {
    const key = `${remoteName}/${exposedModule}`;
    
    // Return cached module if already loaded
    if (this.loadedRemotes.has(key)) {
      return this.loadedRemotes.get(key);
    }

    // Return existing loading promise if already in progress
    if (this.loadingPromises.has(key)) {
      return this.loadingPromises.get(key);
    }

    // Start loading process
    const loadingPromise = this.loadRemoteModule(remoteName, exposedModule);
    this.loadingPromises.set(key, loadingPromise);

    try {
      const module = await loadingPromise;
      this.loadedRemotes.set(key, module);
      this.loadingPromises.delete(key);
      return module;
    } catch (error) {
      this.loadingPromises.delete(key);
      throw error;
    }
  }

  private async loadRemoteModule(remoteName: string, exposedModule: string): Promise<any> {
    try {
      // Dynamically import the remote module
      const container = await import(remoteName);
      
      // Initialize the container
      await container.init(__webpack_share_scopes__.default);
      
      // Get the exposed module
      const factory = await container.get(exposedModule);
      const module = factory();
      
      return module;
    } catch (error) {
      console.error(`Failed to load remote ${remoteName}/${exposedModule}:`, error);
      throw new Error(`Remote loading failed: ${remoteName}/${exposedModule}`);
    }
  }

  async preloadRemote(remoteName: string, exposedModule: string): Promise<void> {
    try {
      await this.loadRemote(remoteName, exposedModule);
      console.log(`Preloaded ${remoteName}/${exposedModule}`);
    } catch (error) {
      console.warn(`Failed to preload ${remoteName}/${exposedModule}:`, error);
    }
  }

  unloadRemote(remoteName: string, exposedModule: string): void {
    const key = `${remoteName}/${exposedModule}`;
    this.loadedRemotes.delete(key);
    this.loadingPromises.delete(key);
  }

  getLoadedRemotes(): string[] {
    return Array.from(this.loadedRemotes.keys());
  }
}

// shell-app/src/components/DynamicRemoteComponent.tsx
import React, { Suspense, lazy, useState, useEffect } from 'react';
import { RemoteLoader } from '../services/RemoteLoader';
import { ErrorBoundary } from './ErrorBoundary';

interface DynamicRemoteComponentProps {
  remoteName: string;
  exposedModule: string;
  fallback?: React.ComponentType;
  props?: Record<string, any>;
  onLoad?: () => void;
  onError?: (error: Error) => void;
}

const remoteLoader = new RemoteLoader();

export function DynamicRemoteComponent({
  remoteName,
  exposedModule,
  fallback: Fallback,
  props = {},
  onLoad,
  onError,
}: DynamicRemoteComponentProps) {
  const [RemoteComponent, setRemoteComponent] = useState<React.ComponentType | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    let isMounted = true;

    const loadComponent = async () => {
      try {
        setIsLoading(true);
        setError(null);

        const module = await remoteLoader.loadRemote(remoteName, exposedModule);
        
        if (isMounted) {
          setRemoteComponent(() => module.default || module);
          onLoad?.();
        }
      } catch (err) {
        const error = err instanceof Error ? err : new Error('Unknown error');
        if (isMounted) {
          setError(error);
          onError?.(error);
        }
      } finally {
        if (isMounted) {
          setIsLoading(false);
        }
      }
    };

    loadComponent();

    return () => {
      isMounted = false;
    };
  }, [remoteName, exposedModule, onLoad, onError]);

  if (isLoading) {
    return (
      <div className="remote-loading">
        <div className="spinner" />
        <p>Loading {remoteName}...</p>
      </div>
    );
  }

  if (error) {
    if (Fallback) {
      return <Fallback />;
    }
    return (
      <div className="remote-error">
        <h3>Failed to load component</h3>
        <p>{error.message}</p>
        <button onClick={() => window.location.reload()}>
          Retry
        </button>
      </div>
    );
  }

  if (!RemoteComponent) {
    return null;
  }

  return (
    <ErrorBoundary
      fallback={Fallback || (() => (
        <div className="remote-error">
          <h3>Component Error</h3>
          <p>The remote component encountered an error</p>
        </div>
      ))}
    >
      <Suspense fallback={<div className="component-loading">Loading...</div>}>
        <RemoteComponent {...props} />
      </Suspense>
    </ErrorBoundary>
  );
}

// Usage example
export function UserManagementPage() {
  return (
    <DynamicRemoteComponent
      remoteName="user-management"
      exposedModule="./UserApp"
      props={{
        apiBaseUrl: process.env.REACT_APP_API_URL,
        theme: 'dark',
      }}
      onLoad={() => console.log('User management loaded')}
      onError={(error) => console.error('User management failed:', error)}
      fallback={() => (
        <div className="fallback-ui">
          <h2>User Management Unavailable</h2>
          <p>Please try again later.</p>
        </div>
      )}
    />
  );
}
```

### Runtime Module Federation

```typescript
// shell-app/src/services/RuntimeFederation.ts
interface RemoteConfig {
  name: string;
  url: string;
  scope: string;
  module: string;
}

export class RuntimeFederation {
  private remoteConfigs: Map<string, RemoteConfig> = new Map();
  private loadedScripts: Set<string> = new Set();

  async registerRemote(config: RemoteConfig): Promise<void> {
    this.remoteConfigs.set(config.name, config);
    
    // Load remote script if not already loaded
    if (!this.loadedScripts.has(config.url)) {
      await this.loadScript(config.url);
      this.loadedScripts.add(config.url);
    }
  }

  async loadComponent(remoteName: string): Promise<any> {
    const config = this.remoteConfigs.get(remoteName);
    if (!config) {
      throw new Error(`Remote ${remoteName} not registered`);
    }

    try {
      // Get the container
      const container = window[config.scope];
      if (!container) {
        throw new Error(`Container ${config.scope} not found`);
      }

      // Initialize the container if needed
      if (!container.__initialized) {
        await container.init(__webpack_share_scopes__.default);
        container.__initialized = true;
      }

      // Get the module factory
      const factory = await container.get(config.module);
      
      // Create and return the module
      return factory();
    } catch (error) {
      console.error(`Failed to load component from ${remoteName}:`, error);
      throw error;
    }
  }

  private loadScript(url: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const script = document.createElement('script');
      script.type = 'text/javascript';
      script.async = true;
      script.src = url;

      script.onload = () => resolve();
      script.onerror = () => reject(new Error(`Failed to load script: ${url}`));

      document.head.appendChild(script);
    });
  }

  unregisterRemote(remoteName: string): void {
    this.remoteConfigs.delete(remoteName);
  }

  getRegisteredRemotes(): string[] {
    return Array.from(this.remoteConfigs.keys());
  }
}

// Configuration-driven remote loading
export class ConfigurableRemoteLoader {
  private runtimeFederation: RuntimeFederation;
  private configService: ConfigService;

  constructor() {
    this.runtimeFederation = new RuntimeFederation();
    this.configService = new ConfigService();
  }

  async initializeFromConfig(): Promise<void> {
    try {
      const config = await this.configService.getRemoteConfig();
      
      for (const remoteConfig of config.remotes) {
        await this.runtimeFederation.registerRemote(remoteConfig);
      }

      console.log('All remotes initialized from configuration');
    } catch (error) {
      console.error('Failed to initialize remotes from config:', error);
      throw error;
    }
  }

  async loadRemoteComponent(remoteName: string): Promise<React.ComponentType> {
    try {
      const module = await this.runtimeFederation.loadComponent(remoteName);
      return module.default || module;
    } catch (error) {
      console.error(`Failed to load remote component ${remoteName}:`, error);
      throw error;
    }
  }
}

// Configuration service
class ConfigService {
  async getRemoteConfig(): Promise<{ remotes: RemoteConfig[] }> {
    const response = await fetch('/api/microfrontend-config');
    if (!response.ok) {
      throw new Error('Failed to fetch remote configuration');
    }
    return response.json();
  }
}
```

## Single-SPA Integration

### Single-SPA Application Setup

```typescript
// shell-app/src/single-spa-config.ts
import { registerApplication, start, NavigationEvent } from 'single-spa';

// Application lifecycle functions
function createLifecycles(name: string) {
  return {
    async bootstrap() {
      console.log(`Bootstrapping ${name}`);
      return Promise.resolve();
    },
    
    async mount(props: any) {
      console.log(`Mounting ${name}`, props);
      return Promise.resolve();
    },
    
    async unmount(props: any) {
      console.log(`Unmounting ${name}`, props);
      return Promise.resolve();
    },
    
    async update(props: any) {
      console.log(`Updating ${name}`, props);
      return Promise.resolve();
    },
  };
}

// Register micro-frontends
registerApplication({
  name: 'user-management',
  app: () => import('user-management/UserApp').then(module => ({
    ...createLifecycles('user-management'),
    mount: async (props) => {
      const { mount } = await import('user-management/UserApp');
      return mount(props);
    },
    unmount: async (props) => {
      const { unmount } = await import('user-management/UserApp');
      return unmount(props);
    },
  })),
  activeWhen: ['/users', '/profile'],
  customProps: {
    domElement: '#user-management-root',
    apiBaseUrl: process.env.REACT_APP_API_URL,
  },
});

registerApplication({
  name: 'product-catalog',
  app: () => import('product-catalog/ProductApp').then(module => ({
    ...createLifecycles('product-catalog'),
    mount: async (props) => {
      const { mount } = await import('product-catalog/ProductApp');
      return mount(props);
    },
    unmount: async (props) => {
      const { unmount } = await import('product-catalog/ProductApp');
      return unmount(props);
    },
  })),
  activeWhen: ['/products', '/catalog'],
  customProps: {
    domElement: '#product-catalog-root',
    apiBaseUrl: process.env.REACT_APP_API_URL,
  },
});

registerApplication({
  name: 'order-processing',
  app: () => import('order-processing/OrderApp').then(module => ({
    ...createLifecycles('order-processing'),
    mount: async (props) => {
      const { mount } = await import('order-processing/OrderApp');
      return mount(props);
    },
    unmount: async (props) => {
      const { unmount } = await import('order-processing/OrderApp');
      return unmount(props);
    },
  })),
  activeWhen: (location) => location.pathname.startsWith('/orders'),
  customProps: {
    domElement: '#order-processing-root',
    apiBaseUrl: process.env.REACT_APP_API_URL,
  },
});

// Global error handling
window.addEventListener('single-spa:routing-event', (event: CustomEvent<NavigationEvent>) => {
  console.log('Navigation event:', event.detail);
});

window.addEventListener('single-spa:app-change', (event: CustomEvent) => {
  console.log('App change event:', event.detail);
});

// Start single-spa
start({
  urlRerouteOnly: true,
});

// Micro-frontend application template
// user-management/src/index.ts
import React from 'react';
import ReactDOM from 'react-dom/client';
import { UserApp } from './App';

let root: ReactDOM.Root | null = null;

export async function mount(props: any) {
  const { domElement, ...otherProps } = props;
  const container = document.querySelector(domElement) || document.getElementById('root');
  
  if (container) {
    root = ReactDOM.createRoot(container);
    root.render(React.createElement(UserApp, otherProps));
  }
}

export async function unmount(props: any) {
  if (root) {
    root.unmount();
    root = null;
  }
}

export async function bootstrap(props: any) {
  console.log('User management bootstrapped', props);
}

export async function update(props: any) {
  console.log('User management updated', props);
  // Re-render with new props if needed
  if (root) {
    const { domElement, ...otherProps } = props;
    root.render(React.createElement(UserApp, otherProps));
  }
}

// Development mode - mount immediately
if (process.env.NODE_ENV === 'development' && !window.singleSpaNavigate) {
  const root = ReactDOM.createRoot(document.getElementById('root')!);
  root.render(React.createElement(UserApp));
}
```

## Cross-Domain Communication

### PostMessage API Implementation

```typescript
// communication/PostMessageBridge.ts
export interface MessagePayload {
  type: string;
  data: any;
  source: string;
  target: string;
  requestId?: string;
  error?: string;
}

export class PostMessageBridge {
  private allowedOrigins: Set<string> = new Set();
  private messageHandlers: Map<string, Function[]> = new Map();
  private pendingRequests: Map<string, { resolve: Function; reject: Function; timeout: NodeJS.Timeout }> = new Map();
  private isInitialized = false;

  constructor(allowedOrigins: string[] = []) {
    allowedOrigins.forEach(origin => this.allowedOrigins.add(origin));
    this.initialize();
  }

  private initialize(): void {
    if (this.isInitialized) return;

    window.addEventListener('message', this.handleMessage.bind(this));
    this.isInitialized = true;
  }

  addAllowedOrigin(origin: string): void {
    this.allowedOrigins.add(origin);
  }

  removeAllowedOrigin(origin: string): void {
    this.allowedOrigins.delete(origin);
  }

  on(messageType: string, handler: Function): () => void {
    if (!this.messageHandlers.has(messageType)) {
      this.messageHandlers.set(messageType, []);
    }
    this.messageHandlers.get(messageType)!.push(handler);

    // Return unsubscribe function
    return () => {
      const handlers = this.messageHandlers.get(messageType);
      if (handlers) {
        const index = handlers.indexOf(handler);
        if (index !== -1) {
          handlers.splice(index, 1);
        }
      }
    };
  }

  off(messageType: string, handler?: Function): void {
    if (!handler) {
      this.messageHandlers.delete(messageType);
    } else {
      const handlers = this.messageHandlers.get(messageType);
      if (handlers) {
        const index = handlers.indexOf(handler);
        if (index !== -1) {
          handlers.splice(index, 1);
        }
      }
    }
  }

  send(targetWindow: Window, payload: Omit<MessagePayload, 'source'>): void {
    const message: MessagePayload = {
      ...payload,
      source: window.location.origin,
    };

    targetWindow.postMessage(message, '*');
  }

  async request(
    targetWindow: Window, 
    type: string, 
    data: any, 
    target: string,
    timeout = 5000
  ): Promise<any> {
    const requestId = crypto.randomUUID();
    
    return new Promise((resolve, reject) => {
      const timeoutHandle = setTimeout(() => {
        this.pendingRequests.delete(requestId);
        reject(new Error(`Request timeout: ${type}`));
      }, timeout);

      this.pendingRequests.set(requestId, {
        resolve,
        reject,
        timeout: timeoutHandle,
      });

      this.send(targetWindow, {
        type,
        data,
        target,
        requestId,
      });
    });
  }

  respond(targetWindow: Window, requestId: string, data: any, error?: string): void {
    this.send(targetWindow, {
      type: 'response',
      data,
      target: 'any',
      requestId,
      error,
    });
  }

  private handleMessage(event: MessageEvent): void {
    // Security check
    if (this.allowedOrigins.size > 0 && !this.allowedOrigins.has(event.origin)) {
      console.warn(`Message from unauthorized origin: ${event.origin}`);
      return;
    }

    const message = event.data as MessagePayload;
    
    // Validate message structure
    if (!this.isValidMessage(message)) {
      console.warn('Invalid message structure:', message);
      return;
    }

    // Handle response messages
    if (message.type === 'response' && message.requestId) {
      this.handleResponse(message);
      return;
    }

    // Handle regular messages
    const handlers = this.messageHandlers.get(message.type);
    if (handlers) {
      handlers.forEach(handler => {
        try {
          handler(message.data, {
            source: message.source,
            requestId: message.requestId,
            respond: (data: any, error?: string) => {
              if (message.requestId) {
                this.respond(event.source as Window, message.requestId, data, error);
              }
            },
          });
        } catch (error) {
          console.error(`Error in message handler for ${message.type}:`, error);
          
          if (message.requestId) {
            this.respond(
              event.source as Window, 
              message.requestId, 
              null, 
              error instanceof Error ? error.message : 'Handler error'
            );
          }
        }
      });
    }
  }

  private handleResponse(message: MessagePayload): void {
    const pending = this.pendingRequests.get(message.requestId!);
    if (pending) {
      clearTimeout(pending.timeout);
      this.pendingRequests.delete(message.requestId!);
      
      if (message.error) {
        pending.reject(new Error(message.error));
      } else {
        pending.resolve(message.data);
      }
    }
  }

  private isValidMessage(message: any): message is MessagePayload {
    return (
      message &&
      typeof message.type === 'string' &&
      typeof message.source === 'string' &&
      typeof message.target === 'string'
    );
  }

  destroy(): void {
    window.removeEventListener('message', this.handleMessage);
    this.messageHandlers.clear();
    this.pendingRequests.forEach(({ timeout }) => clearTimeout(timeout));
    this.pendingRequests.clear();
    this.isInitialized = false;
  }
}

// Usage example
export class MicrofrontendCommunication {
  private bridge: PostMessageBridge;
  private childWindows: Map<string, Window> = new Map();

  constructor() {
    this.bridge = new PostMessageBridge([
      'http://localhost:3001', // user-management
      'http://localhost:3002', // product-catalog
      'http://localhost:3003', // order-processing
    ]);

    this.setupMessageHandlers();
  }

  private setupMessageHandlers(): void {
    // Handle user authentication updates
    this.bridge.on('user:authenticated', (data, context) => {
      console.log('User authenticated:', data);
      this.broadcastToAllChildren('user:authenticated', data);
    });

    // Handle navigation requests
    this.bridge.on('navigate', (data, context) => {
      console.log('Navigation request:', data);
      window.history.pushState({}, '', data.path);
      context.respond({ success: true });
    });

    // Handle theme changes
    this.bridge.on('theme:changed', (data, context) => {
      console.log('Theme changed:', data);
      document.documentElement.setAttribute('data-theme', data.theme);
      this.broadcastToAllChildren('theme:changed', data);
    });

    // Handle errors
    this.bridge.on('error', (data, context) => {
      console.error('Child application error:', data);
      // Handle global error logging, notifications, etc.
    });
  }

  registerChildWindow(name: string, window: Window): void {
    this.childWindows.set(name, window);
  }

  unregisterChildWindow(name: string): void {
    this.childWindows.delete(name);
  }

  async sendToChild(childName: string, type: string, data: any): Promise<any> {
    const childWindow = this.childWindows.get(childName);
    if (!childWindow) {
      throw new Error(`Child window ${childName} not found`);
    }

    return this.bridge.request(childWindow, type, data, childName);
  }

  broadcastToAllChildren(type: string, data: any): void {
    this.childWindows.forEach((childWindow, name) => {
      this.bridge.send(childWindow, {
        type,
        data,
        target: name,
      });
    });
  }
}
```

## Deployment and DevOps

### Container-Based Deployment

```dockerfile
# Base image for all micro-frontends
FROM node:18-alpine as base

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Shell application
FROM base as shell-app
COPY shell-app/ ./
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]

# User management micro-frontend
FROM base as user-management
COPY user-management/ ./
RUN npm run build
EXPOSE 3001
CMD ["npm", "start"]

# Product catalog micro-frontend
FROM base as product-catalog
COPY product-catalog/ ./
RUN npm run build
EXPOSE 3002
CMD ["npm", "start"]

# Order processing micro-frontend
FROM base as order-processing
COPY order-processing/ ./
RUN npm run build
EXPOSE 3003
CMD ["npm", "start"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  shell-app:
    build:
      context: .
      target: shell-app
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - REACT_APP_USER_MANAGEMENT_URL=http://user-management:3001
      - REACT_APP_PRODUCT_CATALOG_URL=http://product-catalog:3002
      - REACT_APP_ORDER_PROCESSING_URL=http://order-processing:3003
    depends_on:
      - user-management
      - product-catalog
      - order-processing
    networks:
      - microfrontend-network

  user-management:
    build:
      context: .
      target: user-management
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - API_BASE_URL=http://api-gateway:8080
    networks:
      - microfrontend-network

  product-catalog:
    build:
      context: .
      target: product-catalog
    ports:
      - "3002:3002"
    environment:
      - NODE_ENV=production
      - API_BASE_URL=http://api-gateway:8080
    networks:
      - microfrontend-network

  order-processing:
    build:
      context: .
      target: order-processing
    ports:
      - "3003:3003"
    environment:
      - NODE_ENV=production
      - API_BASE_URL=http://api-gateway:8080
    networks:
      - microfrontend-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - shell-app
      - user-management
      - product-catalog
      - order-processing
    networks:
      - microfrontend-network

networks:
  microfrontend-network:
    driver: bridge
```

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream shell-app {
        server shell-app:3000;
    }

    upstream user-management {
        server user-management:3001;
    }

    upstream product-catalog {
        server product-catalog:3002;
    }

    upstream order-processing {
        server order-processing:3003;
    }

    server {
        listen 80;
        server_name localhost;

        # Security headers
        add_header X-Frame-Options SAMEORIGIN;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        # Main shell application
        location / {
            proxy_pass http://shell-app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Micro-frontend routes
        location /mf/user-management/ {
            rewrite ^/mf/user-management/(.*) /$1 break;
            proxy_pass http://user-management;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /mf/product-catalog/ {
            rewrite ^/mf/product-catalog/(.*) /$1 break;
            proxy_pass http://product-catalog;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /mf/order-processing/ {
            rewrite ^/mf/order-processing/(.*) /$1 break;
            proxy_pass http://order-processing;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Health checks
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

### Kubernetes Deployment

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microfrontends

---
# k8s/shell-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shell-app
  namespace: microfrontends
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shell-app
  template:
    metadata:
      labels:
        app: shell-app
    spec:
      containers:
      - name: shell-app
        image: microfrontends/shell-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: REACT_APP_USER_MANAGEMENT_URL
          value: "http://user-management-service:3001"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: shell-app-service
  namespace: microfrontends
spec:
  selector:
    app: shell-app
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microfrontends-ingress
  namespace: microfrontends
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - microfrontends.example.com
    secretName: microfrontends-tls
  rules:
  - host: microfrontends.example.com
    http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: shell-app-service
            port:
              number: 3000
      - path: /mf/user-management/(.*)
        pathType: Prefix
        backend:
          service:
            name: user-management-service
            port:
              number: 3001
      - path: /mf/product-catalog/(.*)
        pathType: Prefix
        backend:
          service:
            name: product-catalog-service
            port:
              number: 3002
      - path: /mf/order-processing/(.*)
        pathType: Prefix
        backend:
          service:
            name: order-processing-service
            port:
              number: 3003

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shell-app-hpa
  namespace: microfrontends
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shell-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Interview-Ready Micro-frontends Summary

**Micro-frontends Architecture:**
1. **Distributed Frontend** - Independent deployment and development of frontend applications
2. **Module Federation** - Webpack-based runtime sharing of modules and dependencies
3. **Single-SPA** - Framework-agnostic orchestration of multiple applications
4. **Communication Patterns** - Event-driven, shared state, and postMessage-based integration

**Advanced Implementation Patterns:**
- Dynamic remote loading with error handling and fallbacks
- Cross-domain communication with security considerations
- Shared state management across independent applications
- Runtime configuration and feature flagging

**Enterprise Deployment:**
- Container-based deployment with Docker and Kubernetes
- Load balancing and service mesh integration
- Security boundaries and CORS handling
- Monitoring and observability across distributed frontends

**Key Interview Topics:** Micro-frontend architectural patterns, Module Federation implementation, communication strategies, deployment considerations, security challenges, performance optimization, team organization benefits, and enterprise scaling strategies.