# Server State Management and Data Synchronization

Server state management is fundamentally different from client state, requiring specialized patterns for caching, synchronization, optimistic updates, and real-time data. This guide covers advanced server state management using React Query (TanStack Query), SWR, and custom synchronization strategies.

## React Query (TanStack Query) Advanced Patterns

### 1. Advanced Query Configuration and Optimization
```typescript
// Advanced React Query setup with comprehensive configuration
import {
  useQuery,
  useMutation,
  useQueryClient,
  useInfiniteQuery,
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Enhanced query client configuration
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Stale time - how long until data is considered stale
      staleTime: 5 * 60 * 1000, // 5 minutes
      
      // Cache time - how long inactive data stays in cache
      gcTime: 10 * 60 * 1000, // 10 minutes (was cacheTime)
      
      // Retry configuration
      retry: (failureCount, error: any) => {
        // Don't retry on 4xx errors
        if (error?.status >= 400 && error?.status < 500) {
          return false;
        }
        // Retry up to 3 times for other errors
        return failureCount < 3;
      },
      
      // Retry delay with exponential backoff
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      
      // Refetch configuration
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
      refetchOnMount: true,
      
      // Network mode
      networkMode: 'online',
    },
    mutations: {
      retry: 1,
      networkMode: 'online',
    },
  },
});

// Enhanced API client with interceptors
class ApiClient {
  private baseURL: string;
  private defaultHeaders: Record<string, string>;

  constructor(baseURL: string) {
    this.baseURL = baseURL;
    this.defaultHeaders = {
      'Content-Type': 'application/json',
    };
  }

  private async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseURL}${endpoint}`;
    
    // Add authentication header
    const token = localStorage.getItem('auth_token');
    const headers = {
      ...this.defaultHeaders,
      ...(token && { Authorization: `Bearer ${token}` }),
      ...options.headers,
    };

    const config: RequestInit = {
      ...options,
      headers,
    };

    const response = await fetch(url, config);

    // Handle authentication errors
    if (response.status === 401) {
      // Try to refresh token
      const refreshed = await this.refreshToken();
      if (refreshed) {
        // Retry original request
        const newToken = localStorage.getItem('auth_token');
        return this.request(endpoint, {
          ...options,
          headers: {
            ...headers,
            Authorization: `Bearer ${newToken}`,
          },
        });
      } else {
        // Redirect to login
        window.location.href = '/login';
        throw new Error('Authentication failed');
      }
    }

    if (!response.ok) {
      const errorData = await response.json().catch(() => null);
      throw new ApiError(
        errorData?.message || `HTTP ${response.status}`,
        response.status,
        errorData
      );
    }

    return response.json();
  }

  private async refreshToken(): Promise<boolean> {
    try {
      const refreshToken = localStorage.getItem('refresh_token');
      if (!refreshToken) return false;

      const response = await fetch(`${this.baseURL}/auth/refresh`, {
        method: 'POST',
        headers: this.defaultHeaders,
        body: JSON.stringify({ refreshToken }),
      });

      if (response.ok) {
        const { token, refreshToken: newRefreshToken } = await response.json();
        localStorage.setItem('auth_token', token);
        localStorage.setItem('refresh_token', newRefreshToken);
        return true;
      }
    } catch (error) {
      console.error('Token refresh failed:', error);
    }
    
    return false;
  }

  get<T>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint);
  }

  post<T>(endpoint: string, data: any): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  put<T>(endpoint: string, data: any): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data),
    });
  }

  patch<T>(endpoint: string, data: any): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'PATCH',
      body: JSON.stringify(data),
    });
  }

  delete<T>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'DELETE',
    });
  }
}

class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public data?: any
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export const api = new ApiClient('/api');

