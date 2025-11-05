# Advanced API Patterns and Server State Management

Modern web applications require sophisticated API integration patterns and efficient server state management. This guide covers advanced API patterns, server state management strategies, and integration with modern React applications.

## Advanced API Integration Patterns

### 1. API Client Architecture and Patterns
```typescript
// Advanced API client with interceptors and middleware
interface ApiClientConfig {
  baseURL: string;
  timeout: number;
  retryAttempts: number;
  retryDelay: number;
  enableCaching: boolean;
  enableRequestDeduplication: boolean;
  enableMetrics: boolean;
}

interface RequestMiddleware {
  name: string;
  request?: (config: RequestConfig) => RequestConfig | Promise<RequestConfig>;
  response?: (response: ApiResponse) => ApiResponse | Promise<ApiResponse>;
  error?: (error: ApiError) => Promise<ApiError | ApiResponse>;
}

interface RequestConfig {
  url: string;
  method: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
  headers?: Record<string, string>;
  params?: Record<string, any>;
  data?: any;
  timeout?: number;
  retries?: number;
  cache?: boolean;
  dedupe?: boolean;
  signal?: AbortSignal;
}

interface ApiResponse<T = any> {
  data: T;
  status: number;
  statusText: string;
  headers: Record<string, string>;
  config: RequestConfig;
  cached?: boolean;
  fromCache?: boolean;
  timestamp: number;
}

interface ApiError {
  message: string;
  status?: number;
  code?: string;
  config: RequestConfig;
  response?: ApiResponse;
  isNetworkError: boolean;
  isTimeoutError: boolean;
  isRetryableError: boolean;
}

class AdvancedApiClient {
  private config: ApiClientConfig;
  private middleware: RequestMiddleware[] = [];
  private cache = new Map<string, { data: any; timestamp: number; ttl: number }>();
  private pendingRequests = new Map<string, Promise<ApiResponse>>();
  private metrics = {
    totalRequests: 0,
    successfulRequests: 0,
    failedRequests: 0,
    cacheHits: 0,
    retryCount: 0,
    averageResponseTime: 0,
  };

  constructor(config: ApiClientConfig) {
    this.config = config;
  }

  // Add middleware
  use(middleware: RequestMiddleware): void {
    this.middleware.push(middleware);
  }

  // Main request method
  async request<T = any>(config: RequestConfig): Promise<ApiResponse<T>> {
    const startTime = Date.now();
    this.metrics.totalRequests++;

    try {
      // Apply request middleware
      let processedConfig = { ...config };
      for (const middleware of this.middleware) {
        if (middleware.request) {
          processedConfig = await middleware.request(processedConfig);
        }
      }

      // Check cache
      if (this.config.enableCaching && processedConfig.cache !== false) {
        const cached = this.getCachedResponse<T>(processedConfig);
        if (cached) {
          this.metrics.cacheHits++;
          return cached;
        }
      }

      // Request deduplication
      if (this.config.enableRequestDeduplication && processedConfig.dedupe !== false) {
        const requestKey = this.getRequestKey(processedConfig);
        const pending = this.pendingRequests.get(requestKey);
        if (pending) {
          return pending as Promise<ApiResponse<T>>;
        }
      }

      // Execute request
      const responsePromise = this.executeRequest<T>(processedConfig);
      
      // Store pending request for deduplication
      if (this.config.enableRequestDeduplication && processedConfig.dedupe !== false) {
        const requestKey = this.getRequestKey(processedConfig);
        this.pendingRequests.set(requestKey, responsePromise);
        
        responsePromise.finally(() => {
          this.pendingRequests.delete(requestKey);
        });
      }

      let response = await responsePromise;

      // Apply response middleware
      for (const middleware of this.middleware) {
        if (middleware.response) {
          response = await middleware.response(response);
        }
      }

      // Cache response
      if (this.config.enableCaching && processedConfig.cache !== false) {
        this.cacheResponse(processedConfig, response);
      }

      // Update metrics
      this.metrics.successfulRequests++;
      this.updateResponseTimeMetrics(Date.now() - startTime);

      return response;
    } catch (error) {
      this.metrics.failedRequests++;
      
      // Apply error middleware
      let processedError = error as ApiError;
      for (const middleware of this.middleware) {
        if (middleware.error) {
          try {
            const result = await middleware.error(processedError);
            if ('data' in result) {
              // Middleware converted error to response
              return result as ApiResponse<T>;
            }
            processedError = result as ApiError;
          } catch (middlewareError) {
            processedError = middlewareError as ApiError;
          }
        }
      }

      throw processedError;
    }
  }

  // HTTP method shortcuts
  async get<T = any>(url: string, config?: Partial<RequestConfig>): Promise<ApiResponse<T>> {
    return this.request<T>({ ...config, url, method: 'GET' });
  }

  async post<T = any>(
    url: string,
    data?: any,
    config?: Partial<RequestConfig>
  ): Promise<ApiResponse<T>> {
    return this.request<T>({ ...config, url, method: 'POST', data });
  }

  async put<T = any>(
    url: string,
    data?: any,
    config?: Partial<RequestConfig>
  ): Promise<ApiResponse<T>> {
    return this.request<T>({ ...config, url, method: 'PUT', data });
  }

  async patch<T = any>(
    url: string,
    data?: any,
    config?: Partial<RequestConfig>
  ): Promise<ApiResponse<T>> {
    return this.request<T>({ ...config, url, method: 'PATCH', data });
  }

  async delete<T = any>(url: string, config?: Partial<RequestConfig>): Promise<ApiResponse<T>> {
    return this.request<T>({ ...config, url, method: 'DELETE' });
  }

  // Execute the actual HTTP request
  private async executeRequest<T>(config: RequestConfig): Promise<ApiResponse<T>> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), config.timeout || this.config.timeout);

    try {
      const response = await fetch(this.buildUrl(config), {
        method: config.method,
        headers: {
          'Content-Type': 'application/json',
          ...config.headers,
        },
        body: config.data ? JSON.stringify(config.data) : undefined,
        signal: config.signal || controller.signal,
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw new ApiError({
          message: `HTTP Error: ${response.status} ${response.statusText}`,
          status: response.status,
          config,
          isNetworkError: false,
          isTimeoutError: false,
          isRetryableError: response.status >= 500 || response.status === 429,
        });
      }

      const data = await response.json();

      return {
        data,
        status: response.status,
        statusText: response.statusText,
        headers: Object.fromEntries(response.headers.entries()),
        config,
        timestamp: Date.now(),
      };
    } catch (error) {
      clearTimeout(timeoutId);

      if (error.name === 'AbortError') {
        throw new ApiError({
          message: 'Request timeout',
          config,
          isNetworkError: false,
          isTimeoutError: true,
          isRetryableError: true,
        });
      }

      throw new ApiError({
        message: error.message || 'Network error',
        config,
        isNetworkError: true,
        isTimeoutError: false,
        isRetryableError: true,
      });
    }
  }

  // Cache management
  private getCachedResponse<T>(config: RequestConfig): ApiResponse<T> | null {
    const key = this.getRequestKey(config);
    const cached = this.cache.get(key);
    
    if (!cached) return null;
    
    const now = Date.now();
    if (now - cached.timestamp > cached.ttl) {
      this.cache.delete(key);
      return null;
    }

    return {
      ...cached.data,
      cached: true,
      fromCache: true,
    };
  }

  private cacheResponse(config: RequestConfig, response: ApiResponse): void {
    if (config.method !== 'GET') return; // Only cache GET requests
    
    const key = this.getRequestKey(config);
    const ttl = 5 * 60 * 1000; // 5 minutes default TTL
    
    this.cache.set(key, {
      data: response,
      timestamp: Date.now(),
      ttl,
    });
  }

  private getRequestKey(config: RequestConfig): string {
    const url = this.buildUrl(config);
    const params = config.params ? `?${new URLSearchParams(config.params)}` : '';
    return `${config.method}:${url}${params}`;
  }

  private buildUrl(config: RequestConfig): string {
    const baseURL = this.config.baseURL.replace(/\/$/, '');
    const url = config.url.startsWith('/') ? config.url : `/${config.url}`;
    return `${baseURL}${url}`;
  }

  private updateResponseTimeMetrics(responseTime: number): void {
    const { totalRequests, averageResponseTime } = this.metrics;
    this.metrics.averageResponseTime = 
      (averageResponseTime * (totalRequests - 1) + responseTime) / totalRequests;
  }

  // Public API for metrics
  getMetrics() {
    return { ...this.metrics };
  }

  clearCache(): void {
    this.cache.clear();
  }

  invalidateCache(pattern?: RegExp): void {
    if (!pattern) {
      this.clearCache();
      return;
    }

    for (const key of this.cache.keys()) {
      if (pattern.test(key)) {
        this.cache.delete(key);
      }
    }
  }
}

// Error class with detailed information
class ApiError extends Error {
  public status?: number;
  public code?: string;
  public config: RequestConfig;
  public response?: ApiResponse;
  public isNetworkError: boolean;
  public isTimeoutError: boolean;
  public isRetryableError: boolean;

  constructor(options: {
    message: string;
    status?: number;
    code?: string;
    config: RequestConfig;
    response?: ApiResponse;
    isNetworkError: boolean;
    isTimeoutError: boolean;
    isRetryableError: boolean;
  }) {
    super(options.message);
    this.name = 'ApiError';
    this.status = options.status;
    this.code = options.code;
    this.config = options.config;
    this.response = options.response;
    this.isNetworkError = options.isNetworkError;
    this.isTimeoutError = options.isTimeoutError;
    this.isRetryableError = options.isRetryableError;
  }
}

// Middleware implementations
export const authMiddleware: RequestMiddleware = {
  name: 'auth',
  request: async (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers = {
        ...config.headers,
        Authorization: `Bearer ${token}`,
      };
    }
    return config;
  },
  error: async (error) => {
    if (error.status === 401) {
      // Token expired, try to refresh
      try {
        const refreshResponse = await fetch('/api/auth/refresh', {
          method: 'POST',
          credentials: 'include',
        });
        
        if (refreshResponse.ok) {
          const { accessToken } = await refreshResponse.json();
          localStorage.setItem('authToken', accessToken);
          
          // Retry original request with new token
          error.config.headers = {
            ...error.config.headers,
            Authorization: `Bearer ${accessToken}`,
          };
          
          return apiClient.request(error.config);
        }
      } catch (refreshError) {
        // Refresh failed, redirect to login
        localStorage.removeItem('authToken');
        window.location.href = '/login';
      }
    }
    throw error;
  },
};

export const retryMiddleware: RequestMiddleware = {
  name: 'retry',
  error: async (error) => {
    const maxRetries = error.config.retries || 3;
    const retryCount = error.config.retries || 0;
    
    if (error.isRetryableError && retryCount < maxRetries) {
      const delay = Math.min(1000 * Math.pow(2, retryCount), 10000); // Exponential backoff
      await new Promise(resolve => setTimeout(resolve, delay));
      
      const retryConfig = {
        ...error.config,
        retries: retryCount + 1,
      };
      
      return apiClient.request(retryConfig);
    }
    
    throw error;
  },
};

export const loggingMiddleware: RequestMiddleware = {
  name: 'logging',
  request: async (config) => {
    console.log(`[API] ${config.method} ${config.url}`, config);
    return config;
  },
  response: async (response) => {
    console.log(`[API] ${response.status} ${response.config.method} ${response.config.url}`, response);
    return response;
  },
  error: async (error) => {
    console.error(`[API] Error ${error.config.method} ${error.config.url}`, error);
    throw error;
  },
};

// Create configured API client
export const apiClient = new AdvancedApiClient({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001',
  timeout: 10000,
  retryAttempts: 3,
  retryDelay: 1000,
  enableCaching: true,
  enableRequestDeduplication: true,
  enableMetrics: true,
});

// Add middleware
apiClient.use(authMiddleware);
apiClient.use(retryMiddleware);
if (process.env.NODE_ENV === 'development') {
  apiClient.use(loggingMiddleware);
}
```

