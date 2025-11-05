# Advanced Backend For Frontend (BFF) Architecture

Backend For Frontend (BFF) is a sophisticated architectural pattern that creates dedicated backend services optimized for specific frontend applications or user experiences. This approach addresses the limitations of generic APIs by providing tailored data aggregation, transformation, and business logic.

## BFF Pattern Fundamentals

### 1. Core BFF Architecture Implementation
```typescript
// BFF Server Architecture with Express.js and TypeScript

import express, { Request, Response, NextFunction } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import compression from 'compression';
import { createProxyMiddleware } from 'http-proxy-middleware';

// Interface definitions for type safety
interface UserProfile {
  id: string;
  name: string;
  email: string;
  avatar?: string;
  preferences: UserPreferences;
}

interface UserPreferences {
  theme: 'light' | 'dark';
  language: string;
  notifications: NotificationSettings;
}

interface NotificationSettings {
  email: boolean;
  push: boolean;
  sms: boolean;
}

interface DashboardData {
  user: UserProfile;
  recentActivity: Activity[];
  analytics: AnalyticsData;
  notifications: Notification[];
  quickActions: QuickAction[];
}

// BFF Service Class for Business Logic
class BFFService {
  private userService: UserServiceClient;
  private analyticsService: AnalyticsServiceClient;
  private notificationService: NotificationServiceClient;
  private activityService: ActivityServiceClient;
  private cacheManager: CacheManager;

  constructor() {
    this.userService = new UserServiceClient(process.env.USER_SERVICE_URL!);
    this.analyticsService = new AnalyticsServiceClient(process.env.ANALYTICS_SERVICE_URL!);
    this.notificationService = new NotificationServiceClient(process.env.NOTIFICATION_SERVICE_URL!);
    this.activityService = new ActivityServiceClient(process.env.ACTIVITY_SERVICE_URL!);
    this.cacheManager = new CacheManager();
  }

  // Aggregated dashboard data for web frontend
  async getDashboardData(userId: string, platform: 'web' | 'mobile'): Promise<DashboardData> {
    const cacheKey = `dashboard:${userId}:${platform}`;
    
    // Check cache first
    const cached = await this.cacheManager.get<DashboardData>(cacheKey);
    if (cached) {
      return cached;
    }

    // Parallel data fetching with error handling
    const [
      userResult,
      activityResult,
      analyticsResult,
      notificationsResult
    ] = await Promise.allSettled([
      this.userService.getUserProfile(userId),
      this.activityService.getRecentActivity(userId, { limit: platform === 'mobile' ? 5 : 10 }),
      this.analyticsService.getUserAnalytics(userId, { timeframe: '30d' }),
      this.notificationService.getUnreadNotifications(userId, { limit: 5 })
    ]);

    // Handle partial failures gracefully
    const dashboardData: DashboardData = {
      user: userResult.status === 'fulfilled' ? userResult.value : this.getDefaultUserProfile(userId),
      recentActivity: activityResult.status === 'fulfilled' ? activityResult.value : [],
      analytics: analyticsResult.status === 'fulfilled' ? analyticsResult.value : this.getDefaultAnalytics(),
      notifications: notificationsResult.status === 'fulfilled' ? notificationsResult.value : [],
      quickActions: this.getQuickActionsForPlatform(platform)
    };

    // Transform data based on platform requirements
    const transformedData = this.transformDataForPlatform(dashboardData, platform);

    // Cache the result
    await this.cacheManager.set(cacheKey, transformedData, 300); // 5 minutes

    return transformedData;
  }

  // Platform-specific data transformation
  private transformDataForPlatform(data: DashboardData, platform: 'web' | 'mobile'): DashboardData {
    if (platform === 'mobile') {
      return {
        ...data,
        // Reduce data payload for mobile
        recentActivity: data.recentActivity.slice(0, 5),
        analytics: {
          ...data.analytics,
          // Simplified analytics for mobile
          details: undefined
        },
        // Mobile-optimized quick actions
        quickActions: data.quickActions.filter(action => action.mobile !== false)
      };
    }

    return data;
  }

  // Optimized search with aggregation
  async searchContent(
    query: string,
    filters: SearchFilters,
    userId: string,
    platform: string
  ): Promise<SearchResults> {
    const searchPromises = [];

    // Parallel search across multiple services
    if (filters.includeUsers) {
      searchPromises.push(
        this.userService.searchUsers(query, {
          limit: platform === 'mobile' ? 5 : 10,
          excludeCurrentUser: true
        }).then(results => ({ type: 'users', data: results }))
      );
    }

    if (filters.includeContent) {
      searchPromises.push(
        this.contentService.searchContent(query, {
          userId,
          limit: platform === 'mobile' ? 8 : 20,
          includeMetadata: platform === 'web'
        }).then(results => ({ type: 'content', data: results }))
      );
    }

    if (filters.includeDocuments) {
      searchPromises.push(
        this.documentService.searchDocuments(query, {
          userId,
          limit: platform === 'mobile' ? 5 : 15
        }).then(results => ({ type: 'documents', data: results }))
      );
    }

    const searchResults = await Promise.allSettled(searchPromises);
    
    // Aggregate and rank results
    const aggregatedResults: SearchResults = {
      query,
      total: 0,
      results: [],
      facets: {},
      suggestions: []
    };

    searchResults.forEach(result => {
      if (result.status === 'fulfilled') {
        const { type, data } = result.value;
        aggregatedResults.results.push(...data.results.map(item => ({
          ...item,
          source: type
        })));
        aggregatedResults.total += data.total;
      }
    });

    // Sort by relevance score
    aggregatedResults.results.sort((a, b) => (b.score || 0) - (a.score || 0));

    // Limit final results
    const maxResults = platform === 'mobile' ? 15 : 50;
    aggregatedResults.results = aggregatedResults.results.slice(0, maxResults);

    return aggregatedResults;
  }

  // Batch operations for efficiency
  async batchUpdateUserData(
    userId: string,
    updates: BatchUserUpdates
  ): Promise<BatchUpdateResults> {
    const updatePromises: Promise<any>[] = [];

    if (updates.profile) {
      updatePromises.push(
        this.userService.updateProfile(userId, updates.profile)
          .then(result => ({ type: 'profile', success: true, data: result }))
          .catch(error => ({ type: 'profile', success: false, error: error.message }))
      );
    }

    if (updates.preferences) {
      updatePromises.push(
        this.userService.updatePreferences(userId, updates.preferences)
          .then(result => ({ type: 'preferences', success: true, data: result }))
          .catch(error => ({ type: 'preferences', success: false, error: error.message }))
      );
    }

    if (updates.notifications) {
      updatePromises.push(
        this.notificationService.updateSettings(userId, updates.notifications)
          .then(result => ({ type: 'notifications', success: true, data: result }))
          .catch(error => ({ type: 'notifications', success: false, error: error.message }))
      );
    }

    const results = await Promise.allSettled(updatePromises);
    
    return {
      success: results.every(r => r.status === 'fulfilled' && r.value.success),
      results: results.map(r => r.status === 'fulfilled' ? r.value : { success: false, error: 'Unknown error' }),
      timestamp: new Date().toISOString()
    };
  }

  private getDefaultUserProfile(userId: string): UserProfile {
    return {
      id: userId,
      name: 'Unknown User',
      email: '',
      preferences: {
        theme: 'light',
        language: 'en',
        notifications: {
          email: true,
          push: true,
          sms: false
        }
      }
    };
  }

  private getDefaultAnalytics(): AnalyticsData {
    return {
      pageViews: 0,
      uniqueVisitors: 0,
      bounceRate: 0,
      averageSessionDuration: 0,
      topPages: [],
      timeframe: '30d'
    };
  }

  private getQuickActionsForPlatform(platform: 'web' | 'mobile'): QuickAction[] {
    const baseActions: QuickAction[] = [
      { id: 'create', label: 'Create New', icon: 'plus', mobile: true },
      { id: 'search', label: 'Search', icon: 'search', mobile: true },
      { id: 'notifications', label: 'Notifications', icon: 'bell', mobile: true }
    ];

    if (platform === 'web') {
      baseActions.push(
        { id: 'analytics', label: 'Analytics', icon: 'chart', mobile: false },
        { id: 'settings', label: 'Settings', icon: 'cog', mobile: false },
        { id: 'export', label: 'Export Data', icon: 'download', mobile: false }
      );
    }

    return baseActions;
  }
}

// Express BFF Application Setup
class BFFApplication {
  private app: express.Application;
  private bffService: BFFService;

  constructor() {
    this.app = express();
    this.bffService = new BFFService();
    this.setupMiddleware();
    this.setupRoutes();
    this.setupErrorHandling();
  }

  private setupMiddleware(): void {
    // Security middleware
    this.app.use(helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'"],
          scriptSrc: ["'self'"],
          imgSrc: ["'self'", "data:", "https:"],
        },
      },
    }));

    // CORS configuration
    this.app.use(cors({
      origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
      credentials: true,
      methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
      allowedHeaders: ['Content-Type', 'Authorization', 'X-Platform', 'X-Client-Version']
    }));

    // Rate limiting
    this.app.use(rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 1000, // limit each IP to 1000 requests per windowMs
      message: 'Too many requests from this IP',
      standardHeaders: true,
      legacyHeaders: false,
    }));

    // Compression and parsing
    this.app.use(compression());
    this.app.use(express.json({ limit: '10mb' }));
    this.app.use(express.urlencoded({ extended: true, limit: '10mb' }));

    // Request logging and tracing
    this.app.use((req: Request, res: Response, next: NextFunction) => {
      const traceId = req.headers['x-trace-id'] || generateTraceId();
      req.headers['x-trace-id'] = traceId;
      res.setHeader('x-trace-id', traceId);
      
      console.log(`[${new Date().toISOString()}] ${req.method} ${req.path} - ${req.ip} - ${traceId}`);
      next();
    });

    // Authentication middleware
    this.app.use('/api', this.authenticateUser);
  }

  private setupRoutes(): void {
    // Health check
    this.app.get('/health', (req: Request, res: Response) => {
      res.json({
        status: 'healthy',
        timestamp: new Date().toISOString(),
        version: process.env.APP_VERSION || '1.0.0'
      });
    });

    // Dashboard endpoint - aggregated data
    this.app.get('/api/dashboard', async (req: Request, res: Response, next: NextFunction) => {
      try {
        const userId = req.user?.id;
        const platform = req.headers['x-platform'] as 'web' | 'mobile' || 'web';
        
        const dashboardData = await this.bffService.getDashboardData(userId, platform);
        
        res.json({
          success: true,
          data: dashboardData,
          meta: {
            platform,
            timestamp: new Date().toISOString()
          }
        });
      } catch (error) {
        next(error);
      }
    });

    // Search endpoint - aggregated search across services
    this.app.get('/api/search', async (req: Request, res: Response, next: NextFunction) => {
      try {
        const { q: query, ...filters } = req.query;
        const userId = req.user?.id;
        const platform = req.headers['x-platform'] as string || 'web';
        
        if (!query || typeof query !== 'string') {
          return res.status(400).json({
            success: false,
            error: 'Query parameter is required'
          });
        }

        const searchResults = await this.bffService.searchContent(
          query,
          filters as SearchFilters,
          userId,
          platform
        );
        
        res.json({
          success: true,
          data: searchResults
        });
      } catch (error) {
        next(error);
      }
    });

    // Batch update endpoint
    this.app.patch('/api/user/batch', async (req: Request, res: Response, next: NextFunction) => {
      try {
        const userId = req.user?.id;
        const updates = req.body as BatchUserUpdates;
        
        const results = await this.bffService.batchUpdateUserData(userId, updates);
        
        res.json({
          success: results.success,
          data: results
        });
      } catch (error) {
        next(error);
      }
    });

    // Proxy to microservices with transformation
    this.setupServiceProxies();
  }

  private setupServiceProxies(): void {
    // User service proxy with data transformation
    this.app.use('/api/users', createProxyMiddleware({
      target: process.env.USER_SERVICE_URL,
      changeOrigin: true,
      pathRewrite: {
        '^/api/users': '/users'
      },
      onProxyReq: (proxyReq, req, res) => {
        // Add authentication headers
        if (req.user?.token) {
          proxyReq.setHeader('Authorization', `Bearer ${req.user.token}`);
        }
      },
      onProxyRes: (proxyRes, req, res) => {
        // Transform response based on client platform
        const platform = req.headers['x-platform'];
        if (platform === 'mobile') {
          // Add mobile-specific response headers
          proxyRes.headers['x-mobile-optimized'] = 'true';
        }
      }
    }));

    // Analytics service proxy with aggregation
    this.app.use('/api/analytics', createProxyMiddleware({
      target: process.env.ANALYTICS_SERVICE_URL,
      changeOrigin: true,
      pathRewrite: {
        '^/api/analytics': '/analytics'
      },
      onProxyReq: (proxyReq, req, res) => {
        proxyReq.setHeader('Authorization', `Bearer ${req.user?.token}`);
        proxyReq.setHeader('X-User-ID', req.user?.id);
      }
    }));
  }

  private async authenticateUser(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const token = req.headers.authorization?.replace('Bearer ', '');
      
      if (!token) {
        res.status(401).json({ success: false, error: 'Authentication required' });
        return;
      }

      // Validate token with auth service
      const user = await validateToken(token);
      
      if (!user) {
        res.status(401).json({ success: false, error: 'Invalid token' });
        return;
      }

      req.user = user;
      next();
    } catch (error) {
      res.status(401).json({ success: false, error: 'Authentication failed' });
    }
  }

  private setupErrorHandling(): void {
    // 404 handler
    this.app.use((req: Request, res: Response) => {
      res.status(404).json({
        success: false,
        error: 'Endpoint not found',
        path: req.path
      });
    });

    // Global error handler
    this.app.use((error: Error, req: Request, res: Response, next: NextFunction) => {
      console.error(`[Error] ${req.method} ${req.path}:`, error);

      const isDevelopment = process.env.NODE_ENV === 'development';
      
      res.status(500).json({
        success: false,
        error: 'Internal server error',
        ...(isDevelopment && { details: error.message, stack: error.stack }),
        traceId: req.headers['x-trace-id']
      });
    });
  }

  public start(port: number = 3001): void {
    this.app.listen(port, () => {
      console.log(`BFF Server running on port ${port}`);
      console.log(`Environment: ${process.env.NODE_ENV}`);
      console.log(`Health check: http://localhost:${port}/health`);
    });
  }
}