// Query key factories for consistent cache management
export const queryKeys = {
  all: ['queries'] as const,
  users: () => [...queryKeys.all, 'users'] as const,
  user: (id: string) => [...queryKeys.users(), id] as const,
  userPosts: (userId: string) => [...queryKeys.user(userId), 'posts'] as const,
  posts: () => [...queryKeys.all, 'posts'] as const,
  post: (id: string) => [...queryKeys.posts(), id] as const,
  postComments: (postId: string) => [...queryKeys.post(postId), 'comments'] as const,
  search: (query: string, filters: Record<string, any>) => 
    [...queryKeys.all, 'search', query, filters] as const,
};

// Advanced query hooks with comprehensive error handling
export function useUsers(params: {
  page?: number;
  limit?: number;
  search?: string;
  role?: string;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}) {
  return useQuery({
    queryKey: [...queryKeys.users(), params],
    queryFn: async () => {
      const searchParams = new URLSearchParams();
      
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined) {
          searchParams.set(key, String(value));
        }
      });

      return api.get<{
        users: User[];
        total: number;
        hasMore: boolean;
        page: number;
      }>(`/users?${searchParams}`);
    },
    staleTime: 2 * 60 * 1000, // 2 minutes
    gcTime: 5 * 60 * 1000, // 5 minutes
    placeholderData: (previousData) => previousData,
    select: (data) => ({
      ...data,
      users: data.users.map(user => ({
        ...user,
        isActive: new Date(user.lastActive) > new Date(Date.now() - 24 * 60 * 60 * 1000),
      })),
    }),
  });
}

export function useUser(id: string, options?: { enabled?: boolean }) {
  return useQuery({
    queryKey: queryKeys.user(id),
    queryFn: () => api.get<User>(`/users/${id}`),
    enabled: !!id && (options?.enabled ?? true),
    staleTime: 5 * 60 * 1000,
    gcTime: 10 * 60 * 1000,
  });
}

// Infinite query for pagination
export function useInfiniteUsers(filters: {
  search?: string;
  role?: string;
  limit?: number;
}) {
  return useInfiniteQuery({
    queryKey: [...queryKeys.users(), 'infinite', filters],
    queryFn: async ({ pageParam = 1 }) => {
      const searchParams = new URLSearchParams({
        page: String(pageParam),
        limit: String(filters.limit || 20),
        ...(filters.search && { search: filters.search }),
        ...(filters.role && { role: filters.role }),
      });

      return api.get<{
        users: User[];
        nextPage?: number;
        hasMore: boolean;
      }>(`/users?${searchParams}`);
    },
    getNextPageParam: (lastPage) => lastPage.nextPage,
    getPreviousPageParam: (firstPage, pages) => {
      return pages.length > 1 ? pages.length - 1 : undefined;
    },
    initialPageParam: 1,
    staleTime: 2 * 60 * 1000,
    gcTime: 5 * 60 * 1000,
  });
}

