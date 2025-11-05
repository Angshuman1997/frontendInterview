# Advanced Redux Patterns and Redux Toolkit Query

Modern Redux development leverages Redux Toolkit (RTK) and RTK Query for efficient state management and API data fetching. This guide covers advanced patterns, performance optimization, and best practices for enterprise-level applications.

## Redux Toolkit Advanced Patterns

### 1. Advanced Slice Configuration
```typescript
// Advanced slice with complex state normalization
import { createSlice, createEntityAdapter, PayloadAction, createSelector } from '@reduxjs/toolkit';
import { RootState } from '../store';

// Entity adapter for normalized state management
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'moderator';
  lastActive: string;
  preferences: {
    theme: 'light' | 'dark';
    notifications: boolean;
    language: string;
  };
}

interface UserFilters {
  role?: User['role'];
  search?: string;
  isActive?: boolean;
}

interface UserState {
  users: ReturnType<typeof usersAdapter.getInitialState>;
  filters: UserFilters;
  loading: {
    fetch: boolean;
    create: boolean;
    update: boolean;
    delete: boolean;
  };
  error: {
    fetch: string | null;
    create: string | null;
    update: string | null;
    delete: string | null;
  };
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    hasMore: boolean;
  };
  optimisticUpdates: Record<string, Partial<User>>;
}

// Entity adapter for normalized state
const usersAdapter = createEntityAdapter<User>({
  selectId: (user) => user.id,
  sortComparer: (a, b) => b.lastActive.localeCompare(a.lastActive),
});

const initialState: UserState = {
  users: usersAdapter.getInitialState(),
  filters: {},
  loading: {
    fetch: false,
    create: false,
    update: false,
    delete: false,
  },
  error: {
    fetch: null,
    create: null,
    update: null,
    delete: null,
  },
  pagination: {
    page: 1,
    pageSize: 20,
    total: 0,
    hasMore: true,
  },
  optimisticUpdates: {},
};

const usersSlice = createSlice({
  name: 'users',
  initialState,
  reducers: {
    // Loading states
    setLoading: (state, action: PayloadAction<{ type: keyof UserState['loading']; loading: boolean }>) => {
      state.loading[action.payload.type] = action.payload.loading;
    },

    // Error handling
    setError: (state, action: PayloadAction<{ type: keyof UserState['error']; error: string | null }>) => {
      state.error[action.payload.type] = action.payload.error;
    },
    
    clearErrors: (state) => {
      Object.keys(state.error).forEach(key => {
        state.error[key as keyof UserState['error']] = null;
      });
    },

    // Users CRUD operations
    usersReceived: (state, action: PayloadAction<{ users: User[]; replace?: boolean }>) => {
      if (action.payload.replace) {
        usersAdapter.setAll(state.users, action.payload.users);
      } else {
        usersAdapter.upsertMany(state.users, action.payload.users);
      }
    },

    userAdded: (state, action: PayloadAction<User>) => {
      usersAdapter.addOne(state.users, action.payload);
      // Clear optimistic update if exists
      delete state.optimisticUpdates[action.payload.id];
    },

    userUpdated: (state, action: PayloadAction<{ id: string; changes: Partial<User> }>) => {
      usersAdapter.updateOne(state.users, action.payload);
      // Clear optimistic update
      delete state.optimisticUpdates[action.payload.id];
    },

    userRemoved: (state, action: PayloadAction<string>) => {
      usersAdapter.removeOne(state.users, action.payload);
      delete state.optimisticUpdates[action.payload];
    },

    // Optimistic updates
    userOptimisticUpdate: (state, action: PayloadAction<{ id: string; changes: Partial<User> }>) => {
      state.optimisticUpdates[action.payload.id] = action.payload.changes;
    },

    clearOptimisticUpdate: (state, action: PayloadAction<string>) => {
      delete state.optimisticUpdates[action.payload];
    },

    // Filters and pagination
    setFilters: (state, action: PayloadAction<Partial<UserFilters>>) => {
      state.filters = { ...state.filters, ...action.payload };
      state.pagination.page = 1; // Reset pagination when filters change
    },

    clearFilters: (state) => {
      state.filters = {};
      state.pagination.page = 1;
    },

    setPagination: (state, action: PayloadAction<Partial<UserState['pagination']>>) => {
      state.pagination = { ...state.pagination, ...action.payload };
    },

    // Batch operations
    batchUpdateUsers: (state, action: PayloadAction<Array<{ id: string; changes: Partial<User> }>>) => {
      action.payload.forEach(update => {
        usersAdapter.updateOne(state.users, update);
      });
    },

    // Reset state
    resetUsersState: () => initialState,
  },
});

export const {
  setLoading,
  setError,
  clearErrors,
  usersReceived,
  userAdded,
  userUpdated,
  userRemoved,
  userOptimisticUpdate,
  clearOptimisticUpdate,
  setFilters,
  clearFilters,
  setPagination,
  batchUpdateUsers,
  resetUsersState,
} = usersSlice.actions;

// Entity adapter selectors
export const {
  selectAll: selectAllUsers,
  selectById: selectUserById,
  selectIds: selectUserIds,
  selectEntities: selectUserEntities,
  selectTotal: selectUsersTotal,
} = usersAdapter.getSelectors((state: RootState) => state.users.users);

// Advanced memoized selectors
export const selectUsersState = (state: RootState) => state.users;

export const selectUsersWithOptimisticUpdates = createSelector(
  [selectAllUsers, selectUsersState],
  (users, usersState) => {
    return users.map(user => ({
      ...user,
      ...usersState.optimisticUpdates[user.id],
    }));
  }
);

export const selectFilteredUsers = createSelector(
  [selectUsersWithOptimisticUpdates, selectUsersState],
  (users, usersState) => {
    const { filters } = usersState;
    
    return users.filter(user => {
      if (filters.role && user.role !== filters.role) return false;
      if (filters.search) {
        const searchLower = filters.search.toLowerCase();
        const matchesSearch = 
          user.name.toLowerCase().includes(searchLower) ||
          user.email.toLowerCase().includes(searchLower);
        if (!matchesSearch) return false;
      }
      if (filters.isActive !== undefined) {
        const isActive = new Date(user.lastActive) > new Date(Date.now() - 24 * 60 * 60 * 1000);
        if (isActive !== filters.isActive) return false;
      }
      return true;
    });
  }
);

export const selectPaginatedUsers = createSelector(
  [selectFilteredUsers, selectUsersState],
  (filteredUsers, usersState) => {
    const { pagination } = usersState;
    const startIndex = (pagination.page - 1) * pagination.pageSize;
    const endIndex = startIndex + pagination.pageSize;
    
    return {
      users: filteredUsers.slice(startIndex, endIndex),
      totalFiltered: filteredUsers.length,
      hasMore: endIndex < filteredUsers.length,
    };
  }
);

export const selectUsersByRole = createSelector(
  [selectUsersWithOptimisticUpdates],
  (users) => {
    return users.reduce((acc, user) => {
      if (!acc[user.role]) {
        acc[user.role] = [];
      }
      acc[user.role].push(user);
      return acc;
    }, {} as Record<User['role'], User[]>);
  }
);

export const selectUsersStats = createSelector(
  [selectAllUsers],
  (users) => {
    const now = new Date();
    const dayAgo = new Date(now.getTime() - 24 * 60 * 60 * 1000);
    const weekAgo = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000);
    
    return {
      total: users.length,
      active24h: users.filter(user => new Date(user.lastActive) > dayAgo).length,
      active7d: users.filter(user => new Date(user.lastActive) > weekAgo).length,
      byRole: users.reduce((acc, user) => {
        acc[user.role] = (acc[user.role] || 0) + 1;
        return acc;
      }, {} as Record<User['role'], number>),
    };
  }
);

export default usersSlice.reducer;
```

