# Modern State Management with Zustand, Valtio, and Jotai

Modern React state management has evolved beyond Redux to include lightweight, atomic, and reactive solutions. This guide covers Zustand, Valtio, Jotai, and other modern state management libraries with practical implementation patterns for complex applications.

## Zustand - Lightweight Global State

### 1. Advanced Zustand Patterns
```typescript
// Advanced Zustand store with TypeScript and middleware
import { create } from 'zustand';
import { subscribeWithSelector, devtools, persist, createJSONStorage } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

// Types for our application state
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'moderator';
  preferences: {
    theme: 'light' | 'dark';
    language: string;
    notifications: boolean;
  };
}

interface AppState {
  // User management
  users: Record<string, User>;
  currentUserId: string | null;
  
  // UI state
  isLoading: Record<string, boolean>;
  errors: Record<string, string | null>;
  modals: {
    userEdit: { isOpen: boolean; userId?: string };
    confirmation: { isOpen: boolean; message?: string; onConfirm?: () => void };
  };
  
  // Filters and pagination
  filters: {
    role?: User['role'];
    search?: string;
    isActive?: boolean;
  };
  pagination: {
    page: number;
    pageSize: number;
    total: number;
  };
  
  // Cache management
  cache: {
    lastFetch: Record<string, number>;
    expiryTime: number;
  };
}

interface AppActions {
  // User actions
  setUsers: (users: User[]) => void;
  addUser: (user: User) => void;
  updateUser: (id: string, updates: Partial<User>) => void;
  removeUser: (id: string) => void;
  setCurrentUser: (userId: string | null) => void;
  getCurrentUser: () => User | null;
  
  // Async actions
  fetchUsers: () => Promise<void>;
  createUser: (userData: Omit<User, 'id'>) => Promise<User>;
  updateUserAsync: (id: string, updates: Partial<User>) => Promise<void>;
  deleteUser: (id: string) => Promise<void>;
  
  // UI actions
  setLoading: (key: string, loading: boolean) => void;
  setError: (key: string, error: string | null) => void;
  clearErrors: () => void;
  
  // Modal actions
  openUserEditModal: (userId?: string) => void;
  closeUserEditModal: () => void;
  openConfirmationModal: (message: string, onConfirm: () => void) => void;
  closeConfirmationModal: () => void;
  
  // Filter and pagination actions
  setFilters: (filters: Partial<AppState['filters']>) => void;
  clearFilters: () => void;
  setPagination: (pagination: Partial<AppState['pagination']>) => void;
  
  // Cache actions
  invalidateCache: (key?: string) => void;
  isCacheValid: (key: string) => boolean;
  
  // Utility actions
  reset: () => void;
  hydrate: (state: Partial<AppState>) => void;
}

type AppStore = AppState & AppActions;

// Initial state
const initialState: AppState = {
  users: {},
  currentUserId: null,
  isLoading: {},
  errors: {},
  modals: {
    userEdit: { isOpen: false },
    confirmation: { isOpen: false },
  },
  filters: {},
  pagination: {
    page: 1,
    pageSize: 20,
    total: 0,
  },
  cache: {
    lastFetch: {},
    expiryTime: 5 * 60 * 1000, // 5 minutes
  },
};

// Create store with middleware
export const useAppStore = create<AppStore>()(
  devtools(
    persist(
      subscribeWithSelector(
        immer((set, get) => ({
          ...initialState,
          
          // User actions
          setUsers: (users) => set((state) => {
            state.users = users.reduce((acc, user) => {
              acc[user.id] = user;
              return acc;
            }, {} as Record<string, User>);
            state.cache.lastFetch.users = Date.now();
          }),
          
          addUser: (user) => set((state) => {
            state.users[user.id] = user;
          }),
          
          updateUser: (id, updates) => set((state) => {
            if (state.users[id]) {
              Object.assign(state.users[id], updates);
            }
          }),
          
          removeUser: (id) => set((state) => {
            delete state.users[id];
            if (state.currentUserId === id) {
              state.currentUserId = null;
            }
          }),
          
          setCurrentUser: (userId) => set((state) => {
            state.currentUserId = userId;
          }),
          
          getCurrentUser: () => {
            const state = get();
            return state.currentUserId ? state.users[state.currentUserId] || null : null;
          },
          
          // Async actions with optimistic updates
          fetchUsers: async () => {
            const state = get();
            
            // Check cache validity
            if (state.isCacheValid('users')) {
              return;
            }
            
            set((state) => {
              state.isLoading.users = true;
              state.errors.users = null;
            });
            
            try {
              const response = await fetch('/api/users');
              if (!response.ok) {
                throw new Error('Failed to fetch users');
              }
              
              const users = await response.json();
              state.setUsers(users);
            } catch (error) {
              set((state) => {
                state.errors.users = error instanceof Error ? error.message : 'Unknown error';
              });
            } finally {
              set((state) => {
                state.isLoading.users = false;
              });
            }
          },
          
          createUser: async (userData) => {
            const tempId = `temp_${Date.now()}`;
            const optimisticUser: User = {
              ...userData,
              id: tempId,
            };
            
            // Optimistic update
            set((state) => {
              state.users[tempId] = optimisticUser;
              state.isLoading.createUser = true;
              state.errors.createUser = null;
            });
            
            try {
              const response = await fetch('/api/users', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(userData),
              });
              
              if (!response.ok) {
                throw new Error('Failed to create user');
              }
              
              const newUser = await response.json();
              
              set((state) => {
                delete state.users[tempId];
                state.users[newUser.id] = newUser;
              });
              
              return newUser;
            } catch (error) {
              // Revert optimistic update
              set((state) => {
                delete state.users[tempId];
                state.errors.createUser = error instanceof Error ? error.message : 'Unknown error';
              });
              throw error;
            } finally {
              set((state) => {
                state.isLoading.createUser = false;
              });
            }
          },
          
          updateUserAsync: async (id, updates) => {
            const originalUser = get().users[id];
            if (!originalUser) return;
            
            // Optimistic update
            set((state) => {
              Object.assign(state.users[id], updates);
              state.isLoading.updateUser = true;
              state.errors.updateUser = null;
            });
            
            try {
              const response = await fetch(`/api/users/${id}`, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(updates),
              });
              
              if (!response.ok) {
                throw new Error('Failed to update user');
              }
              
              const updatedUser = await response.json();
              
              set((state) => {
                state.users[id] = updatedUser;
              });
            } catch (error) {
              // Revert optimistic update
              set((state) => {
                state.users[id] = originalUser;
                state.errors.updateUser = error instanceof Error ? error.message : 'Unknown error';
              });
              throw error;
            } finally {
              set((state) => {
                state.isLoading.updateUser = false;
              });
            }
          },
          
          deleteUser: async (id) => {
            const originalUser = get().users[id];
            if (!originalUser) return;
            
            // Optimistic update
            set((state) => {
              delete state.users[id];
              state.isLoading.deleteUser = true;
              state.errors.deleteUser = null;
            });
            
            try {
              const response = await fetch(`/api/users/${id}`, {
                method: 'DELETE',
              });
              
              if (!response.ok) {
                throw new Error('Failed to delete user');
              }
            } catch (error) {
              // Revert optimistic update
              set((state) => {
                state.users[id] = originalUser;
                state.errors.deleteUser = error instanceof Error ? error.message : 'Unknown error';
              });
              throw error;
            } finally {
              set((state) => {
                state.isLoading.deleteUser = false;
              });
            }
          },
          
          // UI actions
          setLoading: (key, loading) => set((state) => {
            state.isLoading[key] = loading;
          }),
          
          setError: (key, error) => set((state) => {
            state.errors[key] = error;
          }),
          
          clearErrors: () => set((state) => {
            Object.keys(state.errors).forEach(key => {
              state.errors[key] = null;
            });
          }),
          
          // Modal actions
          openUserEditModal: (userId) => set((state) => {
            state.modals.userEdit = { isOpen: true, userId };
          }),
          
          closeUserEditModal: () => set((state) => {
            state.modals.userEdit = { isOpen: false };
          }),
          
          openConfirmationModal: (message, onConfirm) => set((state) => {
            state.modals.confirmation = { isOpen: true, message, onConfirm };
          }),
          
          closeConfirmationModal: () => set((state) => {
            state.modals.confirmation = { isOpen: false };
          }),
          
          // Filter and pagination actions
          setFilters: (filters) => set((state) => {
            Object.assign(state.filters, filters);
            state.pagination.page = 1; // Reset pagination when filters change
          }),
          
          clearFilters: () => set((state) => {
            state.filters = {};
            state.pagination.page = 1;
          }),
          
          setPagination: (pagination) => set((state) => {
            Object.assign(state.pagination, pagination);
          }),
          
          // Cache actions
          invalidateCache: (key) => set((state) => {
            if (key) {
              delete state.cache.lastFetch[key];
            } else {
              state.cache.lastFetch = {};
            }
          }),
          
          isCacheValid: (key) => {
            const state = get();
            const lastFetch = state.cache.lastFetch[key];
            return lastFetch ? (Date.now() - lastFetch) < state.cache.expiryTime : false;
          },
          
          // Utility actions
          reset: () => set(initialState),
          
          hydrate: (partialState) => set((state) => {
            Object.assign(state, partialState);
          }),
        }))
      ),
      {
        name: 'app-store',
        storage: createJSONStorage(() => localStorage),
        partialize: (state) => ({
          // Only persist certain parts of the state
          currentUserId: state.currentUserId,
          filters: state.filters,
          pagination: state.pagination,
        }),
      }
    ),
    { name: 'app-store' }
  )
);

// Selectors with computed values
export const useAppSelectors = () => {
  const users = useAppStore((state) => Object.values(state.users));
  const currentUser = useAppStore((state) => state.getCurrentUser());
  const filters = useAppStore((state) => state.filters);
  const pagination = useAppStore((state) => state.pagination);
  
  // Computed selectors
  const filteredUsers = useMemo(() => {
    return users.filter(user => {
      if (filters.role && user.role !== filters.role) return false;
      if (filters.search) {
        const searchLower = filters.search.toLowerCase();
        const matchesSearch = 
          user.name.toLowerCase().includes(searchLower) ||
          user.email.toLowerCase().includes(searchLower);
        if (!matchesSearch) return false;
      }
      return true;
    });
  }, [users, filters]);
  
  const paginatedUsers = useMemo(() => {
    const startIndex = (pagination.page - 1) * pagination.pageSize;
    const endIndex = startIndex + pagination.pageSize;
    return filteredUsers.slice(startIndex, endIndex);
  }, [filteredUsers, pagination]);
  
  const userStats = useMemo(() => ({
    total: users.length,
    filtered: filteredUsers.length,
    byRole: users.reduce((acc, user) => {
      acc[user.role] = (acc[user.role] || 0) + 1;
      return acc;
    }, {} as Record<User['role'], number>),
  }), [users, filteredUsers]);
  
  return {
    users,
    currentUser,
    filteredUsers,
    paginatedUsers,
    userStats,
  };
};

// Custom hooks for specific functionality
export const useUserActions = () => {
  const actions = useAppStore((state) => ({
    fetchUsers: state.fetchUsers,
    createUser: state.createUser,
    updateUserAsync: state.updateUserAsync,
    deleteUser: state.deleteUser,
    setCurrentUser: state.setCurrentUser,
  }));
  
  return actions;
};

export const useUIState = () => {
  const ui = useAppStore((state) => ({
    isLoading: state.isLoading,
    errors: state.errors,
    modals: state.modals,
    setLoading: state.setLoading,
    setError: state.setError,
    clearErrors: state.clearErrors,
    openUserEditModal: state.openUserEditModal,
    closeUserEditModal: state.closeUserEditModal,
    openConfirmationModal: state.openConfirmationModal,
    closeConfirmationModal: state.closeConfirmationModal,
  }));
  
  return ui;
};

// Subscribe to specific state changes
export const useStoreSubscription = () => {
  useEffect(() => {
    const unsubscribe = useAppStore.subscribe(
      (state) => state.users,
      (users, prevUsers) => {
        console.log('Users changed:', { prev: Object.keys(prevUsers).length, current: Object.keys(users).length });
      }
    );
    
    return unsubscribe;
  }, []);
};
```