// Service client implementations
class UserServiceClient {
  constructor(private baseUrl: string) {}

  async getUserProfile(userId: string): Promise<UserProfile> {
    const response = await fetch(`${this.baseUrl}/users/${userId}`, {
      headers: this.getAuthHeaders()
    });
    
    if (!response.ok) {
      throw new Error(`Failed to fetch user profile: ${response.statusText}`);
    }
    
    return response.json();
  }

  async updateProfile(userId: string, updates: Partial<UserProfile>): Promise<UserProfile> {
    const response = await fetch(`${this.baseUrl}/users/${userId}`, {
      method: 'PATCH',
      headers: {
        ...this.getAuthHeaders(),
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(updates)
    });
    
    if (!response.ok) {
      throw new Error(`Failed to update user profile: ${response.statusText}`);
    }
    
    return response.json();
  }

  async searchUsers(query: string, options: SearchOptions): Promise<SearchResult<UserProfile>> {
    const params = new URLSearchParams({
      q: query,
      limit: options.limit?.toString() || '10',
      ...(options.excludeCurrentUser && { excludeCurrent: 'true' })
    });

    const response = await fetch(`${this.baseUrl}/users/search?${params}`, {
      headers: this.getAuthHeaders()
    });
    
    return response.json();
  }

  private getAuthHeaders(): Record<string, string> {
    return {
      'Authorization': `Bearer ${process.env.SERVICE_TOKEN}`,
      'X-Service': 'bff'
    };
  }
}

// Cache manager for performance optimization
class CacheManager {
  private redis: any; // Redis client