### 2. Advanced Async Thunks and Error Handling
```typescript
// Advanced async thunks with comprehensive error handling
import { createAsyncThunk, isRejectedWithValue } from '@reduxjs/toolkit';
import { RootState } from '../store';

// API service with retry logic
class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public code?: string,
    public details?: any
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

const apiRequest = async <T>(
  url: string,
  options: RequestInit = {},
  retries = 3,
  retryDelay = 1000
): Promise<T> => {
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const response = await fetch(url, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
        },
      });

      if (!response.ok) {
        const errorData = await response.json().catch(() => null);
        throw new ApiError(
          errorData?.message || `HTTP ${response.status}`,
          response.status,
          errorData?.code,
          errorData
        );
      }

      return await response.json();
    } catch (error) {
      if (attempt === retries || !(error instanceof ApiError) || error.status < 500) {
        throw error;
      }
      
      // Wait before retry
      await new Promise(resolve => setTimeout(resolve, retryDelay * Math.pow(2, attempt)));
    }
  }
  
  throw new Error('Max retries exceeded');
};

// Generic async thunk creator with error handling
const createAsyncThunkWithErrorHandling = <Returned, ThunkArg = void>(
  typePrefix: string,
  payloadCreator: (arg: ThunkArg, thunkAPI: any) => Promise<Returned>
) => {
  return createAsyncThunk<Returned, ThunkArg, {
    state: RootState;
    rejectValue: {
      message: string;
      status?: number;
      code?: string;
      details?: any;
    };
  }>(
    typePrefix,
    async (arg, thunkAPI) => {
      try {
        return await payloadCreator(arg, thunkAPI);
      } catch (error) {
        if (error instanceof ApiError) {
          return thunkAPI.rejectWithValue({
            message: error.message,
            status: error.status,
            code: error.code,
            details: error.details,
          });
        }
        
        return thunkAPI.rejectWithValue({
          message: error instanceof Error ? error.message : 'An unknown error occurred',
        });
      }
    }
  );
};

// Advanced thunks with optimistic updates
export const fetchUsers = createAsyncThunkWithErrorHandling<
  { users: User[]; total: number; hasMore: boolean },
  { page?: number; pageSize?: number; filters?: UserFilters; force?: boolean }
>(
  'users/fetchUsers',
  async ({ page = 1, pageSize = 20, filters = {}, force = false }, { getState, dispatch }) => {
    const state = getState() as RootState;
    
    // Check if we already have this data and it's fresh
    if (!force && state.users.pagination.page === page && !state.users.loading.fetch) {
      return {
        users: selectFilteredUsers(state).slice((page - 1) * pageSize, page * pageSize),
        total: selectFilteredUsers(state).length,
        hasMore: page * pageSize < selectFilteredUsers(state).length,
      };
    }

    dispatch(setLoading({ type: 'fetch', loading: true }));
    dispatch(setError({ type: 'fetch', error: null }));

    const queryParams = new URLSearchParams({
      page: page.toString(),
      pageSize: pageSize.toString(),
      ...Object.fromEntries(
        Object.entries(filters).filter(([_, value]) => value !== undefined)
      ),
    });

    const response = await apiRequest<{
      users: User[];
      total: number;
      hasMore: boolean;
    }>(`/api/users?${queryParams}`);

    dispatch(setPagination({
      page,
      pageSize,
      total: response.total,
      hasMore: response.hasMore,
    }));

    return response;
  }
);

export const createUser = createAsyncThunkWithErrorHandling<
  User,
  Omit<User, 'id' | 'lastActive'>
>(
  'users/createUser',
  async (userData, { dispatch }) => {
    // Optimistic update
    const tempId = `temp_${Date.now()}`;
    const optimisticUser: User = {
      ...userData,
      id: tempId,
      lastActive: new Date().toISOString(),
    };

    dispatch(userOptimisticUpdate({ id: tempId, changes: optimisticUser }));
    dispatch(setLoading({ type: 'create', loading: true }));
    dispatch(setError({ type: 'create', error: null }));

    try {
      const newUser = await apiRequest<User>('/api/users', {
        method: 'POST',
        body: JSON.stringify(userData),
      });

      // Remove optimistic update and add real user
      dispatch(clearOptimisticUpdate(tempId));
      return newUser;
    } catch (error) {
      // Remove failed optimistic update
      dispatch(clearOptimisticUpdate(tempId));
      throw error;
    }
  }
);

export const updateUser = createAsyncThunkWithErrorHandling<
  User,
  { id: string; changes: Partial<User> }
>(
  'users/updateUser',
  async ({ id, changes }, { dispatch, getState }) => {
    // Optimistic update
    dispatch(userOptimisticUpdate({ id, changes }));
    dispatch(setLoading({ type: 'update', loading: true }));
    dispatch(setError({ type: 'update', error: null }));

    try {
      const updatedUser = await apiRequest<User>(`/api/users/${id}`, {
        method: 'PATCH',
        body: JSON.stringify(changes),
      });

      return updatedUser;
    } catch (error) {
      // Revert optimistic update on failure
      dispatch(clearOptimisticUpdate(id));
      throw error;
    }
  }
);

export const deleteUser = createAsyncThunkWithErrorHandling<
  string,
  string
>(
  'users/deleteUser',
  async (userId, { dispatch }) => {
    dispatch(setLoading({ type: 'delete', loading: true }));
    dispatch(setError({ type: 'delete', error: null }));

    await apiRequest(`/api/users/${userId}`, {
      method: 'DELETE',
    });

    return userId;
  }
);

// Batch operations
export const batchUpdateUsersThunk = createAsyncThunkWithErrorHandling<
  User[],
  Array<{ id: string; changes: Partial<User> }>
>(
  'users/batchUpdateUsers',
  async (updates, { dispatch }) => {
    // Optimistic updates
    updates.forEach(update => {
      dispatch(userOptimisticUpdate(update));
    });

    try {
      const updatedUsers = await apiRequest<User[]>('/api/users/batch', {
        method: 'PATCH',
        body: JSON.stringify({ updates }),
      });

      return updatedUsers;
    } catch (error) {
      // Revert all optimistic updates on failure
      updates.forEach(update => {
        dispatch(clearOptimisticUpdate(update.id));
      });
      throw error;
    }
  }
);

// Add extra reducers to handle async thunk states
const usersSliceWithAsyncReducers = usersSlice.reducer;

// In the slice, add extraReducers:
/*
extraReducers: (builder) => {
  builder
    // Fetch users
    .addCase(fetchUsers.fulfilled, (state, action) => {
      state.loading.fetch = false;
      usersAdapter.upsertMany(state.users, action.payload.users);
    })
    .addCase(fetchUsers.rejected, (state, action) => {
      state.loading.fetch = false;
      state.error.fetch = action.payload?.message || 'Failed to fetch users';
    })
    
    // Create user
    .addCase(createUser.fulfilled, (state, action) => {
      state.loading.create = false;
      usersAdapter.addOne(state.users, action.payload);
    })
    .addCase(createUser.rejected, (state, action) => {
      state.loading.create = false;
      state.error.create = action.payload?.message || 'Failed to create user';
    })
    
    // Update user
    .addCase(updateUser.fulfilled, (state, action) => {
      state.loading.update = false;
      usersAdapter.updateOne(state.users, {
        id: action.payload.id,
        changes: action.payload,
      });
    })
    .addCase(updateUser.rejected, (state, action) => {
      state.loading.update = false;
      state.error.update = action.payload?.message || 'Failed to update user';
    })
    
    // Delete user
    .addCase(deleteUser.fulfilled, (state, action) => {
      state.loading.delete = false;
      usersAdapter.removeOne(state.users, action.payload);
    })
    .addCase(deleteUser.rejected, (state, action) => {
      state.loading.delete = false;
      state.error.delete = action.payload?.message || 'Failed to delete user';
    })
    
    // Batch update
    .addCase(batchUpdateUsersThunk.fulfilled, (state, action) => {
      action.payload.forEach(user => {
        usersAdapter.updateOne(state.users, {
          id: user.id,
          changes: user,
        });
        delete state.optimisticUpdates[user.id];
      });
    })
    .addCase(batchUpdateUsersThunk.rejected, (state, action) => {
      state.error.update = action.payload?.message || 'Failed to batch update users';
    });
}
*/
```