## Valtio - Proxy-based Reactive State

### 1. Valtio Implementation with Complex State
```typescript
// Valtio proxy-based state management
import { proxy, useSnapshot, subscribe, ref, unstable_buildProxyFunction } from 'valtio';
import { subscribeKey, watch } from 'valtio/utils';

// Create proxy state
interface Todo {
  id: string;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  dueDate?: Date;
  tags: string[];
  assignee?: User;
}

interface AppState {
  todos: Todo[];
  users: User[];
  filters: {
    status: 'all' | 'active' | 'completed';
    priority?: Todo['priority'];
    assignee?: string;
    tags: string[];
    search: string;
  };
  ui: {
    selectedTodos: Set<string>;
    draggedTodo: Todo | null;
    isLoading: boolean;
    errors: Record<string, string>;
    modal: {
      isOpen: boolean;
      type: 'create' | 'edit' | 'delete' | null;
      todoId?: string;
    };
  };
  settings: {
    theme: 'light' | 'dark';
    sortBy: 'dueDate' | 'priority' | 'title' | 'created';
    sortOrder: 'asc' | 'desc';
    autoSave: boolean;
  };
}

// Create proxy state with ref for non-reactive data
export const state = proxy<AppState>({
  todos: [],
  users: [],
  filters: {
    status: 'all',
    tags: [],
    search: '',
  },
  ui: {
    selectedTodos: new Set(),
    draggedTodo: null,
    isLoading: false,
    errors: {},
    modal: {
      isOpen: false,
      type: null,
    },
  },
  settings: {
    theme: 'light',
    sortBy: 'dueDate',
    sortOrder: 'asc',
    autoSave: true,
  },
});

// Actions using proxy mutations
export const actions = {
  // Todo actions
  addTodo: (todo: Omit<Todo, 'id'>) => {
    const newTodo: Todo = {
      ...todo,
      id: `todo_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
    };
    state.todos.push(newTodo);
  },

  updateTodo: (id: string, updates: Partial<Todo>) => {
    const index = state.todos.findIndex(todo => todo.id === id);
    if (index !== -1) {
      Object.assign(state.todos[index], updates);
    }
  },

  deleteTodo: (id: string) => {
    const index = state.todos.findIndex(todo => todo.id === id);
    if (index !== -1) {
      state.todos.splice(index, 1);
      state.ui.selectedTodos.delete(id);
    }
  },

  toggleTodo: (id: string) => {
    const todo = state.todos.find(t => t.id === id);
    if (todo) {
      todo.completed = !todo.completed;
    }
  },

  // Batch operations
  batchUpdate: (updates: Array<{ id: string; changes: Partial<Todo> }>) => {
    updates.forEach(({ id, changes }) => {
      const todo = state.todos.find(t => t.id === id);
      if (todo) {
        Object.assign(todo, changes);
      }
    });
  },

  batchDelete: (ids: string[]) => {
    state.todos = state.todos.filter(todo => !ids.includes(todo.id));
    ids.forEach(id => state.ui.selectedTodos.delete(id));
  },

  // Filter actions
  setFilter: (key: keyof AppState['filters'], value: any) => {
    (state.filters as any)[key] = value;
  },

  clearFilters: () => {
    state.filters = {
      status: 'all',
      tags: [],
      search: '',
    };
  },

  // Selection actions
  selectTodo: (id: string) => {
    state.ui.selectedTodos.add(id);
  },

  deselectTodo: (id: string) => {
    state.ui.selectedTodos.delete(id);
  },

  selectAll: () => {
    state.todos.forEach(todo => {
      state.ui.selectedTodos.add(todo.id);
    });
  },

  deselectAll: () => {
    state.ui.selectedTodos.clear();
  },

  // Drag and drop
  setDraggedTodo: (todo: Todo | null) => {
    state.ui.draggedTodo = ref(todo); // Use ref to prevent proxy wrapping
  },

  // Modal actions
  openModal: (type: 'create' | 'edit' | 'delete', todoId?: string) => {
    state.ui.modal = {
      isOpen: true,
      type,
      todoId,
    };
  },

  closeModal: () => {
    state.ui.modal = {
      isOpen: false,
      type: null,
    };
  },

  // Settings actions
  updateSettings: (updates: Partial<AppState['settings']>) => {
    Object.assign(state.settings, updates);
  },

  // Async actions with loading states
  loadTodos: async () => {
    state.ui.isLoading = true;
    state.ui.errors.loadTodos = '';

    try {
      const response = await fetch('/api/todos');
      if (!response.ok) {
        throw new Error('Failed to load todos');
      }
      
      const todos = await response.json();
      state.todos = todos;
    } catch (error) {
      state.ui.errors.loadTodos = error instanceof Error ? error.message : 'Unknown error';
    } finally {
      state.ui.isLoading = false;
    }
  },

  saveTodo: async (todo: Todo) => {
    const isNew = !state.todos.find(t => t.id === todo.id);
    
    try {
      const response = await fetch(`/api/todos${isNew ? '' : `/${todo.id}`}`, {
        method: isNew ? 'POST' : 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(todo),
      });

      if (!response.ok) {
        throw new Error('Failed to save todo');
      }

      const savedTodo = await response.json();
      
      if (isNew) {
        state.todos.push(savedTodo);
      } else {
        const index = state.todos.findIndex(t => t.id === todo.id);
        if (index !== -1) {
          state.todos[index] = savedTodo;
        }
      }
    } catch (error) {
      state.ui.errors.saveTodo = error instanceof Error ? error.message : 'Unknown error';
      throw error;
    }
  },
};