// Advanced mutation hooks with optimistic updates
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (userData: CreateUserData) => api.post<User>('/users', userData),
    
    onMutate: async (newUser) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: queryKeys.users() });

      // Snapshot previous value
      const previousUsers = queryClient.getQueriesData({ queryKey: queryKeys.users() });

      // Optimistically update cache
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old) return old;
          
          const optimisticUser: User = {
            ...newUser,
            id: `temp_${Date.now()}`,
            lastActive: new Date().toISOString(),
          };

          return {
            ...old,
            users: [optimisticUser, ...old.users],
            total: old.total + 1,
          };
        }
      );

      return { previousUsers };
    },

    onError: (error, newUser, context) => {
      // Rollback optimistic updates
      if (context?.previousUsers) {
        context.previousUsers.forEach(([queryKey, data]) => {
          queryClient.setQueryData(queryKey, data);
        });
      }
    },

    onSuccess: (newUser, variables) => {
      // Update specific queries with real data
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old) return old;

          return {
            ...old,
            users: old.users.map((user: User) =>
              user.id.startsWith('temp_') ? newUser : user
            ),
          };
        }
      );

      // Set individual user cache
      queryClient.setQueryData(queryKeys.user(newUser.id), newUser);
    },

    onSettled: () => {
      // Always invalidate queries to ensure consistency
      queryClient.invalidateQueries({ queryKey: queryKeys.users() });
    },
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, updates }: { id: string; updates: Partial<User> }) =>
      api.patch<User>(`/users/${id}`, updates),

    onMutate: async ({ id, updates }) => {
      // Cancel queries
      await queryClient.cancelQueries({ queryKey: queryKeys.user(id) });
      await queryClient.cancelQueries({ queryKey: queryKeys.users() });

      // Snapshot previous values
      const previousUser = queryClient.getQueryData(queryKeys.user(id));
      const previousUsersList = queryClient.getQueriesData({ queryKey: queryKeys.users() });

      // Optimistic update for individual user
      queryClient.setQueryData(queryKeys.user(id), (old: User | undefined) =>
        old ? { ...old, ...updates } : old
      );

      // Optimistic update for user lists
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old?.users) return old;

          return {
            ...old,
            users: old.users.map((user: User) =>
              user.id === id ? { ...user, ...updates } : user
            ),
          };
        }
      );

      return { previousUser, previousUsersList, id };
    },

    onError: (error, { id }, context) => {
      // Rollback changes
      if (context?.previousUser) {
        queryClient.setQueryData(queryKeys.user(id), context.previousUser);
      }
      
      if (context?.previousUsersList) {
        context.previousUsersList.forEach(([queryKey, data]) => {
          queryClient.setQueryData(queryKey, data);
        });
      }
    },

    onSuccess: (updatedUser, { id }) => {
      // Update caches with server response
      queryClient.setQueryData(queryKeys.user(id), updatedUser);
      
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old?.users) return old;

          return {
            ...old,
            users: old.users.map((user: User) =>
              user.id === id ? updatedUser : user
            ),
          };
        }
      );
    },

    onSettled: (data, error, { id }) => {
      // Invalidate related queries
      queryClient.invalidateQueries({ queryKey: queryKeys.user(id) });
      queryClient.invalidateQueries({ queryKey: queryKeys.users() });
    },
  });
}

export function useDeleteUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => api.delete(`/users/${id}`),

    onMutate: async (id) => {
      // Cancel queries
      await queryClient.cancelQueries({ queryKey: queryKeys.users() });

      // Snapshot previous data
      const previousUsersList = queryClient.getQueriesData({ queryKey: queryKeys.users() });
      const previousUser = queryClient.getQueryData(queryKeys.user(id));

      // Optimistic update - remove user from lists
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old?.users) return old;

          return {
            ...old,
            users: old.users.filter((user: User) => user.id !== id),
            total: old.total - 1,
          };
        }
      );

      // Remove individual user cache
      queryClient.removeQueries({ queryKey: queryKeys.user(id) });

      return { previousUsersList, previousUser, id };
    },

    onError: (error, id, context) => {
      // Rollback changes
      if (context?.previousUsersList) {
        context.previousUsersList.forEach(([queryKey, data]) => {
          queryClient.setQueryData(queryKey, data);
        });
      }

      if (context?.previousUser) {
        queryClient.setQueryData(queryKeys.user(id), context.previousUser);
      }
    },

    onSettled: () => {
      // Invalidate queries to ensure consistency
      queryClient.invalidateQueries({ queryKey: queryKeys.users() });
    },
  });
}