## RTK Query - Advanced Data Fetching

### 1. Comprehensive API Slice with Advanced Features
```typescript
// Advanced RTK Query API slice
import { createApi, fetchBaseQuery, retry } from '@reduxjs/toolkit/query/react';
import type { BaseQueryFn, FetchArgs, FetchBaseQueryError } from '@reduxjs/toolkit/query';
import { RootState } from '../store';

// Enhanced base query with authentication and error handling
const baseQuery = fetchBaseQuery({
  baseUrl: '/api',
  prepareHeaders: (headers, { getState }) => {
    const state = getState() as RootState;
    const token = state.auth.token;
    
    if (token) {
      headers.set('authorization', `Bearer ${token}`);
    }
    
    headers.set('accept', 'application/json');
    headers.set('content-type', 'application/json');
    
    return headers;
  },
  credentials: 'include',
});

// Enhanced base query with retry and token refresh
const baseQueryWithReauth: BaseQueryFn<
  string | FetchArgs,
  unknown,
  FetchBaseQueryError
> = async (args, api, extraOptions) => {
  // Add retry logic for network errors
  const baseQueryWithRetry = retry(baseQuery, { maxRetries: 3 });
  
  let result = await baseQueryWithRetry(args, api, extraOptions);
  
  // Handle token expiration
  if (result.error && result.error.status === 401) {
    console.log('Sending refresh token...');
    
    // Try to refresh the token
    const refreshResult = await baseQuery(
      { url: '/auth/refresh', method: 'POST' },
      api,
      extraOptions
    );
    
    if (refreshResult.data) {
      // Store the new token
      api.dispatch(tokenReceived((refreshResult.data as any).token));
      
      // Retry the original query
      result = await baseQueryWithRetry(args, api, extraOptions);
    } else {
      // Refresh failed, logout user
      api.dispatch(logoutUser());
    }
  }
  
  return result;
};

// API slice with advanced features
export const api = createApi({
  reducerPath: 'api',
  baseQuery: baseQueryWithReauth,
  tagTypes: [
    'User', 
    'Post', 
    'Comment', 
    'Tag',
    'UserProfile',
    'UserStats',
    'Notification'
  ],
  endpoints: (builder) => ({
    // Users endpoints with advanced caching
    getUsers: builder.query<
      { users: User[]; total: number; hasMore: boolean },
      {
        page?: number;
        pageSize?: number;
        filters?: UserFilters;
        sortBy?: string;
        sortOrder?: 'asc' | 'desc';
      }
    >({
      query: ({ page = 1, pageSize = 20, filters = {}, sortBy = 'lastActive', sortOrder = 'desc' }) => {
        const params = new URLSearchParams({
          page: page.toString(),
          pageSize: pageSize.toString(),
          sortBy,
          sortOrder,
          ...Object.fromEntries(
            Object.entries(filters).filter(([_, value]) => value !== undefined)
          ),
        });
        
        return `/users?${params}`;
      },
      providesTags: (result) =>
        result
          ? [
              ...result.users.map(({ id }) => ({ type: 'User' as const, id })),
              { type: 'User', id: 'LIST' },
            ]
          : [{ type: 'User', id: 'LIST' }],
      transformResponse: (response: any, meta, arg) => {
        // Transform and normalize response
        return {
          users: response.data.users,
          total: response.data.total,
          hasMore: response.data.hasMore,
        };
      },
      // Advanced caching configuration
      keepUnusedDataFor: 60, // seconds
      refetchOnMountOrArgChange: 30, // seconds
    }),

    getUserById: builder.query<User, string>({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: 'User', id }],
      transformResponse: (response: any) => response.data,
    }),

    // Infinite query pattern
    getUsersInfinite: builder.query<
      { users: User[]; nextCursor?: string; hasMore: boolean },
      { cursor?: string; limit?: number; filters?: UserFilters }
    >({
      query: ({ cursor, limit = 20, filters = {} }) => {
        const params = new URLSearchParams({
          limit: limit.toString(),
          ...(cursor && { cursor }),
          ...Object.fromEntries(
            Object.entries(filters).filter(([_, value]) => value !== undefined)
          ),
        });
        
        return `/users/infinite?${params}`;
      },
      // Merge pages for infinite scrolling
      serializeQueryArgs: ({ queryArgs, endpointDefinition, endpointName }) => {
        const { cursor, ...rest } = queryArgs;
        return endpointName + JSON.stringify(rest);
      },
      merge: (currentCache, newItems) => {
        if (!currentCache) return newItems;
        
        return {
          users: [...currentCache.users, ...newItems.users],
          nextCursor: newItems.nextCursor,
          hasMore: newItems.hasMore,
        };
      },
      forceRefetch({ currentArg, previousArg }) {
        return currentArg?.cursor !== previousArg?.cursor;
      },
      providesTags: ['User'],
    }),

    // Mutations with optimistic updates
    createUser: builder.mutation<User, Omit<User, 'id' | 'lastActive'>>({
      query: (userData) => ({
        url: '/users',
        method: 'POST',
        body: userData,
      }),
      invalidatesTags: [{ type: 'User', id: 'LIST' }],
      // Optimistic update
      async onQueryStarted(userData, { dispatch, queryFulfilled }) {
        const tempId = `temp_${Date.now()}`;
        const optimisticUser: User = {
          ...userData,
          id: tempId,
          lastActive: new Date().toISOString(),
        };

        // Optimistically update the cache
        const patchResult = dispatch(
          api.util.updateQueryData('getUsers', { page: 1 }, (draft) => {
            draft.users.unshift(optimisticUser);
            draft.total += 1;
          })
        );

        try {
          const { data: newUser } = await queryFulfilled;
          
          // Replace optimistic update with real data
          dispatch(
            api.util.updateQueryData('getUsers', { page: 1 }, (draft) => {
              const index = draft.users.findIndex(user => user.id === tempId);
              if (index !== -1) {
                draft.users[index] = newUser;
              }
            })
          );
        } catch {
          // Revert optimistic update on error
          patchResult.undo();
        }
      },
      transformResponse: (response: any) => response.data,
    }),

    updateUser: builder.mutation<User, { id: string; changes: Partial<User> }>({
      query: ({ id, changes }) => ({
        url: `/users/${id}`,
        method: 'PATCH',
        body: changes,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'User', id }],
      // Optimistic update
      async onQueryStarted({ id, changes }, { dispatch, queryFulfilled }) {
        const patchResults = [];

        // Update all relevant queries
        patchResults.push(
          dispatch(
            api.util.updateQueryData('getUserById', id, (draft) => {
              Object.assign(draft, changes);
            })
          )
        );

        // Update user in lists
        patchResults.push(
          dispatch(
            api.util.updateQueryData('getUsers', { page: 1 }, (draft) => {
              const user = draft.users.find(u => u.id === id);
              if (user) {
                Object.assign(user, changes);
              }
            })
          )
        );

        try {
          await queryFulfilled;
        } catch {
          // Revert all optimistic updates on error
          patchResults.forEach(patchResult => patchResult.undo());
        }
      },
      transformResponse: (response: any) => response.data,
    }),

    deleteUser: builder.mutation<void, string>({
      query: (id) => ({
        url: `/users/${id}`,
        method: 'DELETE',
      }),
      invalidatesTags: (result, error, id) => [
        { type: 'User', id },
        { type: 'User', id: 'LIST' },
      ],
      // Optimistic update
      async onQueryStarted(id, { dispatch, queryFulfilled }) {
        const patchResults = [];

        // Remove from all user lists
        patchResults.push(
          dispatch(
            api.util.updateQueryData('getUsers', { page: 1 }, (draft) => {
              draft.users = draft.users.filter(user => user.id !== id);
              draft.total -= 1;
            })
          )
        );

        try {
          await queryFulfilled;
        } catch {
          // Revert optimistic updates on error
          patchResults.forEach(patchResult => patchResult.undo());
        }
      },
    }),

    // Batch operations
    batchUpdateUsers: builder.mutation<
      User[],
      Array<{ id: string; changes: Partial<User> }>
    >({
      query: (updates) => ({
        url: '/users/batch',
        method: 'PATCH',
        body: { updates },
      }),
      invalidatesTags: (result, error, updates) => [
        ...updates.map(({ id }) => ({ type: 'User' as const, id })),
        { type: 'User', id: 'LIST' },
      ],
      transformResponse: (response: any) => response.data,
    }),

    // Real-time subscriptions (WebSocket)
    subscribeToUserUpdates: builder.query<User, string>({
      queryFn: () => ({ data: {} as User }), // Initial empty data
      async onCacheEntryAdded(
        userId,
        { updateCachedData, cacheDataLoaded, cacheEntryRemoved }
      ) {
        const ws = new WebSocket(`ws://localhost:8080/users/${userId}/subscribe`);
        
        try {
          await cacheDataLoaded;
          
          const listener = (event: MessageEvent) => {
            const data = JSON.parse(event.data);
            
            updateCachedData((draft) => {
              Object.assign(draft, data);
            });

            // Also update other cached queries
            dispatch(
              api.util.updateQueryData('getUserById', userId, (draft) => {
                Object.assign(draft, data);
              })
            );
          };

          ws.addEventListener('message', listener);
        } catch {
          // Handle cache entry removed
        }

        await cacheEntryRemoved;
        ws.close();
      },
    }),

    // File upload with progress
    uploadUserAvatar: builder.mutation<
      { url: string },
      { userId: string; file: File; onProgress?: (progress: number) => void }
    >({
      queryFn: async ({ userId, file, onProgress }) => {
        const formData = new FormData();
        formData.append('avatar', file);

        return new Promise((resolve, reject) => {
          const xhr = new XMLHttpRequest();

          xhr.upload.onprogress = (event) => {
            if (event.lengthComputable && onProgress) {
              const progress = (event.loaded / event.total) * 100;
              onProgress(progress);
            }
          };

          xhr.onload = () => {
            if (xhr.status === 200) {
              const response = JSON.parse(xhr.responseText);
              resolve({ data: response.data });
            } else {
              reject({ error: { status: xhr.status, data: xhr.responseText } });
            }
          };

          xhr.onerror = () => {
            reject({ error: { status: 0, data: 'Network error' } });
          };

          xhr.open('POST', `/api/users/${userId}/avatar`);
          xhr.send(formData);
        });
      },
      invalidatesTags: (result, error, { userId }) => [{ type: 'User', id: userId }],
    }),

    // Search with debouncing
    searchUsers: builder.query<User[], { query: string; limit?: number }>({
      query: ({ query, limit = 10 }) => `/users/search?q=${encodeURIComponent(query)}&limit=${limit}`,
      providesTags: ['User'],
      transformResponse: (response: any) => response.data,
      // Only refetch if query changes
      serializeQueryArgs: ({ queryArgs }) => queryArgs.query,
    }),
  }),
});