// Computed values using unstable_buildProxyFunction
export const computed = unstable_buildProxyFunction()({
  filteredTodos: (get) => {
    const todos = get(state.todos);
    const filters = get(state.filters);
    
    return todos.filter(todo => {
      if (filters.status === 'active' && todo.completed) return false;
      if (filters.status === 'completed' && !todo.completed) return false;
      if (filters.priority && todo.priority !== filters.priority) return false;
      if (filters.assignee && todo.assignee?.id !== filters.assignee) return false;
      if (filters.tags.length > 0 && !filters.tags.some(tag => todo.tags.includes(tag))) return false;
      if (filters.search) {
        const searchLower = filters.search.toLowerCase();
        if (!todo.title.toLowerCase().includes(searchLower)) return false;
      }
      return true;
    });
  },

  sortedTodos: (get) => {
    const filteredTodos = get(computed.filteredTodos);
    const settings = get(state.settings);
    
    return [...filteredTodos].sort((a, b) => {
      let comparison = 0;
      
      switch (settings.sortBy) {
        case 'title':
          comparison = a.title.localeCompare(b.title);
          break;
        case 'priority':
          const priorityOrder = { low: 0, medium: 1, high: 2 };
          comparison = priorityOrder[a.priority] - priorityOrder[b.priority];
          break;
        case 'dueDate':
          const aDate = a.dueDate ? new Date(a.dueDate).getTime() : Infinity;
          const bDate = b.dueDate ? new Date(b.dueDate).getTime() : Infinity;
          comparison = aDate - bDate;
          break;
      }
      
      return settings.sortOrder === 'desc' ? -comparison : comparison;
    });
  },

  selectedTodosData: (get) => {
    const selectedIds = get(state.ui.selectedTodos);
    const todos = get(state.todos);
    
    return todos.filter(todo => selectedIds.has(todo.id));
  },

  todoStats: (get) => {
    const todos = get(state.todos);
    
    return {
      total: todos.length,
      completed: todos.filter(t => t.completed).length,
      active: todos.filter(t => !t.completed).length,
      overdue: todos.filter(t => 
        t.dueDate && new Date(t.dueDate) < new Date() && !t.completed
      ).length,
      byPriority: todos.reduce((acc, todo) => {
        acc[todo.priority] = (acc[todo.priority] || 0) + 1;
        return acc;
      }, {} as Record<Todo['priority'], number>),
    };
  },
});

