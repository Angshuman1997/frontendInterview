# BFF with Micro-frontends Integration

The integration of Backend For Frontend (BFF) with micro-frontends architecture creates a powerful pattern for building scalable, maintainable enterprise applications. This guide covers advanced integration patterns, module federation strategies, and production deployment approaches.

## Micro-frontends with BFF Architecture

### 1. Advanced BFF-Micro-frontend Integration
```typescript
// Multi-BFF Architecture for Micro-frontends
import express, { Request, Response } from 'express';
import { ModuleFederationPlugin } from '@module-federation/webpack';
import { SingleSpaApplicationLifecycles, registerApplication } from 'single-spa';

// BFF Orchestrator for Micro-frontends
interface MicroFrontendConfig {
  name: string;
  entry: string;
  activeWhen: string | ((location: Location) => boolean);
  bffEndpoint: string;
  customProps?: Record<string, any>;
}

interface BFFOrchestrationConfig {
  applications: MicroFrontendConfig[];
  sharedBFF: {
    auth: string;
    navigation: string;
    notifications: string;
  };
  routing: {
    strategy: 'path-based' | 'subdomain-based' | 'header-based';
    fallback: string;
  };
}

class BFFOrchestrator {
  private applications: Map<string, MicroFrontendConfig>;
  private sharedServices: Map<string, express.Application>;
  private routingStrategy: string;

  constructor(private config: BFFOrchestrationConfig) {
    this.applications = new Map();
    this.sharedServices = new Map();
    this.routingStrategy = config.routing.strategy;
    this.initializeApplications();
    this.setupSharedServices();
  }

  // Initialize micro-frontend applications with their BFFs
  private initializeApplications(): void {
    this.config.applications.forEach(app => {
      this.applications.set(app.name, app);
      
      // Register each micro-frontend with single-spa
      registerApplication({
        name: app.name,
        app: () => this.loadMicroFrontend(app),
        activeWhen: app.activeWhen,
        customProps: {
          bffEndpoint: app.bffEndpoint,
          ...app.customProps
        }
      });
    });
  }

  // Load micro-frontend with BFF context
  private async loadMicroFrontend(config: MicroFrontendConfig): Promise<SingleSpaApplicationLifecycles> {
    const module = await import(config.entry);
    
    // Inject BFF client into micro-frontend
    const bffClient = new BFFClient({
      baseURL: config.bffEndpoint,
      interceptors: {
        request: this.addSharedContext.bind(this),
        response: this.handleSharedErrors.bind(this)
      }
    });

    return {
      ...module.default,
      mount: (props: any) => {
        return module.default.mount({
          ...props,
          bffClient,
          sharedServices: this.getSharedServices()
        });
      }
    };
  }

  // Setup shared BFF services
  private setupSharedServices(): void {
    // Authentication BFF
    const authBFF = this.createSharedBFF('auth', {
      services: ['auth-service', 'user-service'],
      cache: { ttl: 300, strategy: 'redis' },
      routes: [
        { path: '/login', method: 'POST' },
        { path: '/logout', method: 'POST' },
        { path: '/refresh', method: 'POST' },
        { path: '/profile', method: 'GET' }
      ]
    });

    // Navigation BFF
    const navigationBFF = this.createSharedBFF('navigation', {
      services: ['menu-service', 'permission-service'],
      cache: { ttl: 600, strategy: 'memory' },
      routes: [
        { path: '/menu', method: 'GET' },
        { path: '/permissions', method: 'GET' },
        { path: '/breadcrumbs', method: 'GET' }
      ]
    });

    // Notifications BFF
    const notificationsBFF = this.createSharedBFF('notifications', {
      services: ['notification-service', 'websocket-service'],
      cache: { ttl: 60, strategy: 'none' },
      routes: [
        { path: '/notifications', method: 'GET' },
        { path: '/notifications/mark-read', method: 'POST' },
        { path: '/notifications/subscribe', method: 'POST' }
      ]
    });

    this.sharedServices.set('auth', authBFF);
    this.sharedServices.set('navigation', navigationBFF);
    this.sharedServices.set('notifications', notificationsBFF);
  }

  private createSharedBFF(name: string, config: any): express.Application {
    const app = express();
    
    // Add middleware
    app.use(express.json());
    app.use(this.corsForMicroFrontends());
    app.use(this.authenticationMiddleware());
    
    // Add routes based on configuration
    config.routes.forEach((route: any) => {
      app[route.method.toLowerCase()](route.path, async (req: Request, res: Response) => {
        try {
          const result = await this.handleSharedBFFRequest(name, route.path, req);
          res.json(result);
        } catch (error) {
          res.status(500).json({ error: error.message });
        }
      });
    });

    return app;
  }

  private corsForMicroFrontends() {
    return (req: Request, res: Response, next: any) => {
      const allowedOrigins = this.config.applications.map(app => app.entry);
      const origin = req.headers.origin;
      
      if (allowedOrigins.includes(origin || '')) {
        res.setHeader('Access-Control-Allow-Origin', origin || '');
      }
      
      res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
      res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Micro-Frontend');
      res.setHeader('Access-Control-Allow-Credentials', 'true');
      
      next();
    };
  }

  private addSharedContext(config: any): any {
    // Add shared context to all BFF requests
    return {
      ...config,
      headers: {
        ...config.headers,
        'X-Orchestrator': 'bff-orchestrator',
        'X-Session-Id': this.getCurrentSessionId(),
        'X-User-Context': this.getCurrentUserContext()
      }
    };
  }

  private handleSharedErrors(error: any): void {
    // Centralized error handling for all micro-frontends
    if (error.response?.status === 401) {
      this.handleAuthenticationError();
    } else if (error.response?.status >= 500) {
      this.handleSystemError(error);
    }
  }

  private getCurrentSessionId(): string {
    // Implementation to get current session ID
    return 'session-id-123';
  }

  private getCurrentUserContext(): string {
    // Implementation to get current user context
    return JSON.stringify({ userId: 'user-123', roles: ['user'] });
  }

  private handleAuthenticationError(): void {
    // Redirect to login or refresh token
    window.location.href = '/login';
  }

  private handleSystemError(error: any): void {
    // Log error and show user-friendly message
    console.error('System error:', error);
    // Show global error notification
  }

  private async handleSharedBFFRequest(service: string, path: string, req: Request): Promise<any> {
    // Implementation of shared BFF request handling
    return { message: `Handled ${service}${path}` };
  }

  private getSharedServices(): Record<string, any> {
    return {
      auth: this.sharedServices.get('auth'),
      navigation: this.sharedServices.get('navigation'),
      notifications: this.sharedServices.get('notifications')
    };
  }
}

// BFF Client for Micro-frontends
class BFFClient {
  private baseURL: string;
  private interceptors: any;

  constructor(config: { baseURL: string; interceptors?: any }) {
    this.baseURL = config.baseURL;
    this.interceptors = config.interceptors || {};
  }

  async get<T>(path: string, options: any = {}): Promise<T> {
    const config = this.interceptors.request?.({ 
      method: 'GET', 
      url: `${this.baseURL}${path}`,
      ...options 
    }) || { method: 'GET', url: `${this.baseURL}${path}`, ...options };

    try {
      const response = await fetch(config.url, {
        method: config.method,
        headers: config.headers,
        credentials: 'include'
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      this.interceptors.response?.(error);
      throw error;
    }
  }

  async post<T>(path: string, data: any, options: any = {}): Promise<T> {
    const config = this.interceptors.request?.({
      method: 'POST',
      url: `${this.baseURL}${path}`,
      data,
      ...options
    }) || { method: 'POST', url: `${this.baseURL}${path}`, data, ...options };

    try {
      const response = await fetch(config.url, {
        method: config.method,
        headers: {
          'Content-Type': 'application/json',
          ...config.headers
        },
        body: JSON.stringify(config.data),
        credentials: 'include'
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      this.interceptors.response?.(error);
      throw error;
    }
  }
}
```