// Export hooks
export const {
  useGetUsersQuery,
  useGetUserByIdQuery,
  useGetUsersInfiniteQuery,
  useCreateUserMutation,
  useUpdateUserMutation,
  useDeleteUserMutation,
  useBatchUpdateUsersMutation,
  useSubscribeToUserUpdatesQuery,
  useUploadUserAvatarMutation,
  useSearchUsersQuery,
  useLazyGetUsersQuery,
  useLazySearchUsersQuery,
} = api;

// Export utility functions
export const {
  updateQueryData,
  upsertQueryData,
  invalidateTags,
  resetApiState,
} = api.util;
```

### 2. Advanced RTK Query Patterns and Optimizations
```typescript
// Advanced patterns and utilities for RTK Query
import { useCallback, useEffect, useRef, useState } from 'react';
import { useDispatch } from 'react-redux';
import { debounce } from 'lodash';

// Custom hook for infinite scrolling with RTK Query
export function useInfiniteUsers(filters: UserFilters = {}) {
  const [allUsers, setAllUsers] = useState<User[]>([]);
  const [cursor, setCursor] = useState<string | undefined>();
  const [hasMore, setHasMore] = useState(true);

  const {
    data,
    isLoading,
    isFetching,
    error,
    refetch,
  } = useGetUsersInfiniteQuery({ cursor, filters, limit: 20 });

  useEffect(() => {
    if (data) {
      if (cursor) {
        // Append new data
        setAllUsers(prev => [...prev, ...data.users]);
      } else {
        // Replace data (new search)
        setAllUsers(data.users);
      }
      setHasMore(data.hasMore);
    }
  }, [data, cursor]);

  const loadMore = useCallback(() => {
    if (data?.nextCursor && hasMore && !isFetching) {
      setCursor(data.nextCursor);
    }
  }, [data?.nextCursor, hasMore, isFetching]);

  const reset = useCallback(() => {
    setAllUsers([]);
    setCursor(undefined);
    setHasMore(true);
  }, []);

  useEffect(() => {
    // Reset when filters change
    reset();
  }, [filters, reset]);

  return {
    users: allUsers,
    isLoading,
    isFetching,
    error,
    hasMore,
    loadMore,
    reset,
    refetch,
  };
}