  constructor() {
    // Initialize Redis connection
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379'),
      password: process.env.REDIS_PASSWORD,
      retryDelayOnFailover: 100,
      maxRetriesPerRequest: 3
    });
  }

  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await this.redis.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    try {
      await this.redis.setex(key, ttlSeconds, JSON.stringify(value));
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }

  async del(key: string): Promise<void> {
    try {
      await this.redis.del(key);
    } catch (error) {
      console.error('Cache delete error:', error);
    }
  }

  async invalidatePattern(pattern: string): Promise<void> {
    try {
      const keys = await this.redis.keys(pattern);
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    } catch (error) {
      console.error('Cache invalidation error:', error);
    }
  }
}

// Utility functions
function generateTraceId(): string {
  return Math.random().toString(36).substring(2, 15) + 
         Math.random().toString(36).substring(2, 15);
}

async function validateToken(token: string): Promise<AuthUser | null> {
  try {
    // Validate token with auth service
    const response = await fetch(`${process.env.AUTH_SERVICE_URL}/validate`, {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    });
    
    if (!response.ok) {
      return null;
    }
    
    return response.json();
  } catch (error) {
    console.error('Token validation error:', error);
    return null;
  }
}

// Type definitions
interface AuthUser {
  id: string;
  email: string;
  role: string;
  token: string;
}