// React hooks for Valtio
export const useTodos = () => {
  const snap = useSnapshot(state);
  return {
    todos: snap.todos,
    filteredTodos: useSnapshot(computed.filteredTodos),
    sortedTodos: useSnapshot(computed.sortedTodos),
    selectedTodos: useSnapshot(computed.selectedTodosData),
    stats: useSnapshot(computed.todoStats),
    filters: snap.filters,
    ui: snap.ui,
    settings: snap.settings,
  };
};

export const useTodoActions = () => actions;

// Persistence with Valtio
export const setupPersistence = () => {
  // Save to localStorage on state changes
  const unsubscribe = subscribe(state, () => {
    const persistedState = {
      todos: state.todos,
      filters: state.filters,
      settings: state.settings,
    };
    
    localStorage.setItem('valtio-state', JSON.stringify(persistedState));
  });

  // Load from localStorage on init
  const savedState = localStorage.getItem('valtio-state');
  if (savedState) {
    try {
      const parsed = JSON.parse(savedState);
      Object.assign(state, parsed);
    } catch (error) {
      console.error('Failed to load persisted state:', error);
    }
  }

  return unsubscribe;
};

// Watchers for side effects
export const setupWatchers = () => {
  // Watch for filter changes and log
  const unsubscribeFilter = subscribeKey(state.filters, 'search', (search) => {
    console.log('Search filter changed:', search);
  });

  // Watch for todo completion and show notification
  const unsubscribeTodos = watch((get) => {
    return get(state.todos).filter(todo => todo.completed).length;
  }, (completedCount) => {
    if (completedCount > 0) {
      console.log(`${completedCount} todos completed!`);
    }
  });

  // Auto-save when settings.autoSave is enabled
  const unsubscribeAutoSave = subscribe(state.todos, async () => {
    if (state.settings.autoSave) {
      // Debounced auto-save logic here
      console.log('Auto-saving todos...');
    }
  });

  return () => {
    unsubscribeFilter();
    unsubscribeTodos();
    unsubscribeAutoSave();
  };
};
```

## Jotai - Atomic State Management

### 1. Jotai Atoms and Advanced Patterns
```typescript
// Jotai atomic state management
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';
import { atomWithStorage, atomWithReducer, atomWithReset, RESET } from 'jotai/utils';
import { atomFamily, loadable, atomWithQuery } from 'jotai/utils';