// Debounced search hook
export function useDebouncedSearch(delay = 300) {
  const [searchQuery, setSearchQuery] = useState('');
  const [debouncedQuery, setDebouncedQuery] = useState('');

  const debouncedSetQuery = useRef(
    debounce((query: string) => {
      setDebouncedQuery(query);
    }, delay)
  ).current;

  useEffect(() => {
    debouncedSetQuery(searchQuery);
    
    return () => {
      debouncedSetQuery.cancel();
    };
  }, [searchQuery, debouncedSetQuery]);

  const {
    data: searchResults,
    isLoading: isSearching,
    error: searchError,
  } = useSearchUsersQuery(
    { query: debouncedQuery },
    { skip: debouncedQuery.length < 2 }
  );

  return {
    searchQuery,
    setSearchQuery,
    searchResults: searchResults || [],
    isSearching: isSearching && debouncedQuery.length >= 2,
    searchError,
  };
}

// Optimistic updates hook
export function useOptimisticUpdates() {
  const dispatch = useDispatch();

  const optimisticUpdate = useCallback(<T>(
    queryKey: string,
    queryArgs: any,
    updateFn: (draft: T) => void,
    revertFn?: (draft: T) => void
  ) => {
    const patchResult = dispatch(
      api.util.updateQueryData(queryKey as any, queryArgs, updateFn as any)
    );

    return {
      revert: () => patchResult.undo(),
      patch: patchResult,
    };
  }, [dispatch]);

  return { optimisticUpdate };
}