interface SearchFilters {
  includeUsers?: boolean;
  includeContent?: boolean;
  includeDocuments?: boolean;
  category?: string;
  dateRange?: {
    start: string;
    end: string;
  };
}

interface SearchResults {
  query: string;
  total: number;
  results: SearchResultItem[];
  facets: Record<string, FacetValue[]>;
  suggestions: string[];
}

interface SearchResultItem {
  id: string;
  title: string;
  description: string;
  type: string;
  source: string;
  score?: number;
  metadata?: Record<string, any>;
}

interface BatchUserUpdates {
  profile?: Partial<UserProfile>;
  preferences?: Partial<UserPreferences>;
  notifications?: Partial<NotificationSettings>;
}

interface BatchUpdateResults {
  success: boolean;
  results: Array<{
    type: string;
    success: boolean;
    data?: any;
    error?: string;
  }>;
  timestamp: string;
}

// Application startup
if (require.main === module) {
  const app = new BFFApplication();
  const port = parseInt(process.env.PORT || '3001');
  app.start(port);
}

export { BFFApplication, BFFService };
```

### 2. GraphQL BFF Implementation
```typescript
// GraphQL BFF with Apollo Server
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';
import { buildSubgraphSchema } from '@apollo/subgraph';
import { gql } from 'graphql-tag';
import DataLoader from 'dataloader';

