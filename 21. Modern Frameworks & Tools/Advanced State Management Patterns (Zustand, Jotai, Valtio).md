# Advanced State Management Patterns (Zustand, Jotai, Valtio)

Modern state management has evolved beyond Redux to include more lightweight, flexible solutions. This comprehensive guide covers Zustand, Jotai, and Valtio - three powerful state management libraries that offer different paradigms for handling application state in React applications.

## Zustand: Simple and Scalable State Management

### Basic Zustand Store Creation

```typescript
// stores/useProductStore.ts
import { create } from 'zustand';
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

interface ProductState {
  products: Product[];
  loading: boolean;
  error: string | null;
  filters: {
    category: string;
    priceRange: [number, number];
    inStockOnly: boolean;
  };
  
  // Actions
  setProducts: (products: Product[]) => void;
  addProduct: (product: Product) => void;
  updateProduct: (id: string, updates: Partial<Product>) => void;
  removeProduct: (id: string) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
  setFilters: (filters: Partial<ProductState['filters']>) => void;
  clearFilters: () => void;
  
  // Computed values
  filteredProducts: () => Product[];
  totalValue: () => number;
  categoryCounts: () => Record<string, number>;
}

export const useProductStore = create<ProductState>()(
  devtools(
    persist(
      subscribeWithSelector(
        immer((set, get) => ({
          products: [],
          loading: false,
          error: null,
          filters: {
            category: '',
            priceRange: [0, 1000],
            inStockOnly: false,
          },

          setProducts: (products) => set((state) => {
            state.products = products;
            state.error = null;
          }),

          addProduct: (product) => set((state) => {
            state.products.push(product);
          }),

          updateProduct: (id, updates) => set((state) => {
            const index = state.products.findIndex(p => p.id === id);
            if (index !== -1) {
              Object.assign(state.products[index], updates);
            }
          }),

          removeProduct: (id) => set((state) => {
            state.products = state.products.filter(p => p.id !== id);
          }),

          setLoading: (loading) => set((state) => {
            state.loading = loading;
          }),

          setError: (error) => set((state) => {
            state.error = error;
          }),

          setFilters: (filters) => set((state) => {
            Object.assign(state.filters, filters);
          }),

          clearFilters: () => set((state) => {
            state.filters = {
              category: '',
              priceRange: [0, 1000],
              inStockOnly: false,
            };
          }),

          // Computed values
          filteredProducts: () => {
            const { products, filters } = get();
            return products.filter(product => {
              if (filters.category && product.category !== filters.category) {
                return false;
              }
              if (product.price < filters.priceRange[0] || product.price > filters.priceRange[1]) {
                return false;
              }
              if (filters.inStockOnly && !product.inStock) {
                return false;
              }
              return true;
            });
          },

          totalValue: () => {
            const { products } = get();
            return products.reduce((total, product) => total + product.price, 0);
          },

          categoryCounts: () => {
            const { products } = get();
            return products.reduce((counts, product) => {
              counts[product.category] = (counts[product.category] || 0) + 1;
              return counts;
            }, {} as Record<string, number>);
          },
        }))
      ),
      {
        name: 'product-store',
        partialize: (state) => ({
          products: state.products,
          filters: state.filters,
        }),
      }
    ),
    { name: 'ProductStore' }
  )
);

// Store slices for better organization
export const useProductActions = () => useProductStore(state => ({
  setProducts: state.setProducts,
  addProduct: state.addProduct,
  updateProduct: state.updateProduct,
  removeProduct: state.removeProduct,
  setLoading: state.setLoading,
  setError: state.setError,
}));

export const useProductData = () => useProductStore(state => ({
  products: state.products,
  loading: state.loading,
  error: state.error,
  filteredProducts: state.filteredProducts(),
  totalValue: state.totalValue(),
  categoryCounts: state.categoryCounts(),
}));

export const useProductFilters = () => useProductStore(state => ({
  filters: state.filters,
  setFilters: state.setFilters,
  clearFilters: state.clearFilters,
}));
```

### Advanced Zustand Patterns