// Background refetch and sync utilities
export function useBackgroundSync() {
  const queryClient = useQueryClient();

  const prefetchUser = useCallback((id: string) => {
    queryClient.prefetchQuery({
      queryKey: queryKeys.user(id),
      queryFn: () => api.get<User>(`/users/${id}`),
      staleTime: 5 * 60 * 1000,
    });
  }, [queryClient]);

  const invalidateUserData = useCallback((id?: string) => {
    if (id) {
      queryClient.invalidateQueries({ queryKey: queryKeys.user(id) });
    } else {
      queryClient.invalidateQueries({ queryKey: queryKeys.users() });
    }
  }, [queryClient]);

  const refreshAllData = useCallback(() => {
    queryClient.invalidateQueries({ queryKey: queryKeys.all });
  }, [queryClient]);

  const clearCache = useCallback(() => {
    queryClient.clear();
  }, [queryClient]);

  return {
    prefetchUser,
    invalidateUserData,
    refreshAllData,
    clearCache,
  };
}
```

### 2. Real-time Data Synchronization
```typescript
// Real-time data synchronization with WebSockets
import { useEffect, useRef, useState } from 'react';
import { useQueryClient } from '@tanstack/react-query';

interface WebSocketMessage {
  type: 'USER_UPDATED' | 'USER_CREATED' | 'USER_DELETED' | 'BULK_UPDATE';
  payload: any;
  timestamp: number;
}

class WebSocketManager {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;
  private listeners = new Set<(message: WebSocketMessage) => void>();
  private heartbeatInterval: NodeJS.Timeout | null = null;

  constructor(private url: string) {}

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      try {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = () => {
          console.log('WebSocket connected');
          this.reconnectAttempts = 0;
          this.startHeartbeat();
          resolve();
        };

        this.ws.onmessage = (event) => {
          try {
            const message: WebSocketMessage = JSON.parse(event.data);
            this.notifyListeners(message);
          } catch (error) {
            console.error('Failed to parse WebSocket message:', error);
          }
        };

        this.ws.onclose = (event) => {
          console.log('WebSocket disconnected:', event.code, event.reason);
          this.stopHeartbeat();
          
          if (!event.wasClean && this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnect();
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

  private reconnect(): void {
    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
    
    console.log(`Reconnecting WebSocket in ${delay}ms (attempt ${this.reconnectAttempts})`);
    
    setTimeout(() => {
      this.connect().catch(() => {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
          this.reconnect();
        }
      });
    }, delay);
  }

  private startHeartbeat(): void {
    this.heartbeatInterval = setInterval(() => {
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({ type: 'ping' }));
      }
    }, 30000); // 30 seconds
  }

  private stopHeartbeat(): void {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
      this.heartbeatInterval = null;
    }
  }

  addListener(callback: (message: WebSocketMessage) => void): () => void {
    this.listeners.add(callback);
    
    return () => {
      this.listeners.delete(callback);
    };
  }

  private notifyListeners(message: WebSocketMessage): void {
    this.listeners.forEach(callback => {
      try {
        callback(message);
      } catch (error) {
        console.error('WebSocket listener error:', error);
      }
    });
  }

  send(message: any): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    }
  }

  disconnect(): void {
    this.stopHeartbeat();
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }

  get isConnected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN;
  }
}

// WebSocket hook for real-time updates
export function useRealTimeSync() {
  const queryClient = useQueryClient();
  const wsManager = useRef<WebSocketManager | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    // Initialize WebSocket connection
    wsManager.current = new WebSocketManager('ws://localhost:8080/realtime');
    
    const connectWebSocket = async () => {
      try {
        await wsManager.current!.connect();
        setIsConnected(true);
      } catch (error) {
        console.error('Failed to connect WebSocket:', error);
        setIsConnected(false);
      }
    };

    connectWebSocket();

    // Listen for real-time updates
    const unsubscribe = wsManager.current.addListener((message) => {
      handleRealTimeUpdate(message, queryClient);
    });

    return () => {
      unsubscribe();
      wsManager.current?.disconnect();
      setIsConnected(false);
    };
  }, [queryClient]);

  // Monitor connection status
  useEffect(() => {
    const checkConnection = setInterval(() => {
      setIsConnected(wsManager.current?.isConnected || false);
    }, 5000);

    return () => clearInterval(checkConnection);
  }, []);

  return {
    isConnected,
    send: (message: any) => wsManager.current?.send(message),
  };
}