// Basic atoms
export const userAtom = atom<User | null>(null);
export const usersAtom = atomWithStorage<User[]>('users', []);
export const isLoadingAtom = atom(false);
export const errorAtom = atomWithReset<string | null>(null);

// Theme atom with persistence
export const themeAtom = atomWithStorage<'light' | 'dark'>('theme', 'light');

// Filter atoms
export const searchAtom = atom('');
export const roleFilterAtom = atom<User['role'] | 'all'>('all');
export const activeFilterAtom = atom<boolean | null>(null);

// Computed atoms (derived state)
export const filteredUsersAtom = atom((get) => {
  const users = get(usersAtom);
  const search = get(searchAtom);
  const roleFilter = get(roleFilterAtom);
  const activeFilter = get(activeFilterAtom);

  return users.filter(user => {
    if (search && !user.name.toLowerCase().includes(search.toLowerCase())) {
      return false;
    }
    if (roleFilter !== 'all' && user.role !== roleFilter) {
      return false;
    }
    if (activeFilter !== null) {
      const isActive = new Date(user.lastActive) > new Date(Date.now() - 24 * 60 * 60 * 1000);
      if (isActive !== activeFilter) {
        return false;
      }
    }
    return true;
  });
});

export const userStatsAtom = atom((get) => {
  const users = get(usersAtom);
  const filteredUsers = get(filteredUsersAtom);

  return {
    total: users.length,
    filtered: filteredUsers.length,
    byRole: users.reduce((acc, user) => {
      acc[user.role] = (acc[user.role] || 0) + 1;
      return acc;
    }, {} as Record<User['role'], number>),
  };
});