```typescript
// stores/useAuthStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { createJSONStorage } from 'zustand/middleware';

interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'user';
  permissions: string[];
}

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  
  // Actions
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
  updateUser: (updates: Partial<User>) => void;
  hasPermission: (permission: string) => boolean;
}

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      (set, get) => ({
        user: null,
        token: null,
        isAuthenticated: false,
        isLoading: false,

        login: async (email: string, password: string) => {
          set({ isLoading: true });
          
          try {
            const response = await fetch('/api/auth/login', {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ email, password }),
            });

            if (!response.ok) {
              throw new Error('Login failed');
            }

            const { user, token } = await response.json();
            
            set({
              user,
              token,
              isAuthenticated: true,
              isLoading: false,
            });

            // Set up token refresh
            scheduleTokenRefresh(token);
            
          } catch (error) {
            set({ isLoading: false });
            throw error;
          }
        },

        logout: async () => {
          try {
            await fetch('/api/auth/logout', {
              method: 'POST',
              headers: {
                'Authorization': `Bearer ${get().token}`,
              },
            });
          } catch (error) {
            console.error('Logout error:', error);
          }
          
          set({
            user: null,
            token: null,
            isAuthenticated: false,
            isLoading: false,
          });

          clearTokenRefresh();
        },

        refreshToken: async () => {
          const { token } = get();
          if (!token) return;

          try {
            const response = await fetch('/api/auth/refresh', {
              method: 'POST',
              headers: {
                'Authorization': `Bearer ${token}`,
              },
            });

            if (!response.ok) {
              throw new Error('Token refresh failed');
            }

            const { token: newToken, user } = await response.json();
            
            set({
              token: newToken,
              user,
              isAuthenticated: true,
            });

            scheduleTokenRefresh(newToken);
            
          } catch (error) {
            console.error('Token refresh error:', error);
            get().logout();
          }
        },

        updateUser: (updates) => set((state) => ({
          user: state.user ? { ...state.user, ...updates } : null,
        })),

        hasPermission: (permission) => {
          const { user } = get();
          return user?.permissions.includes(permission) || false;
        },
      }),
      {
        name: 'auth-store',
        storage: createJSONStorage(() => localStorage),
        partialize: (state) => ({
          user: state.user,
          token: state.token,
          isAuthenticated: state.isAuthenticated,
        }),
      }
    ),
    { name: 'AuthStore' }
  )
);

// Token refresh scheduling
let refreshTimeout: NodeJS.Timeout | null = null;

function scheduleTokenRefresh(token: string) {
  clearTokenRefresh();
  
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const expirationTime = payload.exp * 1000;
    const refreshTime = expirationTime - Date.now() - 60000; // Refresh 1 minute before expiry
    
    if (refreshTime > 0) {
      refreshTimeout = setTimeout(() => {
        useAuthStore.getState().refreshToken();
      }, refreshTime);
    }
  } catch (error) {
    console.error('Token parsing error:', error);
  }
}

function clearTokenRefresh() {
  if (refreshTimeout) {
    clearTimeout(refreshTimeout);
    refreshTimeout = null;
  }
}

// Store subscription for side effects
useAuthStore.subscribe(
  (state) => state.isAuthenticated,
  (isAuthenticated) => {
    if (isAuthenticated) {
      console.log('User authenticated');
      // Trigger data fetching, analytics, etc.
    } else {
      console.log('User logged out');
      // Clear sensitive data, stop timers, etc.
    }
  }
);
```

## Jotai: Atomic State Management

### Basic Atoms and Derived State