function handleRealTimeUpdate(message: WebSocketMessage, queryClient: QueryClient): void {
  switch (message.type) {
    case 'USER_UPDATED':
      // Update specific user in cache
      const updatedUser = message.payload;
      queryClient.setQueryData(queryKeys.user(updatedUser.id), updatedUser);
      
      // Update user in all lists
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old?.users) return old;
          
          return {
            ...old,
            users: old.users.map((user: User) =>
              user.id === updatedUser.id ? updatedUser : user
            ),
          };
        }
      );
      break;

    case 'USER_CREATED':
      // Add new user to cache
      const newUser = message.payload;
      queryClient.setQueryData(queryKeys.user(newUser.id), newUser);
      
      // Add to user lists (only if it matches current filters)
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old?.users) return old;
          
          // Check if user already exists (avoid duplicates)
          const exists = old.users.some((user: User) => user.id === newUser.id);
          if (exists) return old;
          
          return {
            ...old,
            users: [newUser, ...old.users],
            total: old.total + 1,
          };
        }
      );
      break;

    case 'USER_DELETED':
      const deletedUserId = message.payload.id;
      
      // Remove from individual cache
      queryClient.removeQueries({ queryKey: queryKeys.user(deletedUserId) });
      
      // Remove from lists
      queryClient.setQueriesData(
        { queryKey: queryKeys.users() },
        (old: any) => {
          if (!old?.users) return old;
          
          return {
            ...old,
            users: old.users.filter((user: User) => user.id !== deletedUserId),
            total: Math.max(0, old.total - 1),
          };
        }
      );
      break;

    case 'BULK_UPDATE':
      // Handle bulk updates efficiently
      const updates = message.payload.updates;
      
      updates.forEach((update: { id: string; data: Partial<User> }) => {
        queryClient.setQueryData(
          queryKeys.user(update.id),
          (old: User | undefined) => old ? { ...old, ...update.data } : old
        );
      });
      
      // Invalidate lists to trigger refetch with latest data
      queryClient.invalidateQueries({ queryKey: queryKeys.users() });
      break;

    default:
      console.warn('Unknown WebSocket message type:', message.type);
  }
}

// Offline sync and conflict resolution
export function useOfflineSync() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  const [pendingMutations, setPendingMutations] = useState<any[]>([]);
  const queryClient = useQueryClient();

  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      syncPendingMutations();
    };

    const handleOffline = () => {
      setIsOnline(false);
    };

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  const addPendingMutation = useCallback((mutation: any) => {
    setPendingMutations(prev => [...prev, mutation]);
  }, []);

  const syncPendingMutations = useCallback(async () => {
    if (pendingMutations.length === 0) return;

    for (const mutation of pendingMutations) {
      try {
        await mutation.execute();
        setPendingMutations(prev => prev.filter(m => m.id !== mutation.id));
      } catch (error) {
        console.error('Failed to sync pending mutation:', error);
        // Handle conflict resolution here
        await handleConflictResolution(mutation, error);
      }
    }
  }, [pendingMutations]);

  const handleConflictResolution = async (mutation: any, error: any) => {
    if (error.status === 409) {
      // Conflict detected - implement resolution strategy
      const conflictData = error.data;
      
      // Show conflict resolution UI or auto-resolve
      const resolved = await resolveConflict(mutation.data, conflictData);
      
      if (resolved) {
        // Retry with resolved data
        mutation.data = resolved;
        await mutation.execute();
        setPendingMutations(prev => prev.filter(m => m.id !== mutation.id));
      }
    }
  };

  const resolveConflict = async (localData: any, serverData: any): Promise<any> => {
    // Implement conflict resolution strategy
    // For example: last-write-wins, merge strategies, user choice, etc.
    
    // Simple last-write-wins strategy
    const localTimestamp = new Date(localData.updatedAt).getTime();
    const serverTimestamp = new Date(serverData.updatedAt).getTime();
    
    if (localTimestamp > serverTimestamp) {
      return localData;
    } else {
      return serverData;
    }
  };

  return {
    isOnline,
    pendingMutations,
    addPendingMutation,
    syncPendingMutations,
  };
}