### 2. Server State Management with React Query/TanStack Query
```typescript
// Advanced React Query configuration and patterns
import {
  QueryClient,
  QueryClientProvider,
  useQuery,
  useMutation,
  useInfiniteQuery,
  useQueryClient,
  QueryKey,
  UseQueryOptions,
  UseMutationOptions,
  UseInfiniteQueryOptions,
} from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import React, { ReactNode, useState, useCallback, useMemo } from 'react';

// Create optimized Query Client
export const createQueryClient = () => {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 5 * 60 * 1000, // 5 minutes
        cacheTime: 10 * 60 * 1000, // 10 minutes
        retry: (failureCount, error: any) => {
          // Don't retry on 4xx errors except 408, 429
          if (error?.status >= 400 && error?.status < 500) {
            return error?.status === 408 || error?.status === 429;
          }
          return failureCount < 3;
        },
        retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
        refetchOnWindowFocus: false,
        refetchOnReconnect: 'always',
      },
      mutations: {
        retry: false,
        onError: (error: any) => {
          console.error('Mutation error:', error);
          // Global error handling
          if (error?.status === 401) {
            // Handle unauthorized
          }
        },
      },
    },
  });
};

// Query client instance
export const queryClient = createQueryClient();

// Provider with error boundary
interface QueryProviderProps {
  children: ReactNode;
}

export const QueryProvider: React.FC<QueryProviderProps> = ({ children }) => {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} />
      )}
    </QueryClientProvider>
  );
};

// Type-safe query keys factory
export const queryKeys = {
  all: ['api'] as const,
  
  // Users
  users: () => [...queryKeys.all, 'users'] as const,
  user: (id: string) => [...queryKeys.users(), 'detail', id] as const,
  userPosts: (id: string, filters?: any) => 
    [...queryKeys.user(id), 'posts', filters] as const,
  
  // Posts
  posts: () => [...queryKeys.all, 'posts'] as const,
  post: (id: string) => [...queryKeys.posts(), 'detail', id] as const,
  postComments: (id: string) => [...queryKeys.post(id), 'comments'] as const,
  
  // Search
  search: (query: string, filters?: any) => 
    [...queryKeys.all, 'search', query, filters] as const,
} as const;

// Advanced API service with React Query integration
interface User {
  id: string;
  name: string;
  email: string;
  avatar?: string;
  createdAt: string;
}

interface Post {
  id: string;
  title: string;
  content: string;
  authorId: string;
  author?: User;
  createdAt: string;
  updatedAt: string;
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    hasNextPage: boolean;
    hasPreviousPage: boolean;
  };
}

class ApiService {
  private client: AdvancedApiClient;

  constructor(client: AdvancedApiClient) {
    this.client = client;
  }

  // Users API
  async getUsers(params?: {
    page?: number;
    limit?: number;
    search?: string;
  }): Promise<PaginatedResponse<User>> {
    const response = await this.client.get('/users', { params });
    return response.data;
  }

  async getUser(id: string): Promise<User> {
    const response = await this.client.get(`/users/${id}`);
    return response.data;
  }

  async createUser(userData: Partial<User>): Promise<User> {
    const response = await this.client.post('/users', userData);
    return response.data;
  }

  async updateUser(id: string, userData: Partial<User>): Promise<User> {
    const response = await this.client.patch(`/users/${id}`, userData);
    return response.data;
  }

  async deleteUser(id: string): Promise<void> {
    await this.client.delete(`/users/${id}`);
  }

  // Posts API
  async getPosts(params?: {
    page?: number;
    limit?: number;
    authorId?: string;
    search?: string;
  }): Promise<PaginatedResponse<Post>> {
    const response = await this.client.get('/posts', { params });
    return response.data;
  }

  async getPost(id: string): Promise<Post> {
    const response = await this.client.get(`/posts/${id}`);
    return response.data;
  }

  async createPost(postData: Partial<Post>): Promise<Post> {
    const response = await this.client.post('/posts', postData);
    return response.data;
  }

  async updatePost(id: string, postData: Partial<Post>): Promise<Post> {
    const response = await this.client.patch(`/posts/${id}`, postData);
    return response.data;
  }

  async deletePost(id: string): Promise<void> {
    await this.client.delete(`/posts/${id}`);
  }

  // Infinite scroll posts
  async getInfinitePosts(params: {
    page: number;
    limit: number;
    authorId?: string;
    search?: string;
  }): Promise<PaginatedResponse<Post>> {
    const response = await this.client.get('/posts', { params });
    return response.data;
  }
}

// Create API service instance
export const apiService = new ApiService(apiClient);

// Custom hooks for data fetching
interface UseUsersOptions {
  page?: number;
  limit?: number;
  search?: string;
  enabled?: boolean;
}

export function useUsers(options: UseUsersOptions = {}) {
  const { page = 1, limit = 10, search, enabled = true } = options;

  return useQuery({
    queryKey: queryKeys.users(),
    queryFn: () => apiService.getUsers({ page, limit, search }),
    enabled,
    select: (data) => ({
      users: data.data,
      pagination: data.pagination,
    }),
    placeholderData: (previousData) => previousData, // Keep previous data while loading
  });
}

export function useUser(id: string, options?: { enabled?: boolean }) {
  return useQuery({
    queryKey: queryKeys.user(id),
    queryFn: () => apiService.getUser(id),
    enabled: options?.enabled ?? !!id,
    retry: (failureCount, error: any) => {
      // Don't retry on 404 errors
      if (error?.status === 404) return false;
      return failureCount < 3;
    },
  });
}

// Infinite query for posts
interface UseInfinitePostsOptions {
  limit?: number;
  authorId?: string;
  search?: string;
  enabled?: boolean;
}

export function useInfinitePosts(options: UseInfinitePostsOptions = {}) {
  const { limit = 10, authorId, search, enabled = true } = options;

  return useInfiniteQuery({
    queryKey: queryKeys.posts(),
    queryFn: ({ pageParam = 1 }) =>
      apiService.getInfinitePosts({
        page: pageParam,
        limit,
        authorId,
        search,
      }),
    getNextPageParam: (lastPage) =>
      lastPage.pagination.hasNextPage ? lastPage.pagination.page + 1 : undefined,
    getPreviousPageParam: (firstPage) =>
      firstPage.pagination.hasPreviousPage ? firstPage.pagination.page - 1 : undefined,
    enabled,
    select: (data) => ({
      pages: data.pages,
      pageParams: data.pageParams,
      posts: data.pages.flatMap(page => page.data),
      totalCount: data.pages[0]?.pagination.total || 0,
    }),
  });
}

// Optimistic mutations
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: apiService.createUser,
    onMutate: async (newUser) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: queryKeys.users() });

      // Snapshot previous value
      const previousUsers = queryClient.getQueryData(queryKeys.users());

      // Optimistically update
      if (previousUsers) {
        queryClient.setQueryData(queryKeys.users(), (old: any) => ({
          ...old,
          data: [
            {
              id: `temp-${Date.now()}`,
              ...newUser,
              createdAt: new Date().toISOString(),
            },
            ...old.data,
          ],
          pagination: {
            ...old.pagination,
            total: old.pagination.total + 1,
          },
        }));
      }

      return { previousUsers };
    },
    onError: (error, newUser, context) => {
      // Rollback on error
      if (context?.previousUsers) {
        queryClient.setQueryData(queryKeys.users(), context.previousUsers);
      }
    },
    onSettled: () => {
      // Refetch users after mutation
      queryClient.invalidateQueries({ queryKey: queryKeys.users() });
    },
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<User> }) =>
      apiService.updateUser(id, data),
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: queryKeys.user(id) });

      const previousUser = queryClient.getQueryData(queryKeys.user(id));

      // Optimistically update user
      queryClient.setQueryData(queryKeys.user(id), (old: User) => ({
        ...old,
        ...data,
        updatedAt: new Date().toISOString(),
      }));

      return { previousUser };
    },
    onError: (error, { id }, context) => {
      if (context?.previousUser) {
        queryClient.setQueryData(queryKeys.user(id), context.previousUser);
      }
    },
    onSettled: (data, error, { id }) => {
      queryClient.invalidateQueries({ queryKey: queryKeys.user(id) });
      queryClient.invalidateQueries({ queryKey: queryKeys.users() });
    },
  });
}

// Prefetching utilities
export function usePrefetchUser() {
  const queryClient = useQueryClient();

  return useCallback(
    (id: string) => {
      queryClient.prefetchQuery({
        queryKey: queryKeys.user(id),
        queryFn: () => apiService.getUser(id),
        staleTime: 60 * 1000, // 1 minute
      });
    },
    [queryClient]
  );
}

// Background refetching
export function useBackgroundRefetch() {
  const queryClient = useQueryClient();

  const refetchAll = useCallback(() => {
    queryClient.refetchQueries({
      type: 'active',
      stale: true,
    });
  }, [queryClient]);

  const refetchUsers = useCallback(() => {
    queryClient.refetchQueries({
      queryKey: queryKeys.users(),
    });
  }, [queryClient]);

  const refetchPosts = useCallback(() => {
    queryClient.refetchQueries({
      queryKey: queryKeys.posts(),
    });
  }, [queryClient]);

  return {
    refetchAll,
    refetchUsers,
    refetchPosts,
  };
}

// Cache management utilities
export function useCacheManagement() {
  const queryClient = useQueryClient();

  const invalidateAll = useCallback(() => {
    queryClient.invalidateQueries();
  }, [queryClient]);

  const clearCache = useCallback(() => {
    queryClient.clear();
  }, [queryClient]);

  const invalidateUsers = useCallback(() => {
    queryClient.invalidateQueries({ queryKey: queryKeys.users() });
  }, [queryClient]);

  const invalidateUser = useCallback((id: string) => {
    queryClient.invalidateQueries({ queryKey: queryKeys.user(id) });
  }, [queryClient]);

  const removeUser = useCallback((id: string) => {
    queryClient.removeQueries({ queryKey: queryKeys.user(id) });
  }, [queryClient]);

  const setUserData = useCallback((id: string, data: User) => {
    queryClient.setQueryData(queryKeys.user(id), data);
  }, [queryClient]);

  return {
    invalidateAll,
    clearCache,
    invalidateUsers,
    invalidateUser,
    removeUser,
    setUserData,
  };
}
```