### 2. Module Federation with BFF Integration
```typescript
// Webpack Module Federation Configuration for BFF-enabled Micro-frontends
import { ModuleFederationPlugin } from '@module-federation/webpack';

// Shell Application Configuration
const shellWebpackConfig = {
  mode: 'development',
  entry: './src/index.ts',
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        dashboard: 'dashboard@http://localhost:3001/remoteEntry.js',
        profile: 'profile@http://localhost:3002/remoteEntry.js',
        analytics: 'analytics@http://localhost:3003/remoteEntry.js',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        '@bff-client/core': { singleton: true },
        '@shared-components/ui': { singleton: true },
      },
    }),
  ],
};

// Micro-frontend Configuration (Dashboard)
const dashboardWebpackConfig = {
  mode: 'development',
  entry: './src/index.ts',
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'dashboard',
      filename: 'remoteEntry.js',
      exposes: {
        './Dashboard': './src/Dashboard',
        './DashboardBFF': './src/bff/dashboard-bff-client',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        '@bff-client/core': { singleton: true },
      },
    }),
  ],
};

// Dashboard Micro-frontend with BFF Integration
import React, { useEffect, useState } from 'react';
import { BFFClient } from '@bff-client/core';

interface DashboardProps {
  bffClient: BFFClient;
  sharedServices: any;
}

interface DashboardData {
  widgets: Widget[];
  metrics: Metric[];
  notifications: Notification[];
}

interface Widget {
  id: string;
  type: 'chart' | 'table' | 'summary';
  title: string;
  data: any;
  config: WidgetConfig;
}

interface WidgetConfig {
  refreshInterval?: number;
  interactive?: boolean;
  filters?: Filter[];
}

const Dashboard: React.FC<DashboardProps> = ({ bffClient, sharedServices }) => {
  const [dashboardData, setDashboardData] = useState<DashboardData | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      setLoading(true);
      setError(null);

      // Load dashboard-specific data from BFF
      const data = await bffClient.get<DashboardData>('/dashboard', {
        headers: {
          'X-Micro-Frontend': 'dashboard',
          'X-Viewport': getViewportSize()
        }
      });

      setDashboardData(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load dashboard');
    } finally {
      setLoading(false);
    }
  };

  const handleWidgetRefresh = async (widgetId: string) => {
    try {
      const updatedWidget = await bffClient.get<Widget>(`/dashboard/widgets/${widgetId}/refresh`);
      
      setDashboardData(prev => prev ? {
        ...prev,
        widgets: prev.widgets.map(w => w.id === widgetId ? updatedWidget : w)
      } : null);
    } catch (err) {
      console.error('Failed to refresh widget:', err);
    }
  };

  const getViewportSize = (): string => {
    const width = window.innerWidth;
    if (width < 768) return 'mobile';
    if (width < 1024) return 'tablet';
    return 'desktop';
  };

  if (loading) {
    return <DashboardSkeleton />;
  }

  if (error) {
    return <ErrorBoundary error={error} onRetry={loadDashboardData} />;
  }

  return (
    <div className="dashboard">
      <DashboardHeader 
        notifications={dashboardData?.notifications || []}
        sharedServices={sharedServices}
      />
      
      <div className="dashboard-grid">
        {dashboardData?.widgets.map(widget => (
          <WidgetContainer
            key={widget.id}
            widget={widget}
            onRefresh={() => handleWidgetRefresh(widget.id)}
            bffClient={bffClient}
          />
        ))}
      </div>
      
      <DashboardMetrics metrics={dashboardData?.metrics || []} />
    </div>
  );
};

// Widget Container with BFF Integration
const WidgetContainer: React.FC<{
  widget: Widget;
  onRefresh: () => void;
  bffClient: BFFClient;
}> = ({ widget, onRefresh, bffClient }) => {
  const [widgetData, setWidgetData] = useState(widget.data);
  const [refreshing, setRefreshing] = useState(false);

  useEffect(() => {
    if (widget.config.refreshInterval) {
      const interval = setInterval(() => {
        handleAutoRefresh();
      }, widget.config.refreshInterval * 1000);

      return () => clearInterval(interval);
    }
  }, [widget.config.refreshInterval]);

  const handleAutoRefresh = async () => {
    try {
      setRefreshing(true);
      const updatedData = await bffClient.get(`/dashboard/widgets/${widget.id}/data`);
      setWidgetData(updatedData);
    } catch (error) {
      console.error('Auto-refresh failed:', error);
    } finally {
      setRefreshing(false);
    }
  };

  return (
    <div className={`widget widget-${widget.type}`}>
      <div className="widget-header">
        <h3>{widget.title}</h3>
        <div className="widget-actions">
          {refreshing && <Spinner size="small" />}
          <button onClick={onRefresh} disabled={refreshing}>
            Refresh
          </button>
        </div>
      </div>
      
      <div className="widget-content">
        {widget.type === 'chart' && <ChartWidget data={widgetData} />}
        {widget.type === 'table' && <TableWidget data={widgetData} />}
        {widget.type === 'summary' && <SummaryWidget data={widgetData} />}
      </div>
    </div>
  );
};

// BFF Client Factory for Micro-frontends
class MicroFrontendBFFFactory {
  static createBFFClient(microFrontendName: string, config: any): BFFClient {
    const baseURL = config.bffEndpoints[microFrontendName] || config.defaultBFFEndpoint;
    
    return new BFFClient({
      baseURL,
      interceptors: {
        request: (requestConfig: any) => ({
          ...requestConfig,
          headers: {
            ...requestConfig.headers,
            'X-Micro-Frontend': microFrontendName,
            'X-Version': config.version || '1.0.0',
            'X-Build': config.buildId || 'unknown'
          }
        }),
        response: (error: any) => {
          if (error.response?.status === 404) {
            // Handle micro-frontend specific 404s
            console.warn(`Resource not found in ${microFrontendName} BFF:`, error);
          }
        }
      }
    });
  }
}
```