// Advanced caching strategies
export function useCacheManagement() {
  const queryClient = useQueryClient();

  const warmupCache = useCallback(async (userIds: string[]) => {
    // Prefetch multiple users in parallel
    const prefetchPromises = userIds.map(id =>
      queryClient.prefetchQuery({
        queryKey: queryKeys.user(id),
        queryFn: () => api.get<User>(`/users/${id}`),
        staleTime: 5 * 60 * 1000,
      })
    );

    await Promise.allSettled(prefetchPromises);
  }, [queryClient]);

  const getCachedData = useCallback(<T>(queryKey: any[]): T | undefined => {
    return queryClient.getQueryData<T>(queryKey);
  }, [queryClient]);

  const setCachedData = useCallback(<T>(queryKey: any[], data: T) => {
    queryClient.setQueryData(queryKey, data);
  }, [queryClient]);

  const invalidateByPattern = useCallback((pattern: any) => {
    queryClient.invalidateQueries({ queryKey: pattern });
  }, [queryClient]);

  const removeStaleQueries = useCallback(() => {
    queryClient.removeQueries({
      predicate: (query) => {
        const staleness = Date.now() - query.state.dataUpdatedAt;
        return staleness > 10 * 60 * 1000; // Remove queries older than 10 minutes
      },
    });
  }, [queryClient]);

  const getCacheStats = useCallback(() => {
    const cache = queryClient.getQueryCache();
    const queries = cache.getAll();
    
    return {
      totalQueries: queries.length,
      activeQueries: queries.filter(q => q.getObserversCount() > 0).length,
      staleQueries: queries.filter(q => q.isStale()).length,
      cacheSize: JSON.stringify(queries).length, // Rough estimate
    };
  }, [queryClient]);

  return {
    warmupCache,
    getCachedData,
    setCachedData,
    invalidateByPattern,
    removeStaleQueries,
    getCacheStats,
  };
}
```

## SWR Integration and Comparison

### 1. SWR Implementation for Server State
```typescript
// SWR implementation with advanced patterns
import useSWR, { mutate, preload } from 'swr';
import useSWRMutation from 'swr/mutation';
import useSWRInfinite from 'swr/infinite';