```typescript
// atoms/productAtoms.ts
import { atom } from 'jotai';
import { atomWithStorage, atomWithReset, atomWithReducer } from 'jotai/utils';
import { focusAtom } from 'jotai-optics';

// Base atoms
export const productsAtom = atom<Product[]>([]);
export const loadingAtom = atom(false);
export const errorAtom = atom<string | null>(null);

// Storage atoms (persisted)
export const userPreferencesAtom = atomWithStorage('userPreferences', {
  theme: 'light',
  currency: 'USD',
  itemsPerPage: 10,
});

// Resettable atoms
export const filtersAtom = atomWithReset({
  category: '',
  priceRange: [0, 1000] as [number, number],
  search: '',
  sortBy: 'name' as 'name' | 'price' | 'rating',
  sortOrder: 'asc' as 'asc' | 'desc',
});

// Derived atoms (computed)
export const filteredProductsAtom = atom((get) => {
  const products = get(productsAtom);
  const filters = get(filtersAtom);
  
  let filtered = products.filter(product => {
    if (filters.category && product.category !== filters.category) {
      return false;
    }
    if (product.price < filters.priceRange[0] || product.price > filters.priceRange[1]) {
      return false;
    }
    if (filters.search && !product.name.toLowerCase().includes(filters.search.toLowerCase())) {
      return false;
    }
    return true;
  });

  // Apply sorting
  filtered.sort((a, b) => {
    let comparison = 0;
    switch (filters.sortBy) {
      case 'name':
        comparison = a.name.localeCompare(b.name);
        break;
      case 'price':
        comparison = a.price - b.price;
        break;
      case 'rating':
        comparison = (a.rating || 0) - (b.rating || 0);
        break;
    }
    return filters.sortOrder === 'desc' ? -comparison : comparison;
  });

  return filtered;
});

export const productCategoriesAtom = atom((get) => {
  const products = get(productsAtom);
  const categories = Array.from(new Set(products.map(p => p.category)));
  return categories.sort();
});

export const productStatsAtom = atom((get) => {
  const products = get(filteredProductsAtom);
  return {
    total: products.length,
    averagePrice: products.length > 0 
      ? products.reduce((sum, p) => sum + p.price, 0) / products.length 
      : 0,
    totalValue: products.reduce((sum, p) => sum + p.price, 0),
    byCategory: products.reduce((acc, product) => {
      acc[product.category] = (acc[product.category] || 0) + 1;
      return acc;
    }, {} as Record<string, number>),
  };
});

// Write-only atoms (actions)
export const addProductAtom = atom(
  null,
  (get, set, product: Product) => {
    const products = get(productsAtom);
    set(productsAtom, [...products, product]);
  }
);

export const updateProductAtom = atom(
  null,
  (get, set, { id, updates }: { id: string; updates: Partial<Product> }) => {
    const products = get(productsAtom);
    set(productsAtom, products.map(p => 
      p.id === id ? { ...p, ...updates } : p
    ));
  }
);

export const removeProductAtom = atom(
  null,
  (get, set, id: string) => {
    const products = get(productsAtom);
    set(productsAtom, products.filter(p => p.id !== id));
  }
);

// Async atoms
export const fetchProductsAtom = atom(
  null,
  async (get, set) => {
    set(loadingAtom, true);
    set(errorAtom, null);
    
    try {
      const response = await fetch('/api/products');
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      const products = await response.json();
      set(productsAtom, products);
    } catch (error) {
      set(errorAtom, error instanceof Error ? error.message : 'Unknown error');
    } finally {
      set(loadingAtom, false);
    }
  }
);

// Focused atoms using Optics
export const firstProductAtom = focusAtom(productsAtom, (optic) =>
  optic.at(0)
);

export const productByIdAtom = (id: string) =>
  focusAtom(productsAtom, (optic) =>
    optic.find((product) => product.id === id)
  );

// Reducer atom for complex state management
interface CartState {
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
}

type CartAction =
  | { type: 'ADD_ITEM'; productId: string; price: number }
  | { type: 'REMOVE_ITEM'; productId: string }
  | { type: 'UPDATE_QUANTITY'; productId: string; quantity: number }
  | { type: 'CLEAR_CART' };

const cartReducer = (state: CartState, action: CartAction): CartState => {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existingItem = state.items.find(item => item.productId === action.productId);
      const newItems = existingItem
        ? state.items.map(item =>
            item.productId === action.productId
              ? { ...item, quantity: item.quantity + 1 }
              : item
          )
        : [...state.items, { productId: action.productId, quantity: 1, price: action.price }];
      
      const total = newItems.reduce((sum, item) => sum + (item.price * item.quantity), 0);
      return { items: newItems, total };
    }
    
    case 'REMOVE_ITEM': {
      const newItems = state.items.filter(item => item.productId !== action.productId);
      const total = newItems.reduce((sum, item) => sum + (item.price * item.quantity), 0);
      return { items: newItems, total };
    }
    
    case 'UPDATE_QUANTITY': {
      const newItems = action.quantity === 0
        ? state.items.filter(item => item.productId !== action.productId)
        : state.items.map(item =>
            item.productId === action.productId
              ? { ...item, quantity: action.quantity }
              : item
          );
      
      const total = newItems.reduce((sum, item) => sum + (item.price * item.quantity), 0);
      return { items: newItems, total };
    }
    
    case 'CLEAR_CART':
      return { items: [], total: 0 };
    
    default:
      return state;
  }
};

export const cartAtom = atomWithReducer({ items: [], total: 0 }, cartReducer);
```

### Advanced Jotai Patterns