// GraphQL schema definition
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0",
          import: ["@key", "@requires", "@provides", "@external"])

  type Query {
    dashboard(platform: Platform = WEB): Dashboard
    search(query: String!, filters: SearchFilters): SearchResults
    user(id: ID!): User
  }

  type Mutation {
    updateUserBatch(input: BatchUpdateInput!): BatchUpdateResult
    createContent(input: CreateContentInput!): Content
  }

  enum Platform {
    WEB
    MOBILE
    TABLET
  }

  type Dashboard {
    user: User!
    recentActivity: [Activity!]!
    analytics: Analytics
    notifications: [Notification!]!
    quickActions: [QuickAction!]!
    widgets: [Widget!]!
  }

  type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
    avatar: String
    preferences: UserPreferences!
    analytics: UserAnalytics
  }

  type UserPreferences {
    theme: Theme!
    language: String!
    notifications: NotificationSettings!
    dashboard: DashboardSettings!
  }

  type Analytics {
    pageViews: Int!
    uniqueVisitors: Int!
    bounceRate: Float!
    averageSessionDuration: Int!
    topPages: [PageAnalytics!]!
    timeframe: String!
  }

  type SearchResults {
    query: String!
    total: Int!
    results: [SearchResult!]!
    facets: [Facet!]!
    suggestions: [String!]!
    executionTime: Float!
  }

  type SearchResult {
    id: ID!
    title: String!
    description: String
    type: SearchResultType!
    source: String!
    score: Float
    metadata: JSON
    highlights: [String!]
  }

  input SearchFilters {
    types: [SearchResultType!]
    dateRange: DateRangeInput
    categories: [String!]
    sortBy: SortOption = RELEVANCE
  }

  input BatchUpdateInput {
    profile: UserProfileInput
    preferences: UserPreferencesInput
    notifications: NotificationSettingsInput
  }

  type BatchUpdateResult {
    success: Boolean!
    results: [UpdateResult!]!
    errors: [UpdateError!]!
  }

  # Custom scalar for JSON data
  scalar JSON
  scalar DateTime