// Cache management utilities
export function useCacheManagement() {
  const dispatch = useDispatch();

  const invalidateCache = useCallback((tags: string[]) => {
    dispatch(api.util.invalidateTags(tags as any));
  }, [dispatch]);

  const prefetchData = useCallback((endpoint: string, args: any) => {
    dispatch(api.util.prefetch(endpoint as any, args));
  }, [dispatch]);

  const clearCache = useCallback(() => {
    dispatch(api.util.resetApiState());
  }, [dispatch]);

  return {
    invalidateCache,
    prefetchData,
    clearCache,
  };
}

// Real-time data synchronization
export function useRealTimeSync() {
  const dispatch = useDispatch();

  useEffect(() => {
    const ws = new WebSocket('ws://localhost:8080/realtime');

    ws.onmessage = (event) => {
      const { type, data } = JSON.parse(event.data);

      switch (type) {
        case 'USER_UPDATED':
          // Update specific user in cache
          dispatch(
            api.util.updateQueryData('getUserById', data.id, (draft) => {
              Object.assign(draft, data);
            })
          );
          
          // Update user in lists
          dispatch(
            api.util.updateQueryData('getUsers', { page: 1 }, (draft) => {
              const userIndex = draft.users.findIndex(u => u.id === data.id);
              if (userIndex !== -1) {
                Object.assign(draft.users[userIndex], data);
              }
            })
          );
          break;

        case 'USER_CREATED':
          // Invalidate user lists to refetch
          dispatch(api.util.invalidateTags([{ type: 'User', id: 'LIST' }]));
          break;

        case 'USER_DELETED':
          // Remove user from cache
          dispatch(
            api.util.updateQueryData('getUsers', { page: 1 }, (draft) => {
              draft.users = draft.users.filter(u => u.id !== data.id);
              draft.total -= 1;
            })
          );
          break;

        default:
          console.warn('Unknown real-time event type:', type);
      }
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    return () => {
      ws.close();
    };
  }, [dispatch]);
}

// Background sync for offline support
export function useBackgroundSync() {
  const dispatch = useDispatch();
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      // Refetch critical data when coming back online
      dispatch(api.util.invalidateTags(['User']));
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
  }, [dispatch]);

  return { isOnline };
}