### 3. Advanced State Synchronization Patterns
```typescript
// Real-time state synchronization with WebSockets
interface WebSocketMessage<T = any> {
  type: string;
  id?: string;
  data: T;
  timestamp: number;
  userId?: string;
}

interface WebSocketConfig {
  url: string;
  reconnectAttempts: number;
  reconnectDelay: number;
  heartbeatInterval: number;
  enableCompression: boolean;
}

class RealtimeStateManager {
  private ws: WebSocket | null = null;
  private config: WebSocketConfig;
  private listeners = new Map<string, Set<Function>>();
  private reconnectCount = 0;
  private heartbeatTimer: NodeJS.Timeout | null = null;
  private queryClient: QueryClient;

  constructor(config: WebSocketConfig, queryClient: QueryClient) {
    this.config = config;
    this.queryClient = queryClient;
  }

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      try {
        this.ws = new WebSocket(this.config.url);

        this.ws.onopen = () => {
          console.log('WebSocket connected');
          this.reconnectCount = 0;
          this.startHeartbeat();
          resolve();
        };

        this.ws.onmessage = (event) => {
          try {
            const message: WebSocketMessage = JSON.parse(event.data);
            this.handleMessage(message);
          } catch (error) {
            console.error('Failed to parse WebSocket message:', error);
          }
        };

        this.ws.onclose = (event) => {
          console.log('WebSocket disconnected:', event.code, event.reason);
          this.stopHeartbeat();
          
          if (!event.wasClean && this.reconnectCount < this.config.reconnectAttempts) {
            this.scheduleReconnect();
          }
        };

        this.ws.onerror = (error) => {
          console.error('WebSocket error:', error);
          reject(error);
        };
      } catch (error) {
        reject(error);
      }
    });
  }

  disconnect(): void {
    if (this.ws) {
      this.ws.close(1000, 'Client disconnect');
      this.ws = null;
    }
    this.stopHeartbeat();
  }

  // Subscribe to specific message types
  subscribe(messageType: string, callback: Function): () => void {
    if (!this.listeners.has(messageType)) {
      this.listeners.set(messageType, new Set());
    }
    
    this.listeners.get(messageType)!.add(callback);

    // Return unsubscribe function
    return () => {
      const typeListeners = this.listeners.get(messageType);
      if (typeListeners) {
        typeListeners.delete(callback);
        if (typeListeners.size === 0) {
          this.listeners.delete(messageType);
        }
      }
    };
  }

  // Send message
  send(message: Omit<WebSocketMessage, 'timestamp'>): void {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      const fullMessage: WebSocketMessage = {
        ...message,
        timestamp: Date.now(),
      };
      this.ws.send(JSON.stringify(fullMessage));
    } else {
      console.warn('WebSocket not connected, message not sent:', message);
    }
  }

  private handleMessage(message: WebSocketMessage): void {
    // Handle specific message types for cache updates
    switch (message.type) {
      case 'USER_CREATED':
        this.handleUserCreated(message.data);
        break;
      case 'USER_UPDATED':
        this.handleUserUpdated(message.data);
        break;
      case 'USER_DELETED':
        this.handleUserDeleted(message.data);
        break;
      case 'POST_CREATED':
        this.handlePostCreated(message.data);
        break;
      case 'POST_UPDATED':
        this.handlePostUpdated(message.data);
        break;
      case 'POST_DELETED':
        this.handlePostDeleted(message.data);
        break;
    }

    // Notify subscribers
    const typeListeners = this.listeners.get(message.type);
    if (typeListeners) {
      typeListeners.forEach(callback => {
        try {
          callback(message);
        } catch (error) {
          console.error('Error in WebSocket message callback:', error);
        }
      });
    }
  }

  private handleUserCreated(user: User): void {
    // Update users list in cache
    this.queryClient.setQueryData(queryKeys.users(), (old: any) => {
      if (!old) return old;
      
      return {
        ...old,
        data: [user, ...old.data],
        pagination: {
          ...old.pagination,
          total: old.pagination.total + 1,
        },
      };
    });

    // Set individual user cache
    this.queryClient.setQueryData(queryKeys.user(user.id), user);
  }

  private handleUserUpdated(user: User): void {
    // Update individual user cache
    this.queryClient.setQueryData(queryKeys.user(user.id), user);

    // Update user in users list
    this.queryClient.setQueryData(queryKeys.users(), (old: any) => {
      if (!old) return old;

      return {
        ...old,
        data: old.data.map((u: User) => u.id === user.id ? user : u),
      };
    });
  }

  private handleUserDeleted(data: { id: string }): void {
    // Remove from users list
    this.queryClient.setQueryData(queryKeys.users(), (old: any) => {
      if (!old) return old;

      return {
        ...old,
        data: old.data.filter((u: User) => u.id !== data.id),
        pagination: {
          ...old.pagination,
          total: old.pagination.total - 1,
        },
      };
    });

    // Remove individual user cache
    this.queryClient.removeQueries({ queryKey: queryKeys.user(data.id) });
  }

  private handlePostCreated(post: Post): void {
    // Update posts list
    this.queryClient.setQueryData(queryKeys.posts(), (old: any) => {
      if (!old) return old;

      // For infinite queries
      if (old.pages) {
        const newPages = [...old.pages];
        if (newPages[0]) {
          newPages[0] = {
            ...newPages[0],
            data: [post, ...newPages[0].data],
            pagination: {
              ...newPages[0].pagination,
              total: newPages[0].pagination.total + 1,
            },
          };
        }
        return {
          ...old,
          pages: newPages,
        };
      }

      // For regular paginated queries
      return {
        ...old,
        data: [post, ...old.data],
        pagination: {
          ...old.pagination,
          total: old.pagination.total + 1,
        },
      };
    });

    // Set individual post cache
    this.queryClient.setQueryData(queryKeys.post(post.id), post);
  }

  private handlePostUpdated(post: Post): void {
    // Update individual post cache
    this.queryClient.setQueryData(queryKeys.post(post.id), post);

    // Update post in lists
    this.queryClient.setQueryData(queryKeys.posts(), (old: any) => {
      if (!old) return old;

      if (old.pages) {
        // Handle infinite query structure
        const newPages = old.pages.map((page: any) => ({
          ...page,
          data: page.data.map((p: Post) => p.id === post.id ? post : p),
        }));
        return { ...old, pages: newPages };
      }

      // Handle regular pagination
      return {
        ...old,
        data: old.data.map((p: Post) => p.id === post.id ? post : p),
      };
    });
  }

  private handlePostDeleted(data: { id: string }): void {
    // Remove from posts list
    this.queryClient.setQueryData(queryKeys.posts(), (old: any) => {
      if (!old) return old;

      if (old.pages) {
        const newPages = old.pages.map((page: any) => ({
          ...page,
          data: page.data.filter((p: Post) => p.id !== data.id),
          pagination: {
            ...page.pagination,
            total: page.pagination.total - 1,
          },
        }));
        return { ...old, pages: newPages };
      }

      return {
        ...old,
        data: old.data.filter((p: Post) => p.id !== data.id),
        pagination: {
          ...old.pagination,
          total: old.pagination.total - 1,
        },
      };
    });

    // Remove individual post cache
    this.queryClient.removeQueries({ queryKey: queryKeys.post(data.id) });
  }

  private startHeartbeat(): void {
    this.heartbeatTimer = setInterval(() => {
      this.send({ type: 'HEARTBEAT', data: {} });
    }, this.config.heartbeatInterval);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  private scheduleReconnect(): void {
    const delay = this.config.reconnectDelay * Math.pow(2, this.reconnectCount);
    
    setTimeout(() => {
      this.reconnectCount++;
      console.log(`Attempting WebSocket reconnection (${this.reconnectCount}/${this.config.reconnectAttempts})`);
      this.connect().catch(console.error);
    }, delay);
  }
}

// React hook for WebSocket integration
export function useRealtimeSync() {
  const queryClient = useQueryClient();
  const [connectionStatus, setConnectionStatus] = useState<'connecting' | 'connected' | 'disconnected'>('disconnected');
  const managerRef = useRef<RealtimeStateManager>();

  useEffect(() => {
    const config: WebSocketConfig = {
      url: process.env.NEXT_PUBLIC_WS_URL || 'ws://localhost:3001',
      reconnectAttempts: 5,
      reconnectDelay: 1000,
      heartbeatInterval: 30000,
      enableCompression: true,
    };

    managerRef.current = new RealtimeStateManager(config, queryClient);
    
    setConnectionStatus('connecting');
    managerRef.current.connect()
      .then(() => setConnectionStatus('connected'))
      .catch(() => setConnectionStatus('disconnected'));

    return () => {
      managerRef.current?.disconnect();
    };
  }, [queryClient]);

  const subscribe = useCallback((messageType: string, callback: Function) => {
    return managerRef.current?.subscribe(messageType, callback) || (() => {});
  }, []);

  const send = useCallback((message: Omit<WebSocketMessage, 'timestamp'>) => {
    managerRef.current?.send(message);
  }, []);

  return {
    connectionStatus,
    subscribe,
    send,
  };
}

// Optimistic UI patterns
export function useOptimisticMutation<TData, TVariables>(
  mutationFn: (variables: TVariables) => Promise<TData>,
  options: {
    onMutate?: (variables: TVariables) => Promise<any>;
    onError?: (error: any, variables: TVariables, context: any) => void;
    onSettled?: (data: TData | undefined, error: any, variables: TVariables, context: any) => void;
  }
) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn,
    onMutate: options.onMutate,
    onError: (error, variables, context) => {
      // Rollback optimistic updates
      options.onError?.(error, variables, context);
    },
    onSettled: options.onSettled,
  });
}

// Batch operations
export function useBatchOperations() {
  const queryClient = useQueryClient();

  const batchInvalidate = useCallback((queryKeys: QueryKey[]) => {
    queryKeys.forEach(key => {
      queryClient.invalidateQueries({ queryKey: key });
    });
  }, [queryClient]);

  const batchPrefetch = useCallback(async (queries: Array<{ queryKey: QueryKey; queryFn: () => Promise<any> }>) => {
    await Promise.all(
      queries.map(({ queryKey, queryFn }) =>
        queryClient.prefetchQuery({ queryKey, queryFn })
      )
    );
  }, [queryClient]);

  const batchUpdate = useCallback((updates: Array<{ queryKey: QueryKey; updater: (old: any) => any }>) => {
    updates.forEach(({ queryKey, updater }) => {
      queryClient.setQueryData(queryKey, updater);
    });
  }, [queryClient]);

  return {
    batchInvalidate,
    batchPrefetch,
    batchUpdate,
  };
}
```

## Interview-Ready Summary

**Advanced API Patterns and Server State require:**

1. **API Client Architecture** - Advanced HTTP client with middleware, retry logic, caching, request deduplication, comprehensive error handling
2. **Server State Management** - React Query/TanStack Query with optimistic updates, infinite queries, cache management, prefetching strategies
3. **Real-time Synchronization** - WebSocket integration, automatic cache updates, conflict resolution, connection management
4. **Performance Optimization** - Request batching, intelligent caching, background refetching, state synchronization

**Key implementation patterns:**
- **Middleware System** - Auth, retry, logging, caching middleware for API requests
- **Optimistic Updates** - Immediate UI updates with rollback on errors
- **Cache Strategy** - Multi-level caching with TTL, invalidation patterns, background updates
- **Real-time Sync** - WebSocket-driven cache updates, conflict resolution, offline support

**Common challenges:** Stale data, cache invalidation, network failures, race conditions, memory leaks, state synchronization conflicts.

**Best practices:** Implement comprehensive error handling, use TypeScript for type safety, optimize cache policies, handle offline scenarios, implement proper retry strategies, monitor API performance, use optimistic updates judiciously, manage WebSocket connections efficiently.