```typescript
// atoms/asyncAtoms.ts
import { atom } from 'jotai';
import { atomWithQuery, atomWithMutation } from 'jotai-tanstack-query';
import { unwrap } from 'jotai/utils';

// Query atoms with React Query integration
export const userQueryAtom = atomWithQuery(() => ({
  queryKey: ['user'],
  queryFn: async () => {
    const response = await fetch('/api/user');
    if (!response.ok) {
      throw new Error('Failed to fetch user');
    }
    return response.json();
  },
}));

export const postsQueryAtom = atomWithQuery((get) => ({
  queryKey: ['posts', get(userQueryAtom).data?.id],
  queryFn: async ({ queryKey }) => {
    const [, userId] = queryKey;
    if (!userId) return [];
    
    const response = await fetch(`/api/posts?userId=${userId}`);
    if (!response.ok) {
      throw new Error('Failed to fetch posts');
    }
    return response.json();
  },
  enabled: !!get(userQueryAtom).data?.id,
}));

// Mutation atoms
export const createPostMutationAtom = atomWithMutation(() => ({
  mutationFn: async (newPost: { title: string; content: string }) => {
    const response = await fetch('/api/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(newPost),
    });
    
    if (!response.ok) {
      throw new Error('Failed to create post');
    }
    
    return response.json();
  },
  onSuccess: () => {
    // Invalidate posts query
    queryClient.invalidateQueries({ queryKey: ['posts'] });
  },
}));

// Unwrapped atoms for easier consumption
export const userAtom = unwrap(userQueryAtom, (prev) => prev ?? null);
export const postsAtom = unwrap(postsQueryAtom, (prev) => prev ?? []);

// Family of atoms for dynamic atom creation
import { atomFamily } from 'jotai/utils';

export const postAtomFamily = atomFamily((postId: string) =>
  atomWithQuery(() => ({
    queryKey: ['post', postId],
    queryFn: async () => {
      const response = await fetch(`/api/posts/${postId}`);
      if (!response.ok) {
        throw new Error('Failed to fetch post');
      }
      return response.json();
    },
  }))
);

// Conditional atoms
export const conditionalDataAtom = atom(async (get) => {
  const user = get(userAtom);
  
  if (!user) {
    return null;
  }
  
  if (user.role === 'admin') {
    const response = await fetch('/api/admin/data');
    return response.json();
  } else {
    const response = await fetch('/api/user/data');
    return response.json();
  }
});

// Debounced atoms
import { atomWithDebounce } from './utils/atomWithDebounce';

export const searchQueryAtom = atom('');
export const debouncedSearchAtom = atomWithDebounce(searchQueryAtom, 300);

export const searchResultsAtom = atom(async (get) => {
  const query = get(debouncedSearchAtom);
  
  if (!query.trim()) {
    return [];
  }
  
  const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
  return response.json();
});

// Validation atoms
export const formDataAtom = atom({
  name: '',
  email: '',
  age: 0,
});

export const formValidationAtom = atom((get) => {
  const data = get(formDataAtom);
  const errors: Record<string, string> = {};
  
  if (!data.name.trim()) {
    errors.name = 'Name is required';
  }
  
  if (!data.email.trim()) {
    errors.email = 'Email is required';
  } else if (!/\S+@\S+\.\S+/.test(data.email)) {
    errors.email = 'Email is invalid';
  }
  
  if (data.age < 18) {
    errors.age = 'Must be 18 or older';
  }
  
  return {
    errors,
    isValid: Object.keys(errors).length === 0,
  };
});

// utils/atomWithDebounce.ts
import { atom } from 'jotai';

export function atomWithDebounce<T>(sourceAtom: any, delay: number) {
  const debouncedAtom = atom(
    (get) => get(sourceAtom),
    (get, set, update: T) => {
      clearTimeout(get(timeoutAtom));
      
      const newTimeout = setTimeout(() => {
        set(sourceAtom, update);
      }, delay);
      
      set(timeoutAtom, newTimeout);
    }
  );
  
  const timeoutAtom = atom<NodeJS.Timeout | null>(null);
  
  return debouncedAtom;
}
```

## Valtio: Proxy-Based State Management

### Valtio Fundamentals