// Performance monitoring for RTK Query
export function useQueryPerformance() {
  const [metrics, setMetrics] = useState<{
    queryCounts: Record<string, number>;
    averageResponseTimes: Record<string, number>;
    errorRates: Record<string, number>;
  }>({
    queryCounts: {},
    averageResponseTimes: {},
    errorRates: {},
  });

  useEffect(() => {
    // Monitor API calls and collect metrics
    const originalFetch = window.fetch;
    const responseTimes: Record<string, number[]> = {};
    const errorCounts: Record<string, number> = {};
    const successCounts: Record<string, number> = {};

    window.fetch = async (...args) => {
      const url = typeof args[0] === 'string' ? args[0] : args[0].url;
      const startTime = performance.now();

      try {
        const response = await originalFetch(...args);
        const endTime = performance.now();
        const responseTime = endTime - startTime;

        // Track response times
        if (!responseTimes[url]) {
          responseTimes[url] = [];
        }
        responseTimes[url].push(responseTime);

        // Track success counts
        successCounts[url] = (successCounts[url] || 0) + 1;

        return response;
      } catch (error) {
        // Track error counts
        errorCounts[url] = (errorCounts[url] || 0) + 1;
        throw error;
      }
    };

    // Update metrics periodically
    const interval = setInterval(() => {
      const newMetrics = {
        queryCounts: { ...successCounts },
        averageResponseTimes: {} as Record<string, number>,
        errorRates: {} as Record<string, number>,
      };

      // Calculate average response times
      Object.entries(responseTimes).forEach(([url, times]) => {
        const average = times.reduce((sum, time) => sum + time, 0) / times.length;
        newMetrics.averageResponseTimes[url] = average;
      });

      // Calculate error rates
      Object.keys({ ...successCounts, ...errorCounts }).forEach(url => {
        const errors = errorCounts[url] || 0;
        const successes = successCounts[url] || 0;
        const total = errors + successes;
        newMetrics.errorRates[url] = total > 0 ? (errors / total) * 100 : 0;
      });

      setMetrics(newMetrics);
    }, 5000);

    return () => {
      clearInterval(interval);
      window.fetch = originalFetch;
    };
  }, []);

  return metrics;
}
```

## Interview-Ready Summary

**Advanced Redux patterns include:**

1. **Entity Adapters** - Normalized state management with automatic CRUD operations and optimized selectors
2. **Advanced Async Thunks** - Error handling, retry logic, optimistic updates, and batch operations  
3. **RTK Query** - Declarative data fetching with caching, invalidation, optimistic updates, and real-time sync
4. **Performance Optimization** - Memoized selectors, entity normalization, background sync, and query deduplication

**Key advanced patterns:**
- **Optimistic Updates** - Immediate UI feedback with automatic rollback on errors
- **Real-time Synchronization** - WebSocket integration with cache updates
- **Infinite Scrolling** - Cursor-based pagination with cache merging
- **Background Sync** - Offline support with automatic retry mechanisms

**RTK Query features:** Automatic cache management, query deduplication, background refetching, optimistic updates, tag-based invalidation, real-time subscriptions.

**Best practices:** Use entity adapters for normalized data, implement optimistic updates for better UX, leverage RTK Query for server state, monitor performance metrics, handle offline scenarios, use TypeScript for type safety.