`;

// Resolvers with data loading optimization
const resolvers = {
  Query: {
    async dashboard(
      _: any,
      { platform }: { platform: 'WEB' | 'MOBILE' | 'TABLET' },
      { dataSources, user }: GraphQLContext
    ) {
      // Use DataLoader for efficient batch loading
      const [
        userProfile,
        recentActivity,
        analytics,
        notifications
      ] = await Promise.all([
        dataSources.userLoader.load(user.id),
        dataSources.activityLoader.load({ userId: user.id, platform }),
        dataSources.analyticsLoader.load({ userId: user.id, timeframe: '30d' }),
        dataSources.notificationLoader.load({ userId: user.id, unreadOnly: true })
      ]);

      return {
        user: userProfile,
        recentActivity: recentActivity.slice(0, platform === 'MOBILE' ? 5 : 10),
        analytics,
        notifications: notifications.slice(0, 5),
        quickActions: getQuickActionsForPlatform(platform),
        widgets: await getWidgetsForUser(user.id, platform, dataSources)
      };
    },

    async search(
      _: any,
      { query, filters }: { query: string; filters?: SearchFilters },
      { dataSources, user }: GraphQLContext
    ) {
      const startTime = Date.now();
      
      // Parallel search across different content types
      const searchPromises = [];
      
      if (!filters?.types || filters.types.includes('USER')) {
        searchPromises.push(
          dataSources.searchService.searchUsers(query, filters)
            .then(results => results.map(r => ({ ...r, type: 'USER' })))
        );
      }
      
      if (!filters?.types || filters.types.includes('CONTENT')) {
        searchPromises.push(
          dataSources.searchService.searchContent(query, filters, user.id)
            .then(results => results.map(r => ({ ...r, type: 'CONTENT' })))
        );
      }
      
      if (!filters?.types || filters.types.includes('DOCUMENT')) {
        searchPromises.push(
          dataSources.searchService.searchDocuments(query, filters, user.id)
            .then(results => results.map(r => ({ ...r, type: 'DOCUMENT' })))
        );
      }

      const searchResults = await Promise.allSettled(searchPromises);
      
      // Aggregate results
      const allResults = searchResults
        .filter(result => result.status === 'fulfilled')
        .flatMap(result => (result as PromiseFulfilledResult<any>).value);

      // Sort by relevance score
      allResults.sort((a, b) => (b.score || 0) - (a.score || 0));

      const executionTime = Date.now() - startTime;

      return {
        query,
        total: allResults.length,
        results: allResults,
        facets: await generateSearchFacets(allResults),
        suggestions: await generateSearchSuggestions(query),
        executionTime
      };
    }
  },

  Mutation: {
    async updateUserBatch(
      _: any,
      { input }: { input: BatchUpdateInput },
      { dataSources, user }: GraphQLContext
    ) {
      const updatePromises = [];
      
      if (input.profile) {
        updatePromises.push(
          dataSources.userService.updateProfile(user.id, input.profile)
            .then(result => ({ type: 'profile', success: true, data: result }))
            .catch(error => ({ type: 'profile', success: false, error: error.message }))
        );
      }
      
      if (input.preferences) {
        updatePromises.push(
          dataSources.userService.updatePreferences(user.id, input.preferences)
            .then(result => ({ type: 'preferences', success: true, data: result }))
            .catch(error => ({ type: 'preferences', success: false, error: error.message }))
        );
      }
      
      if (input.notifications) {
        updatePromises.push(
          dataSources.notificationService.updateSettings(user.id, input.notifications)
            .then(result => ({ type: 'notifications', success: true, data: result }))
            .catch(error => ({ type: 'notifications', success: false, error: error.message }))
        );
      }

      const results = await Promise.allSettled(updatePromises);
      
      const updateResults = results.map(result => 
        result.status === 'fulfilled' ? result.value : 
        { success: false, error: 'Update failed' }
      );

      return {
        success: updateResults.every(r => r.success),
        results: updateResults.filter(r => r.success),
        errors: updateResults.filter(r => !r.success)
      };
    }
  },

  User: {
    async analytics(
      user: User,
      _: any,
      { dataSources }: GraphQLContext
    ) {
      return dataSources.analyticsLoader.load({
        userId: user.id,
        timeframe: '30d'
      });
    }
  }
};

// DataLoader implementations for efficient batching
class DataLoaderService {
  public userLoader: DataLoader<string, User>;
  public activityLoader: DataLoader<ActivityQuery, Activity[]>;
  public analyticsLoader: DataLoader<AnalyticsQuery, Analytics>;
  public notificationLoader: DataLoader<NotificationQuery, Notification[]>;

  constructor(private services: ServiceClients) {
    this.userLoader = new DataLoader(async (userIds: readonly string[]) => {
      const users = await this.services.userService.getUsersBatch(Array.from(userIds));
      return userIds.map(id => users.find(user => user.id === id));
    });

    this.activityLoader = new DataLoader(async (queries: readonly ActivityQuery[]) => {
      const activities = await this.services.activityService.getActivitiesBatch(Array.from(queries));
      return queries.map(query => 
        activities.filter(activity => 
          activity.userId === query.userId && 
          (!query.platform || activity.platform === query.platform)
        )
      );
    });

    this.analyticsLoader = new DataLoader(async (queries: readonly AnalyticsQuery[]) => {
      const analytics = await this.services.analyticsService.getAnalyticsBatch(Array.from(queries));
      return queries.map(query => 
        analytics.find(analytic => 
          analytic.userId === query.userId && 
          analytic.timeframe === query.timeframe
        )
      );
    });

    this.notificationLoader = new DataLoader(async (queries: readonly NotificationQuery[]) => {
      const notifications = await this.services.notificationService.getNotificationsBatch(Array.from(queries));
      return queries.map(query => 
        notifications.filter(notification => 
          notification.userId === query.userId &&
          (!query.unreadOnly || !notification.read)
        )
      );
    });
  }
}

// GraphQL context with authentication and services
interface GraphQLContext {
  user: AuthUser;
  dataSources: DataLoaderService;
  services: ServiceClients;
}

// Apollo Server setup with federation
async function createBFFGraphQLServer() {
  const server = new ApolloServer({
    schema: buildSubgraphSchema({ typeDefs, resolvers }),
    plugins: [
      // Request logging plugin
      {
        requestDidStart() {
          return {
            didResolveOperation(requestContext) {
              console.log(`GraphQL Operation: ${requestContext.request.operationName}`);
            },
            willSendResponse(requestContext) {
              console.log(`Response time: ${Date.now() - requestContext.request.http?.startTime}ms`);
            }
          };
        }
      },
      // Caching plugin
      {
        requestDidStart() {
          return {
            willSendResponse(requestContext) {
              // Add cache headers for specific queries
              if (requestContext.request.operationName === 'Dashboard') {
                requestContext.response.http?.headers.set('Cache-Control', 'public, max-age=300');
              }
            }
          };
        }
      }
    ],
    introspection: process.env.NODE_ENV !== 'production',
    includeStacktraceInErrorResponses: process.env.NODE_ENV === 'development'
  });

  const { url } = await startStandaloneServer(server, {
    listen: { port: parseInt(process.env.GRAPHQL_PORT || '4000') },
    context: async ({ req }) => {
      // Authentication
      const token = req.headers.authorization?.replace('Bearer ', '');
      const user = await validateGraphQLToken(token);
      
      if (!user) {
        throw new Error('Authentication required');
      }

      // Initialize services and data loaders
      const services = new ServiceClients();
      const dataSources = new DataLoaderService(services);

      return {
        user,
        dataSources,
        services
      };
    }
  });

  console.log(`GraphQL BFF Server ready at: ${url}`);
  return server;
}

// Service clients for microservices
class ServiceClients {
  public userService: UserServiceClient;
  public activityService: ActivityServiceClient;
  public analyticsService: AnalyticsServiceClient;
  public notificationService: NotificationServiceClient;
  public searchService: SearchServiceClient;

  constructor() {
    this.userService = new UserServiceClient(process.env.USER_SERVICE_URL!);
    this.activityService = new ActivityServiceClient(process.env.ACTIVITY_SERVICE_URL!);
    this.analyticsService = new AnalyticsServiceClient(process.env.ANALYTICS_SERVICE_URL!);
    this.notificationService = new NotificationServiceClient(process.env.NOTIFICATION_SERVICE_URL!);
    this.searchService = new SearchServiceClient(process.env.SEARCH_SERVICE_URL!);
  }
}

// Utility functions
async function getWidgetsForUser(
  userId: string, 
  platform: Platform, 
  dataSources: DataLoaderService
): Promise<Widget[]> {
  const userPreferences = await dataSources.userLoader.load(userId);
  
  const baseWidgets: Widget[] = [
    { id: 'stats', type: 'STATISTICS', position: 1, enabled: true },
    { id: 'activity', type: 'ACTIVITY_FEED', position: 2, enabled: true },
    { id: 'notifications', type: 'NOTIFICATIONS', position: 3, enabled: true }
  ];

  if (platform === 'WEB') {
    baseWidgets.push(
      { id: 'analytics', type: 'ANALYTICS_CHART', position: 4, enabled: true },
      { id: 'quick-actions', type: 'QUICK_ACTIONS', position: 5, enabled: true }
    );
  }

  // Filter based on user preferences
  return baseWidgets.filter(widget => 
    userPreferences?.preferences?.dashboard?.widgets?.includes(widget.id) !== false
  );
}

async function generateSearchFacets(results: SearchResult[]): Promise<Facet[]> {
  const facets: Facet[] = [];
  
  // Type facet
  const typeCounts = results.reduce((acc, result) => {
    acc[result.type] = (acc[result.type] || 0) + 1;
    return acc;
  }, {} as Record<string, number>);
  
  facets.push({
    name: 'type',
    values: Object.entries(typeCounts).map(([value, count]) => ({ value, count }))
  });

  // Source facet
  const sourceCounts = results.reduce((acc, result) => {
    acc[result.source] = (acc[result.source] || 0) + 1;
    return acc;
  }, {} as Record<string, number>);
  
  facets.push({
    name: 'source',
    values: Object.entries(sourceCounts).map(([value, count]) => ({ value, count }))
  });

  return facets;
}

async function generateSearchSuggestions(query: string): Promise<string[]> {
  // Simple suggestion logic - in production, use ML-based suggestions
  const suggestions = [
    `${query} tutorial`,
    `${query} examples`,
    `${query} documentation`,
    `how to ${query}`,
    `${query} best practices`
  ];

  return suggestions.slice(0, 3);
}

function getQuickActionsForPlatform(platform: Platform): QuickAction[] {
  const baseActions = [
    { id: 'create', label: 'Create', icon: 'plus', priority: 1 },
    { id: 'search', label: 'Search', icon: 'search', priority: 2 },
    { id: 'notifications', label: 'Notifications', icon: 'bell', priority: 3 }
  ];

  if (platform === 'WEB') {
    baseActions.push(
      { id: 'analytics', label: 'Analytics', icon: 'chart', priority: 4 },
      { id: 'settings', label: 'Settings', icon: 'cog', priority: 5 }
    );
  }

  return baseActions.sort((a, b) => a.priority - b.priority);
}

async function validateGraphQLToken(token?: string): Promise<AuthUser | null> {
  if (!token) return null;
  
  try {
    const response = await fetch(`${process.env.AUTH_SERVICE_URL}/validate`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    
    return response.ok ? response.json() : null;
  } catch {
    return null;
  }
}

// Start the GraphQL BFF server
if (require.main === module) {
  createBFFGraphQLServer().catch(console.error);
}

export { createBFFGraphQLServer, DataLoaderService };
```