```typescript
// stores/valtioStore.ts
import { proxy, useSnapshot, subscribe, ref } from 'valtio';
import { subscribeKey, watch } from 'valtio/utils';
import { devtools } from 'valtio/utils';

// Basic proxy state
interface AppState {
  user: {
    id: string | null;
    name: string;
    email: string;
    preferences: {
      theme: 'light' | 'dark';
      language: string;
      notifications: boolean;
    };
  };
  products: Product[];
  cart: {
    items: Array<{
      productId: string;
      quantity: number;
      price: number;
    }>;
    total: number;
  };
  ui: {
    loading: Record<string, boolean>;
    errors: Record<string, string | null>;
    modals: {
      productModal: {
        isOpen: boolean;
        productId: string | null;
      };
      cartModal: {
        isOpen: boolean;
      };
    };
  };
}

export const appState = proxy<AppState>({
  user: {
    id: null,
    name: '',
    email: '',
    preferences: {
      theme: 'light',
      language: 'en',
      notifications: true,
    },
  },
  products: [],
  cart: {
    items: [],
    total: 0,
  },
  ui: {
    loading: {},
    errors: {},
    modals: {
      productModal: {
        isOpen: false,
        productId: null,
      },
      cartModal: {
        isOpen: false,
      },
    },
  },
});

// Add devtools support
devtools(appState, { name: 'AppState', enabled: true });

// Actions (mutations)
export const actions = {
  // User actions
  setUser: (user: Partial<AppState['user']>) => {
    Object.assign(appState.user, user);
  },

  updateUserPreferences: (preferences: Partial<AppState['user']['preferences']>) => {
    Object.assign(appState.user.preferences, preferences);
  },

  logout: () => {
    appState.user.id = null;
    appState.user.name = '';
    appState.user.email = '';
    appState.cart.items = [];
    appState.cart.total = 0;
  },

  // Product actions
  setProducts: (products: Product[]) => {
    appState.products = products;
  },

  addProduct: (product: Product) => {
    appState.products.push(product);
  },

  updateProduct: (id: string, updates: Partial<Product>) => {
    const index = appState.products.findIndex(p => p.id === id);
    if (index !== -1) {
      Object.assign(appState.products[index], updates);
    }
  },

  removeProduct: (id: string) => {
    const index = appState.products.findIndex(p => p.id === id);
    if (index !== -1) {
      appState.products.splice(index, 1);
    }
  },

  // Cart actions
  addToCart: (productId: string, price: number) => {
    const existingItem = appState.cart.items.find(item => item.productId === productId);
    
    if (existingItem) {
      existingItem.quantity += 1;
    } else {
      appState.cart.items.push({
        productId,
        quantity: 1,
        price,
      });
    }
    
    actions.updateCartTotal();
  },

  removeFromCart: (productId: string) => {
    const index = appState.cart.items.findIndex(item => item.productId === productId);
    if (index !== -1) {
      appState.cart.items.splice(index, 1);
      actions.updateCartTotal();
    }
  },

  updateCartItemQuantity: (productId: string, quantity: number) => {
    const item = appState.cart.items.find(item => item.productId === productId);
    if (item) {
      if (quantity <= 0) {
        actions.removeFromCart(productId);
      } else {
        item.quantity = quantity;
        actions.updateCartTotal();
      }
    }
  },

  updateCartTotal: () => {
    appState.cart.total = appState.cart.items.reduce(
      (total, item) => total + (item.price * item.quantity),
      0
    );
  },

  clearCart: () => {
    appState.cart.items = [];
    appState.cart.total = 0;
  },

  // UI actions
  setLoading: (key: string, loading: boolean) => {
    appState.ui.loading[key] = loading;
  },

  setError: (key: string, error: string | null) => {
    appState.ui.errors[key] = error;
  },

  openProductModal: (productId: string) => {
    appState.ui.modals.productModal.isOpen = true;
    appState.ui.modals.productModal.productId = productId;
  },

  closeProductModal: () => {
    appState.ui.modals.productModal.isOpen = false;
    appState.ui.modals.productModal.productId = null;
  },

  toggleCartModal: () => {
    appState.ui.modals.cartModal.isOpen = !appState.ui.modals.cartModal.isOpen;
  },
};

// Computed values (derived state)
export const computed = {
  get cartItemCount() {
    return appState.cart.items.reduce((total, item) => total + item.quantity, 0);
  },

  get productsInCart() {
    return appState.cart.items.map(cartItem => {
      const product = appState.products.find(p => p.id === cartItem.productId);
      return {
        ...cartItem,
        product,
      };
    });
  },

  get productCategories() {
    return Array.from(new Set(appState.products.map(p => p.category))).sort();
  },

  get isUserLoggedIn() {
    return appState.user.id !== null;
  },
};

// Selectors for specific data
export const selectors = {
  productById: (id: string) => {
    return appState.products.find(p => p.id === id);
  },

  productsByCategory: (category: string) => {
    return appState.products.filter(p => p.category === category);
  },

  cartItemByProductId: (productId: string) => {
    return appState.cart.items.find(item => item.productId === productId);
  },
};

// Subscriptions and watchers
subscribe(appState, () => {
  console.log('State changed:', appState);
});

// Subscribe to specific keys
subscribeKey(appState.user, 'preferences', (preferences) => {
  // Save preferences to localStorage
  localStorage.setItem('userPreferences', JSON.stringify(preferences));
});

subscribeKey(appState.cart, 'items', (items) => {
  // Auto-save cart to localStorage
  localStorage.setItem('cart', JSON.stringify(items));
});

// Watch computed values
watch((get) => {
  const itemCount = get(computed).cartItemCount;
  // Update page title with cart count
  document.title = itemCount > 0 
    ? `My Store (${itemCount})` 
    : 'My Store';
});

// React hooks for using Valtio state
export const useAppState = () => {
  return useSnapshot(appState);
};

export const useUserState = () => {
  return useSnapshot(appState.user);
};

export const useCartState = () => {
  return useSnapshot(appState.cart);
};

export const useUIState = () => {
  return useSnapshot(appState.ui);
};

export const useProductsState = () => {
  return useSnapshot(appState.products);
};
```