// Async atoms with loading states
export const fetchUsersAtom = atom(
  null,
  async (get, set) => {
    set(isLoadingAtom, true);
    set(errorAtom, RESET);

    try {
      const response = await fetch('/api/users');
      if (!response.ok) {
        throw new Error('Failed to fetch users');
      }
      
      const users = await response.json();
      set(usersAtom, users);
      return users;
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      set(errorAtom, errorMessage);
      throw error;
    } finally {
      set(isLoadingAtom, false);
    }
  }
);

// Atom family for individual user management
export const userAtomFamily = atomFamily((id: string) =>
  atom<User | null>(null)
);

export const updateUserAtom = atom(
  null,
  async (get, set, { id, updates }: { id: string; updates: Partial<User> }) => {
    const users = get(usersAtom);
    const userIndex = users.findIndex(u => u.id === id);
    
    if (userIndex === -1) {
      throw new Error('User not found');
    }

    // Optimistic update
    const optimisticUsers = [...users];
    optimisticUsers[userIndex] = { ...optimisticUsers[userIndex], ...updates };
    set(usersAtom, optimisticUsers);

    try {
      const response = await fetch(`/api/users/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates),
      });

      if (!response.ok) {
        throw new Error('Failed to update user');
      }

      const updatedUser = await response.json();
      
      // Update with server response
      const finalUsers = [...get(usersAtom)];
      const finalIndex = finalUsers.findIndex(u => u.id === id);
      if (finalIndex !== -1) {
        finalUsers[finalIndex] = updatedUser;
        set(usersAtom, finalUsers);
      }

      return updatedUser;
    } catch (error) {
      // Revert optimistic update
      set(usersAtom, users);
      throw error;
    }
  }
);

// Reducer-based atom for complex state management
interface TodoState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  editingId: string | null;
}

type TodoAction =
  | { type: 'ADD_TODO'; payload: Omit<Todo, 'id'> }
  | { type: 'UPDATE_TODO'; payload: { id: string; updates: Partial<Todo> } }
  | { type: 'DELETE_TODO'; payload: string }
  | { type: 'TOGGLE_TODO'; payload: string }
  | { type: 'SET_FILTER'; payload: TodoState['filter'] }
  | { type: 'SET_EDITING'; payload: string | null }
  | { type: 'CLEAR_COMPLETED' };

const todoReducer = (state: TodoState, action: TodoAction): TodoState => {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [
          ...state.todos,
          {
            ...action.payload,
            id: `todo_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
          },
        ],
      };

    case 'UPDATE_TODO':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, ...action.payload.updates }
            : todo
        ),
      };

    case 'DELETE_TODO':
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload),
        editingId: state.editingId === action.payload ? null : state.editingId,
      };

    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload
            ? { ...todo, completed: !todo.completed }
            : todo
        ),
      };

    case 'SET_FILTER':
      return {
        ...state,
        filter: action.payload,
      };

    case 'SET_EDITING':
      return {
        ...state,
        editingId: action.payload,
      };

    case 'CLEAR_COMPLETED':
      return {
        ...state,
        todos: state.todos.filter(todo => !todo.completed),
      };

    default:
      return state;
  }
};

export const todoStateAtom = atomWithReducer(
  {
    todos: [],
    filter: 'all' as const,
    editingId: null,
  },
  todoReducer
);

// Derived atoms from reducer state
export const todosAtom = atom((get) => get(todoStateAtom).todos);
export const todoFilterAtom = atom((get) => get(todoStateAtom).filter);
export const editingIdAtom = atom((get) => get(todoStateAtom).editingId);

export const filteredTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  const filter = get(todoFilterAtom);

  switch (filter) {
    case 'active':
      return todos.filter(todo => !todo.completed);
    case 'completed':
      return todos.filter(todo => todo.completed);
    default:
      return todos;
  }
});

export const todoStatsAtom = atom((get) => {
  const todos = get(todosAtom);
  
  return {
    total: todos.length,
    active: todos.filter(todo => !todo.completed).length,
    completed: todos.filter(todo => todo.completed).length,
  };
});