// Enhanced fetcher with error handling and authentication
const fetcher = async (url: string): Promise<any> => {
  const token = localStorage.getItem('auth_token');
  
  const response = await fetch(url, {
    headers: {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` }),
    },
  });

  if (!response.ok) {
    const error = new Error('API request failed');
    (error as any).status = response.status;
    (error as any).data = await response.json().catch(() => null);
    throw error;
  }

  return response.json();
};

// SWR configuration
export const swrConfig = {
  fetcher,
  revalidateOnFocus: true,
  revalidateOnReconnect: true,
  refreshInterval: 0,
  dedupingInterval: 2000,
  errorRetryCount: 3,
  errorRetryInterval: 5000,
  onError: (error: any) => {
    console.error('SWR Error:', error);
    
    if (error.status === 401) {
      // Handle authentication error
      window.location.href = '/login';
    }
  },
};

// SWR hooks for users
export function useUsersWithSWR(params: {
  page?: number;
  limit?: number;
  search?: string;
  role?: string;
}) {
  const searchParams = new URLSearchParams();
  Object.entries(params).forEach(([key, value]) => {
    if (value !== undefined) {
      searchParams.set(key, String(value));
    }
  });

  const { data, error, isLoading, mutate: revalidate } = useSWR(
    `/api/users?${searchParams}`,
    fetcher,
    {
      revalidateOnFocus: false,
      refreshInterval: 60000, // Refresh every minute
    }
  );

  return {
    users: data?.users || [],
    total: data?.total || 0,
    hasMore: data?.hasMore || false,
    error,
    isLoading,
    revalidate,
  };
}

export function useUserWithSWR(id: string) {
  const { data, error, isLoading, mutate: revalidate } = useSWR(
    id ? `/api/users/${id}` : null,
    fetcher,
    {
      revalidateOnFocus: false,
      dedupingInterval: 5000,
    }
  );

  return {
    user: data,
    error,
    isLoading,
    revalidate,
  };
}

// SWR Infinite for pagination
export function useInfiniteUsersWithSWR(filters: {
  search?: string;
  role?: string;
  limit?: number;
}) {
  const getKey = (pageIndex: number, previousPageData: any) => {
    // Reached the end
    if (previousPageData && !previousPageData.hasMore) return null;

    const searchParams = new URLSearchParams({
      page: String(pageIndex + 1),
      limit: String(filters.limit || 20),
      ...(filters.search && { search: filters.search }),
      ...(filters.role && { role: filters.role }),
    });

    return `/api/users?${searchParams}`;
  };

  const {
    data,
    error,
    isLoading,
    isValidating,
    size,
    setSize,
    mutate: revalidate,
  } = useSWRInfinite(getKey, fetcher, {
    revalidateOnFocus: false,
    revalidateFirstPage: false,
  });

  const users = data ? data.flatMap(page => page.users) : [];
  const hasMore = data ? data[data.length - 1]?.hasMore : false;
  const loadMore = () => setSize(size + 1);

  return {
    users,
    hasMore,
    loadMore,
    error,
    isLoading,
    isValidating,
    revalidate,
  };
}

// SWR Mutations
export function useCreateUserWithSWR() {
  const { trigger, isMutating, error } = useSWRMutation(
    '/api/users',
    async (url: string, { arg }: { arg: CreateUserData }) => {
      const response = await fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${localStorage.getItem('auth_token')}`,
        },
        body: JSON.stringify(arg),
      });

      if (!response.ok) {
        throw new Error('Failed to create user');
      }

      return response.json();
    },
    {
      onSuccess: (newUser) => {
        // Optimistically update cache
        mutate(
          (key) => typeof key === 'string' && key.startsWith('/api/users?'),
          (data: any) => {
            if (!data) return data;
            
            return {
              ...data,
              users: [newUser, ...data.users],
              total: data.total + 1,
            };
          },
          { revalidate: false }
        );
      },
    }
  );

  return {
    createUser: trigger,
    isCreating: isMutating,
    error,
  };
}