### Advanced Valtio Patterns

```typescript
// stores/valtioAdvanced.ts
import { proxy, useSnapshot, ref, snapshot } from 'valtio';
import { proxyMap, proxySet } from 'valtio/utils';

// Using ref() for non-reactive data
interface FileUpload {
  file: File; // Won't be reactive
  progress: number;
  status: 'pending' | 'uploading' | 'completed' | 'error';
  id: string;
}

export const uploadState = proxy({
  uploads: new Map<string, FileUpload>(),
  
  addUpload: (file: File) => {
    const id = crypto.randomUUID();
    uploadState.uploads.set(id, {
      file: ref(file), // File object won't be proxied
      progress: 0,
      status: 'pending',
      id,
    });
    return id;
  },

  updateUploadProgress: (id: string, progress: number) => {
    const upload = uploadState.uploads.get(id);
    if (upload) {
      upload.progress = progress;
    }
  },

  setUploadStatus: (id: string, status: FileUpload['status']) => {
    const upload = uploadState.uploads.get(id);
    if (upload) {
      upload.status = status;
    }
  },

  removeUpload: (id: string) => {
    uploadState.uploads.delete(id);
  },
});

// Using proxyMap and proxySet for better collection handling
export const collectionState = proxy({
  // ProxyMap maintains reactivity for Map operations
  userProfiles: proxyMap<string, UserProfile>(),
  
  // ProxySet maintains reactivity for Set operations
  selectedProducts: proxySet<string>(),
  favoriteProducts: proxySet<string>(),
  
  // Actions
  setUserProfile: (id: string, profile: UserProfile) => {
    collectionState.userProfiles.set(id, profile);
  },

  selectProduct: (productId: string) => {
    collectionState.selectedProducts.add(productId);
  },

  deselectProduct: (productId: string) => {
    collectionState.selectedProducts.delete(productId);
  },

  toggleProductSelection: (productId: string) => {
    if (collectionState.selectedProducts.has(productId)) {
      collectionState.selectedProducts.delete(productId);
    } else {
      collectionState.selectedProducts.add(productId);
    }
  },

  toggleFavorite: (productId: string) => {
    if (collectionState.favoriteProducts.has(productId)) {
      collectionState.favoriteProducts.delete(productId);
    } else {
      collectionState.favoriteProducts.add(productId);
    }
  },

  clearSelections: () => {
    collectionState.selectedProducts.clear();
  },
});

// Async actions with proper error handling
export const asyncActions = {
  async fetchProducts() {
    actions.setLoading('products', true);
    actions.setError('products', null);
    
    try {
      const response = await fetch('/api/products');
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const products = await response.json();
      actions.setProducts(products);
    } catch (error) {
      actions.setError('products', error instanceof Error ? error.message : 'Unknown error');
    } finally {
      actions.setLoading('products', false);
    }
  },

  async createProduct(productData: Omit<Product, 'id'>) {
    actions.setLoading('createProduct', true);
    actions.setError('createProduct', null);
    
    try {
      const response = await fetch('/api/products', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(productData),
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const newProduct = await response.json();
      actions.addProduct(newProduct);
      
      return newProduct;
    } catch (error) {
      actions.setError('createProduct', error instanceof Error ? error.message : 'Unknown error');
      throw error;
    } finally {
      actions.setLoading('createProduct', false);
    }
  },

  async uploadFile(file: File, onProgress?: (progress: number) => void) {
    const uploadId = uploadState.addUpload(file);
    uploadState.setUploadStatus(uploadId, 'uploading');
    
    try {
      const formData = new FormData();
      formData.append('file', file);
      
      const response = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
      });
      
      if (!response.ok) {
        throw new Error(`Upload failed: ${response.status}`);
      }
      
      // Simulate progress updates
      const total = file.size;
      let loaded = 0;
      
      const reader = response.body?.getReader();
      if (reader) {
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          
          loaded += value.length;
          const progress = Math.round((loaded / total) * 100);
          uploadState.updateUploadProgress(uploadId, progress);
          onProgress?.(progress);
        }
      }
      
      uploadState.setUploadStatus(uploadId, 'completed');
      
      const result = await response.json();
      return result;
      
    } catch (error) {
      uploadState.setUploadStatus(uploadId, 'error');
      throw error;
    }
  },
};

// Time-travel debugging with snapshots
export const debugActions = {
  saveSnapshot: () => {
    const snap = snapshot(appState);
    debugState.snapshots.push({
      timestamp: Date.now(),
      state: snap,
    });
  },

  restoreSnapshot: (index: number) => {
    const snapshot = debugState.snapshots[index];
    if (snapshot) {
      Object.assign(appState, snapshot.state);
    }
  },

  clearSnapshots: () => {
    debugState.snapshots = [];
  },
};

const debugState = proxy({
  snapshots: [] as Array<{
    timestamp: number;
    state: any;
  }>,
});

// Middleware for automatic snapshots
let lastSnapshotTime = 0;
subscribe(appState, () => {
  const now = Date.now();
  // Save snapshot every 5 seconds of activity
  if (now - lastSnapshotTime > 5000) {
    debugActions.saveSnapshot();
    lastSnapshotTime = now;
  }
});

// Custom hooks for component optimization
export const useProductById = (id: string) => {
  const products = useSnapshot(appState.products);
  return products.find(p => p.id === id);
};

export const useCartItemCount = () => {
  const cart = useSnapshot(appState.cart);
  return cart.items.reduce((total, item) => total + item.quantity, 0);
};

export const useIsProductInCart = (productId: string) => {
  const cart = useSnapshot(appState.cart);
  return cart.items.some(item => item.productId === productId);
};

export const useProductCategories = () => {
  const products = useSnapshot(appState.products);
  return Array.from(new Set(products.map(p => p.category))).sort();
};
```