// Query atom for server state
export const userQueryAtom = atomWithQuery((get) => ({
  queryKey: ['users', get(searchAtom), get(roleFilterAtom)],
  queryFn: async ({ queryKey }) => {
    const [_key, search, roleFilter] = queryKey;
    const params = new URLSearchParams();
    
    if (search) params.set('search', search);
    if (roleFilter !== 'all') params.set('role', roleFilter);
    
    const response = await fetch(`/api/users?${params}`);
    if (!response.ok) {
      throw new Error('Failed to fetch users');
    }
    
    return response.json();
  },
  staleTime: 60000, // 1 minute
  cacheTime: 300000, // 5 minutes
}));

// Loadable atom for handling async state
export const loadableUsersAtom = loadable(userQueryAtom);

// Custom hooks for Jotai
export const useUsers = () => {
  const users = useAtomValue(usersAtom);
  const filteredUsers = useAtomValue(filteredUsersAtom);
  const stats = useAtomValue(userStatsAtom);
  const isLoading = useAtomValue(isLoadingAtom);
  const error = useAtomValue(errorAtom);

  return {
    users,
    filteredUsers,
    stats,
    isLoading,
    error,
  };
};

export const useUserActions = () => {
  const fetchUsers = useSetAtom(fetchUsersAtom);
  const updateUser = useSetAtom(updateUserAtom);
  const setError = useSetAtom(errorAtom);

  return {
    fetchUsers,
    updateUser,
    setError,
    clearError: () => setError(RESET),
  };
};

export const useFilters = () => {
  const [search, setSearch] = useAtom(searchAtom);
  const [roleFilter, setRoleFilter] = useAtom(roleFilterAtom);
  const [activeFilter, setActiveFilter] = useAtom(activeFilterAtom);

  const clearFilters = () => {
    setSearch('');
    setRoleFilter('all');
    setActiveFilter(null);
  };

  return {
    search,
    setSearch,
    roleFilter,
    setRoleFilter,
    activeFilter,
    setActiveFilter,
    clearFilters,
  };
};

export const useTodos = () => {
  const [todoState, dispatch] = useAtom(todoStateAtom);
  const filteredTodos = useAtomValue(filteredTodosAtom);
  const stats = useAtomValue(todoStatsAtom);

  const actions = {
    addTodo: (todo: Omit<Todo, 'id'>) => 
      dispatch({ type: 'ADD_TODO', payload: todo }),
    
    updateTodo: (id: string, updates: Partial<Todo>) =>
      dispatch({ type: 'UPDATE_TODO', payload: { id, updates } }),
    
    deleteTodo: (id: string) =>
      dispatch({ type: 'DELETE_TODO', payload: id }),
    
    toggleTodo: (id: string) =>
      dispatch({ type: 'TOGGLE_TODO', payload: id }),
    
    setFilter: (filter: TodoState['filter']) =>
      dispatch({ type: 'SET_FILTER', payload: filter }),
    
    setEditing: (id: string | null) =>
      dispatch({ type: 'SET_EDITING', payload: id }),
    
    clearCompleted: () =>
      dispatch({ type: 'CLEAR_COMPLETED' }),
  };

  return {
    ...todoState,
    filteredTodos,
    stats,
    actions,
  };
};

// Async query hook with loadable
export const useAsyncUsers = () => {
  const loadableUsers = useAtomValue(loadableUsersAtom);

  switch (loadableUsers.state) {
    case 'hasError':
      return {
        data: null,
        isLoading: false,
        error: loadableUsers.error,
      };
    
    case 'loading':
      return {
        data: null,
        isLoading: true,
        error: null,
      };
    
    case 'hasData':
      return {
        data: loadableUsers.data,
        isLoading: false,
        error: null,
      };
    
    default:
      return {
        data: null,
        isLoading: false,
        error: null,
      };
  }
};
```

## Server State Management Integration

### 1. Combining Client and Server State
```typescript
// Integration patterns for client and server state
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { create } from 'zustand';
import { atom, useAtom } from 'jotai';

// Zustand for client state, React Query for server state
interface ClientState {
  ui: {
    selectedItems: Set<string>;
    filters: Record<string, any>;
    modals: Record<string, boolean>;
  };
  cache: {
    lastSync: number;
    preferences: Record<string, any>;
  };
}