## Interview-Ready Summary

**BFF Architecture Benefits:**

1. **Frontend-Optimized APIs** - Tailored data shapes and aggregation for specific clients
2. **Performance Optimization** - Reduced network requests, efficient data loading, caching strategies
3. **Platform-Specific Logic** - Different data requirements for web, mobile, and tablet applications
4. **Microservice Aggregation** - Single entry point for complex multi-service operations
5. **Security Layer** - Centralized authentication, authorization, and input validation

**Advanced BFF Patterns:**
- **Data Aggregation** - Combining multiple microservice responses into optimized frontend payloads
- **Response Transformation** - Platform-specific data formatting and filtering
- **Caching Strategies** - Multi-layer caching with Redis and in-memory optimizations
- **Batch Operations** - Efficient bulk updates and parallel processing
- **GraphQL Integration** - Type-safe APIs with DataLoader for N+1 query prevention

**Production Considerations:**
- **Error Handling** - Graceful degradation with partial failures
- **Monitoring** - Request tracing, performance metrics, error tracking
- **Scalability** - Horizontal scaling, load balancing, circuit breakers
- **Security** - Rate limiting, input validation, service-to-service authentication

**Key Interview Topics:** BFF vs API Gateway differences, microservice communication patterns, data aggregation strategies, caching implementations, GraphQL vs REST in BFF context, performance optimization techniques, security considerations, monitoring and observability.