## React Integration Examples

### Using Zustand in Components

```typescript
// components/ProductList.tsx
import React from 'react';
import { useProductStore, useProductActions, useProductData } from '../stores/useProductStore';

export function ProductList() {
  const { filteredProducts, loading, error } = useProductData();
  const { setProducts } = useProductActions();

  React.useEffect(() => {
    async function fetchProducts() {
      try {
        const response = await fetch('/api/products');
        const products = await response.json();
        setProducts(products);
      } catch (error) {
        console.error('Failed to fetch products:', error);
      }
    }

    fetchProducts();
  }, [setProducts]);

  if (loading) return <div>Loading products...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div className="product-list">
      {filteredProducts.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

// Optimized component that only re-renders when specific data changes
export function ProductStats() {
  const totalValue = useProductStore(state => state.totalValue());
  const categoryCounts = useProductStore(state => state.categoryCounts());
  
  return (
    <div className="product-stats">
      <h3>Statistics</h3>
      <p>Total Value: ${totalValue.toFixed(2)}</p>
      <div className="category-counts">
        {Object.entries(categoryCounts).map(([category, count]) => (
          <span key={category} className="category-stat">
            {category}: {count}
          </span>
        ))}
      </div>
    </div>
  );
}
```

### Using Jotai in Components

```typescript
// components/JotaiComponents.tsx
import React from 'react';
import { useAtom, useAtomValue, useSetAtom } from 'jotai';
import { 
  productsAtom, 
  filteredProductsAtom, 
  filtersAtom, 
  addProductAtom,
  fetchProductsAtom,
  productStatsAtom
} from '../atoms/productAtoms';

export function JotaiProductList() {
  const products = useAtomValue(filteredProductsAtom);
  const fetchProducts = useSetAtom(fetchProductsAtom);

  React.useEffect(() => {
    fetchProducts();
  }, [fetchProducts]);

  return (
    <div className="product-list">
      {products.map(product => (
        <JotaiProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

export function JotaiFilters() {
  const [filters, setFilters] = useAtom(filtersAtom);

  const updateFilter = (key: string, value: any) => {
    setFilters(prev => ({ ...prev, [key]: value }));
  };

  return (
    <div className="filters">
      <input
        type="text"
        placeholder="Search products..."
        value={filters.search}
        onChange={(e) => updateFilter('search', e.target.value)}
      />
      
      <select
        value={filters.category}
        onChange={(e) => updateFilter('category', e.target.value)}
      >
        <option value="">All Categories</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>
      
      <select
        value={filters.sortBy}
        onChange={(e) => updateFilter('sortBy', e.target.value)}
      >
        <option value="name">Name</option>
        <option value="price">Price</option>
        <option value="rating">Rating</option>
      </select>
    </div>
  );
}

export function JotaiProductStats() {
  const stats = useAtomValue(productStatsAtom);
  
  return (
    <div className="product-stats">
      <h3>Statistics</h3>
      <p>Total Products: {stats.total}</p>
      <p>Average Price: ${stats.averagePrice.toFixed(2)}</p>
      <p>Total Value: ${stats.totalValue.toFixed(2)}</p>
      
      <div className="category-breakdown">
        <h4>By Category:</h4>
        {Object.entries(stats.byCategory).map(([category, count]) => (
          <div key={category}>
            {category}: {count}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Using Valtio in Components

```typescript
// components/ValtioComponents.tsx
import React from 'react';
import { 
  useAppState, 
  useProductsState, 
  useCartState,
  actions, 
  asyncActions,
  computed,
  selectors 
} from '../stores/valtioStore';