export const useClientStore = create<ClientState & {
  setSelected: (items: Set<string>) => void;
  setFilter: (key: string, value: any) => void;
  setModal: (key: string, isOpen: boolean) => void;
  setPreference: (key: string, value: any) => void;
}>((set) => ({
  ui: {
    selectedItems: new Set(),
    filters: {},
    modals: {},
  },
  cache: {
    lastSync: 0,
    preferences: {},
  },
  
  setSelected: (items) => set((state) => ({
    ui: { ...state.ui, selectedItems: items },
  })),
  
  setFilter: (key, value) => set((state) => ({
    ui: { ...state.ui, filters: { ...state.ui.filters, [key]: value } },
  })),
  
  setModal: (key, isOpen) => set((state) => ({
    ui: { ...state.ui, modals: { ...state.ui.modals, [key]: isOpen } },
  })),
  
  setPreference: (key, value) => set((state) => ({
    cache: { ...state.cache, preferences: { ...state.cache.preferences, [key]: value } },
  })),
}));

// Combined hook for managing both client and server state
export const useIntegratedState = () => {
  const queryClient = useQueryClient();
  const clientState = useClientStore();
  
  // Server state
  const {
    data: users,
    isLoading,
    error,
    refetch,
  } = useQuery({
    queryKey: ['users', clientState.ui.filters],
    queryFn: () => fetchUsers(clientState.ui.filters),
    staleTime: 60000,
  });

  const createUserMutation = useMutation({
    mutationFn: createUser,
    onMutate: async (newUser) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['users'] });
      
      // Snapshot previous value
      const previousUsers = queryClient.getQueryData(['users']);
      
      // Optimistically update
      queryClient.setQueryData(['users'], (old: User[]) => [
        ...old,
        { ...newUser, id: `temp_${Date.now()}` },
      ]);
      
      return { previousUsers };
    },
    onError: (err, newUser, context) => {
      // Rollback on error
      queryClient.setQueryData(['users'], context?.previousUsers);
    },
    onSettled: () => {
      // Always refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });

  // Sync client and server state
  useEffect(() => {
    if (users) {
      const currentTime = Date.now();
      clientState.cache.lastSync = currentTime;
    }
  }, [users, clientState]);

  return {
    // Server state
    users,
    isLoading,
    error,
    refetch,
    createUser: createUserMutation.mutate,
    isCreating: createUserMutation.isPending,
    
    // Client state
    selectedItems: clientState.ui.selectedItems,
    filters: clientState.ui.filters,
    modals: clientState.ui.modals,
    preferences: clientState.cache.preferences,
    
    // Combined actions
    setSelected: clientState.setSelected,
    setFilter: clientState.setFilter,
    setModal: clientState.setModal,
    setPreference: clientState.setPreference,
    
    // Utility methods
    clearCache: () => {
      queryClient.clear();
      clientState.cache.lastSync = 0;
    },
    
    syncState: async () => {
      await refetch();
      clientState.cache.lastSync = Date.now();
    },
  };
};

// Jotai integration with React Query
export const serverUsersAtom = atom(async () => {
  const response = await fetch('/api/users');
  if (!response.ok) {
    throw new Error('Failed to fetch users');
  }
  return response.json();
});

export const usersWithClientStateAtom = atom(
  (get) => get(serverUsersAtom),
  async (get, set, update: { type: 'optimistic_add' | 'optimistic_update'; payload: any }) => {
    const currentUsers = await get(serverUsersAtom);
    
    switch (update.type) {
      case 'optimistic_add':
        // Optimistic update for new user
        set(serverUsersAtom, [...currentUsers, update.payload]);
        
        try {
          const newUser = await createUser(update.payload);
          // Update with real data
          set(serverUsersAtom, [
            ...currentUsers,
            newUser,
          ]);
        } catch (error) {
          // Revert on error
          set(serverUsersAtom, currentUsers);
          throw error;
        }
        break;
        
      default:
        break;
    }
  }
);
```

## Interview-Ready Summary

**Modern state management libraries offer different paradigms:**

1. **Zustand** - Lightweight global state with hooks API, middleware support, and TypeScript integration
2. **Valtio** - Proxy-based reactive state with automatic change detection and computed values
3. **Jotai** - Atomic state management with granular updates and dependency tracking
4. **Integration Patterns** - Combining client state libraries with server state management (React Query/SWR)

**Key advantages by library:**
- **Zustand**: Minimal boilerplate, excellent TypeScript support, middleware ecosystem, easy migration from Redux
- **Valtio**: Mutable API, automatic reactivity, great for complex nested state, minimal re-renders
- **Jotai**: Atomic updates, fine-grained reactivity, excellent for large applications, bottom-up approach

**When to use each:**
- **Zustand**: Small to medium apps, need Redux-like patterns without complexity, want simple global state
- **Valtio**: Complex nested state, need mutable updates, want automatic reactivity without selectors
- **Jotai**: Large apps with many independent state pieces, need fine-grained updates, atomic design patterns

**Best practices:** Choose based on app complexity, team preferences, and performance requirements. Combine with React Query for server state. Use TypeScript for better DX. Implement proper error handling and optimistic updates.