export function useUpdateUserWithSWR() {
  const { trigger, isMutating, error } = useSWRMutation(
    '/api/users',
    async (url: string, { arg }: { arg: { id: string; updates: Partial<User> } }) => {
      const response = await fetch(`${url}/${arg.id}`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${localStorage.getItem('auth_token')}`,
        },
        body: JSON.stringify(arg.updates),
      });

      if (!response.ok) {
        throw new Error('Failed to update user');
      }

      return response.json();
    },
    {
      onSuccess: (updatedUser, { arg }) => {
        // Update individual user cache
        mutate(`/api/users/${arg.id}`, updatedUser, { revalidate: false });
        
        // Update user in lists
        mutate(
          (key) => typeof key === 'string' && key.startsWith('/api/users?'),
          (data: any) => {
            if (!data?.users) return data;
            
            return {
              ...data,
              users: data.users.map((user: User) =>
                user.id === updatedUser.id ? updatedUser : user
              ),
            };
          },
          { revalidate: false }
        );
      },
    }
  );

  return {
    updateUser: trigger,
    isUpdating: isMutating,
    error,
  };
}

// SWR with optimistic updates
export function useOptimisticUpdateSWR() {
  const updateUserOptimistic = async (id: string, updates: Partial<User>) => {
    const userKey = `/api/users/${id}`;
    
    // Get current data
    const currentUser = mutate(userKey);
    
    // Optimistic update
    mutate(
      userKey,
      (current: User) => ({ ...current, ...updates }),
      { revalidate: false }
    );

    // Update in lists
    mutate(
      (key) => typeof key === 'string' && key.startsWith('/api/users?'),
      (data: any) => {
        if (!data?.users) return data;
        
        return {
          ...data,
          users: data.users.map((user: User) =>
            user.id === id ? { ...user, ...updates } : user
          ),
        };
      },
      { revalidate: false }
    );

    try {
      // Make actual API call
      const response = await fetch(`/api/users/${id}`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${localStorage.getItem('auth_token')}`,
        },
        body: JSON.stringify(updates),
      });

      if (!response.ok) {
        throw new Error('Update failed');
      }

      const updatedUser = await response.json();
      
      // Update with real data
      mutate(userKey, updatedUser, { revalidate: false });
      
      return updatedUser;
    } catch (error) {
      // Revert optimistic update
      mutate(userKey, currentUser, { revalidate: false });
      throw error;
    }
  };

  return { updateUserOptimistic };
}

// SWR cache management utilities
export function useSWRCacheManagement() {
  const prefetchUser = useCallback((id: string) => {
    preload(`/api/users/${id}`, fetcher);
  }, []);

  const invalidateUsers = useCallback(() => {
    mutate((key) => typeof key === 'string' && key.startsWith('/api/users'));
  }, []);

  const clearCache = useCallback(() => {
    // SWR doesn't have a built-in clear all method
    // You would need to track keys manually or use a cache implementation
    console.warn('SWR clear cache not implemented');
  }, []);

  return {
    prefetchUser,
    invalidateUsers,
    clearCache,
  };
}

// React Query vs SWR comparison hook
export function useServerStateComparison() {
  // This would be used for A/B testing or migration
  const useReactQuery = true; // Feature flag
  
  if (useReactQuery) {
    return {
      users: useUsers({ page: 1, limit: 20 }),
      createUser: useCreateUser(),
      updateUser: useUpdateUser(),
      deleteUser: useDeleteUser(),
      library: 'react-query',
    };
  } else {
    return {
      users: useUsersWithSWR({ page: 1, limit: 20 }),
      createUser: useCreateUserWithSWR(),
      updateUser: useUpdateUserWithSWR(),
      deleteUser: null, // Would implement if needed
      library: 'swr',
    };
  }
}
```

## Interview-Ready Summary

**Server state management requires specialized patterns:**

1. **React Query (TanStack Query)** - Comprehensive caching, background updates, optimistic updates, infinite queries, devtools
2. **SWR** - Lightweight, simple API, revalidation strategies, mutation support, good TypeScript support
3. **Real-time Sync** - WebSocket integration, conflict resolution, offline support, automatic cache updates
4. **Advanced Patterns** - Query key factories, optimistic updates, background prefetching, cache warming

**Key differences:**
- **React Query**: More features, larger bundle, extensive configurability, better devtools, stronger typing
- **SWR**: Smaller bundle, simpler API, automatic revalidation, good for straightforward use cases

**Server state characteristics:**
- **Asynchronous** - Always fetched from external source
- **Shared** - Multiple components may need same data
- **Stale** - Can become outdated and needs revalidation
- **Cacheable** - Benefits from intelligent caching strategies

**Advanced patterns:** Optimistic updates for better UX, background sync for performance, real-time updates for fresh data, conflict resolution for offline scenarios, query invalidation for consistency.

**Best practices:** Use query keys strategically, implement proper error handling, leverage optimistic updates, handle offline scenarios, monitor cache performance, use TypeScript for type safety, implement proper loading states.