export function ValtioProductList() {
  const products = useProductsState();

  React.useEffect(() => {
    asyncActions.fetchProducts();
  }, []);

  return (
    <div className="product-list">
      {products.map(product => (
        <ValtioProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

export function ValtioCart() {
  const cart = useCartState();
  const appState = useAppState();

  return (
    <div className="cart">
      <h3>Shopping Cart ({computed.cartItemCount})</h3>
      
      {cart.items.length === 0 ? (
        <p>Your cart is empty</p>
      ) : (
        <>
          {computed.productsInCart.map(({ productId, quantity, price, product }) => (
            <div key={productId} className="cart-item">
              <span>{product?.name || 'Unknown Product'}</span>
              <span>${price.toFixed(2)} x {quantity}</span>
              <button
                onClick={() => actions.updateCartItemQuantity(productId, quantity - 1)}
              >
                -
              </button>
              <button
                onClick={() => actions.updateCartItemQuantity(productId, quantity + 1)}
              >
                +
              </button>
              <button
                onClick={() => actions.removeFromCart(productId)}
              >
                Remove
              </button>
            </div>
          ))}
          
          <div className="cart-total">
            Total: ${cart.total.toFixed(2)}
          </div>
          
          <button onClick={actions.clearCart}>
            Clear Cart
          </button>
        </>
      )}
    </div>
  );
}

function ValtioProductCard({ product }: { product: Product }) {
  const cartItem = selectors.cartItemByProductId(product.id);
  
  return (
    <div className="product-card">
      <h4>{product.name}</h4>
      <p>${product.price.toFixed(2)}</p>
      
      {cartItem ? (
        <div>
          <span>In Cart: {cartItem.quantity}</span>
          <button
            onClick={() => actions.updateCartItemQuantity(product.id, cartItem.quantity + 1)}
          >
            Add More
          </button>
        </div>
      ) : (
        <button
          onClick={() => actions.addToCart(product.id, product.price)}
        >
          Add to Cart
        </button>
      )}
    </div>
  );
}
```

## Performance Comparison and Best Practices

### Library Comparison

| Feature | Zustand | Jotai | Valtio |
|---------|---------|-------|--------|
| Bundle Size | ~1.5kb | ~3kb | ~4kb |
| Learning Curve | Easy | Moderate | Easy |
| TypeScript Support | Excellent | Excellent | Good |
| DevTools | Yes | Yes | Yes |
| Persistence | Built-in | Utilities | Custom |
| Async Handling | Manual | Built-in | Manual |
| Boilerplate | Low | Very Low | Very Low |
| Selective Updates | Manual | Automatic | Automatic |

### Best Practices

**Zustand:**
- Use store slices for better organization
- Implement selectors for performance optimization
- Use middleware for cross-cutting concerns
- Keep actions close to state for better encapsulation

**Jotai:**
- Create focused atoms for specific concerns
- Use derived atoms for computed values
- Leverage atom families for dynamic data
- Use async atoms for data fetching

**Valtio:**
- Use ref() for non-reactive data like File objects
- Implement computed values as getters
- Use proxyMap/proxySet for better collection handling
- Subscribe to specific changes for side effects

**General Performance Tips:**
- Minimize state granularity to reduce unnecessary re-renders
- Use React.memo() for expensive components
- Implement proper error boundaries
- Use React DevTools Profiler to identify bottlenecks
- Consider code splitting for large state management files

## Interview-Ready State Management Summary

**Modern State Management Evolution:**
1. **Beyond Redux** - Lighter alternatives with less boilerplate and better developer experience
2. **Atomic Design** - Jotai's bottom-up approach with composable atoms
3. **Proxy-Based** - Valtio's mutation-friendly API with automatic reactivity
4. **Store-Based** - Zustand's simple yet powerful centralized state

**Advanced Patterns:**
- Optimistic updates with error rollback
- Async state management with proper loading/error states
- Computed values and derived state
- Middleware and plugins for enhanced functionality
- Time-travel debugging and state snapshots

**Performance Considerations:**
- Selective subscriptions to prevent unnecessary re-renders
- Proper memoization strategies
- Bundle size optimization
- Memory leak prevention in subscriptions

**Key Interview Topics:** State management paradigms comparison, atomic vs store-based approaches, performance optimization techniques, TypeScript integration patterns, async state handling, and modern React state management best practices.