### 3. Deployment and Orchestration
```typescript
// Docker Compose for BFF + Micro-frontends
// docker-compose.yml
version: '3.8'
services:
  # Shared BFF Services
  auth-bff:
    build: ./bff/auth
    ports:
      - "3010:3000"
    environment:
      - NODE_ENV=production
      - REDIS_URL=redis://redis:6379
      - AUTH_SERVICE_URL=http://auth-service:3000
    depends_on:
      - redis
      - auth-service

  navigation-bff:
    build: ./bff/navigation
    ports:
      - "3011:3000"
    environment:
      - NODE_ENV=production
      - MENU_SERVICE_URL=http://menu-service:3000
    depends_on:
      - menu-service

  # Micro-frontend Specific BFFs
  dashboard-bff:
    build: ./bff/dashboard
    ports:
      - "3020:3000"
    environment:
      - NODE_ENV=production
      - ANALYTICS_SERVICE_URL=http://analytics-service:3000
      - METRICS_SERVICE_URL=http://metrics-service:3000
    depends_on:
      - analytics-service
      - metrics-service

  profile-bff:
    build: ./bff/profile
    ports:
      - "3021:3000"
    environment:
      - NODE_ENV=production
      - USER_SERVICE_URL=http://user-service:3000
      - PREFERENCES_SERVICE_URL=http://preferences-service:3000

  # Micro-frontend Applications
  shell-app:
    build: ./apps/shell
    ports:
      - "3000:80"
    environment:
      - REACT_APP_DASHBOARD_URL=http://localhost:3001
      - REACT_APP_PROFILE_URL=http://localhost:3002
    depends_on:
      - dashboard-app
      - profile-app

  dashboard-app:
    build: ./apps/dashboard
    ports:
      - "3001:80"
    environment:
      - REACT_APP_BFF_URL=http://dashboard-bff:3000

  profile-app:
    build: ./apps/profile
    ports:
      - "3002:80"
    environment:
      - REACT_APP_BFF_URL=http://profile-bff:3000

  # Supporting Services
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - shell-app
      - dashboard-app
      - profile-app
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf

// Kubernetes Deployment for Production
// k8s-bff-microfrontends.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dashboard-bff
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dashboard-bff
  template:
    metadata:
      labels:
        app: dashboard-bff
    spec:
      containers:
      - name: dashboard-bff
        image: myregistry/dashboard-bff:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: ANALYTICS_SERVICE_URL
          value: "http://analytics-service:3000"
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: dashboard-bff-service
spec:
  selector:
    app: dashboard-bff
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microfrontend-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shell-app-service
            port:
              number: 80
      - path: /bff/dashboard
        pathType: Prefix
        backend:
          service:
            name: dashboard-bff-service
            port:
              number: 80
      - path: /bff/profile
        pathType: Prefix
        backend:
          service:
            name: profile-bff-service
            port:
              number: 80

// NGINX Configuration for Micro-frontends
// nginx.conf
upstream shell_app {
    server shell-app:80;
}

upstream dashboard_app {
    server dashboard-app:80;
}

upstream profile_app {
    server profile-app:80;
}

upstream dashboard_bff {
    server dashboard-bff:3000;
}

upstream profile_bff {
    server profile-bff:3000;
}

server {
    listen 80;
    server_name localhost;

    # Shell application (main entry point)
    location / {
        proxy_pass http://shell_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Micro-frontend applications
    location /mf/dashboard {
        proxy_pass http://dashboard_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /mf/profile {
        proxy_pass http://profile_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # BFF APIs
    location /api/dashboard {
        proxy_pass http://dashboard_bff;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api/profile {
        proxy_pass http://profile_bff;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # CORS headers for micro-frontends
    add_header Access-Control-Allow-Origin $http_origin;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
    add_header Access-Control-Allow-Headers "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Micro-Frontend";
    add_header Access-Control-Expose-Headers "Content-Length,Content-Range";
}
```

## Interview-Ready Summary

**BFF + Micro-frontends Integration provides:**

1. **Dedicated BFFs** - Each micro-frontend gets optimized backend integration
2. **Shared Services** - Common BFFs for authentication, navigation, notifications
3. **Module Federation** - Seamless integration with Webpack Module Federation
4. **Orchestration** - Centralized routing and cross-cutting concerns
5. **Independent Deployment** - Each micro-frontend and BFF can deploy independently

**Key Patterns:**
- **BFF Orchestrator** - Manages multiple BFFs and shared services
- **Client Factory** - Standardized BFF client creation per micro-frontend
- **Shared Context** - Common authentication and user context across apps
- **Error Boundaries** - Graceful degradation when BFF services fail

**Production Benefits:**
- **Team Independence** - Frontend teams own their BFF and micro-frontend
- **Technology Diversity** - Different teams can use different tech stacks
- **Scalability** - Independent scaling of micro-frontends and their BFFs
- **Fault Isolation** - Failures in one micro-frontend don't affect others

**Deployment Strategies:** Docker Compose for development, Kubernetes for production, NGINX for routing, comprehensive monitoring and health checks across all services.