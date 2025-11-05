# React Server Components and Advanced Patterns

React Server Components represent a paradigm shift in modern React development, enabling server-side rendering at the component level with zero client-side JavaScript. This comprehensive guide covers Server Components, Suspense boundaries, streaming, concurrent features, and enterprise architecture patterns for high-performance applications.

## React Server Components Architecture

### Server Component Fundamentals

```typescript
// app/server-components/ProductCatalog.tsx
import { Suspense } from 'react';
import { ProductGrid } from './ProductGrid';
import { ProductFilters } from './ProductFilters';
import { ProductSearch } from './ProductSearch';
import { ErrorBoundary } from '../components/ErrorBoundary';

// Server Component - runs on server, no client-side JavaScript
export default async function ProductCatalog({
  searchParams,
}: {
  searchParams: { category?: string; search?: string; page?: string };
}) {
  // Server-side data fetching with parallel loading
  const [products, categories, recommendations] = await Promise.all([
    fetchProducts({
      category: searchParams.category,
      search: searchParams.search,
      page: parseInt(searchParams.page || '1'),
      limit: 20,
    }),
    fetchCategories(),
    fetchRecommendations(searchParams.category),
  ]);

  return (
    <div className="product-catalog">
      <div className="catalog-header">
        <h1>Product Catalog</h1>
        <Suspense fallback={<SearchSkeleton />}>
          <ProductSearch defaultValue={searchParams.search} />
        </Suspense>
      </div>

      <div className="catalog-content">
        <aside className="filters-sidebar">
          <ErrorBoundary fallback={<FiltersError />}>
            <Suspense fallback={<FiltersSkeleton />}>
              <ProductFilters 
                categories={categories}
                selectedCategory={searchParams.category}
              />
            </Suspense>
          </ErrorBoundary>
        </aside>

        <main className="products-main">
          <ErrorBoundary fallback={<ProductsError />}>
            <Suspense fallback={<ProductGridSkeleton />}>
              <ProductGrid 
                products={products}
                recommendations={recommendations}
              />
            </Suspense>
          </ErrorBoundary>
        </main>
      </div>
    </div>
  );
}

// Data fetching functions for Server Components
async function fetchProducts(params: {
  category?: string;
  search?: string;
  page: number;
  limit: number;
}) {
  const searchParams = new URLSearchParams();
  if (params.category) searchParams.set('category', params.category);
  if (params.search) searchParams.set('search', params.search);
  searchParams.set('page', params.page.toString());
  searchParams.set('limit', params.limit.toString());

  const response = await fetch(`${process.env.API_BASE_URL}/products?${searchParams}`, {
    headers: {
      'Authorization': `Bearer ${process.env.API_TOKEN}`,
      'Cache-Control': 'max-age=300', // 5 minutes cache
    },
    next: { revalidate: 300 }, // Next.js ISR
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch products: ${response.status}`);
  }

  return response.json();
}

async function fetchCategories() {
  const response = await fetch(`${process.env.API_BASE_URL}/categories`, {
    headers: {
      'Authorization': `Bearer ${process.env.API_TOKEN}`,
    },
    next: { revalidate: 3600 }, // 1 hour cache
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch categories: ${response.status}`);
  }

  return response.json();
}

async function fetchRecommendations(category?: string) {
  if (!category) return [];

  const response = await fetch(
    `${process.env.API_BASE_URL}/recommendations?category=${category}`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.API_TOKEN}`,
      },
      next: { revalidate: 600 }, // 10 minutes cache
    }
  );

  if (!response.ok) {
    return []; // Graceful degradation for recommendations
  }

  return response.json();
}
```

### Advanced Suspense and Streaming Patterns

```typescript
// app/components/StreamingDataComponents.tsx
import { Suspense } from 'react';
import { unstable_noStore as noStore } from 'next/cache';

// High-priority component that loads first
export async function CriticalData() {
  noStore(); // Opt out of caching for real-time data
  
  const criticalData = await fetch(`${process.env.API_BASE_URL}/critical`, {
    headers: { 'Priority': 'high' },
  }).then(res => res.json());

  return (
    <div className="critical-section">
      <h2>Critical Information</h2>
      <div className="critical-data">
        {criticalData.items.map((item: any) => (
          <div key={item.id} className="critical-item">
            {item.title}
          </div>
        ))}
      </div>
    </div>
  );
}

// Progressive loading with nested Suspense boundaries
export function ProgressiveContent() {
  return (
    <div className="progressive-content">
      {/* Load critical content first */}
      <Suspense fallback={<CriticalSkeleton />}>
        <CriticalData />
      </Suspense>

      {/* Load secondary content in parallel */}
      <div className="secondary-content">
        <Suspense fallback={<SecondarySkeletonA />}>
          <SecondaryDataA />
        </Suspense>
        
        <Suspense fallback={<SecondarySkeletonB />}>
          <SecondaryDataB />
        </Suspense>
      </div>

      {/* Load tertiary content last */}
      <Suspense fallback={<TertiaryContentSkeleton />}>
        <TertiaryContent />
      </Suspense>
    </div>
  );
}

// Streaming component with incremental loading
export async function IncrementalFeed({
  pageSize = 10,
  initialPage = 1,
}: {
  pageSize?: number;
  initialPage?: number;
}) {
  const initialData = await fetchFeedPage(initialPage, pageSize);

  return (
    <div className="incremental-feed">
      <FeedItems items={initialData.items} />
      
      {initialData.hasMore && (
        <Suspense fallback={<LoadMoreSkeleton />}>
          <IncrementalLoader 
            nextPage={initialPage + 1}
            pageSize={pageSize}
          />
        </Suspense>
      )}
    </div>
  );
}

async function SecondaryDataA() {
  await new Promise(resolve => setTimeout(resolve, 1000)); // Simulate delay
  const data = await fetch(`${process.env.API_BASE_URL}/secondary-a`).then(res => res.json());
  
  return (
    <div className="secondary-a">
      <h3>Secondary Data A</h3>
      <p>{data.content}</p>
    </div>
  );
}

async function SecondaryDataB() {
  await new Promise(resolve => setTimeout(resolve, 1500)); // Simulate delay
  const data = await fetch(`${process.env.API_BASE_URL}/secondary-b`).then(res => res.json());
  
  return (
    <div className="secondary-b">
      <h3>Secondary Data B</h3>
      <p>{data.content}</p>
    </div>
  );
}

async function TertiaryContent() {
  await new Promise(resolve => setTimeout(resolve, 2000)); // Simulate delay
  const data = await fetch(`${process.env.API_BASE_URL}/tertiary`).then(res => res.json());
  
  return (
    <div className="tertiary-content">
      <h3>Tertiary Content</h3>
      <div className="tertiary-grid">
        {data.items.map((item: any) => (
          <div key={item.id} className="tertiary-item">
            {item.title}
          </div>
        ))}
      </div>
    </div>
  );
}

async function fetchFeedPage(page: number, pageSize: number) {
  const response = await fetch(
    `${process.env.API_BASE_URL}/feed?page=${page}&size=${pageSize}`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.API_TOKEN}`,
      },
    }
  );

  return response.json();
}
```

### Client-Server Component Composition

```typescript
// app/components/HybridComponents.tsx
'use client';

import { useState, useTransition, startTransition } from 'react';
import { useRouter, useSearchParams } from 'next/navigation';

// Client Component that wraps Server Components
export function InteractiveProductCatalog({
  children,
  initialFilters,
}: {
  children: React.ReactNode;
  initialFilters: any;
}) {
  const [filters, setFilters] = useState(initialFilters);
  const [isPending, startTransition] = useTransition();
  const router = useRouter();
  const searchParams = useSearchParams();

  const updateFilters = (newFilters: any) => {
    setFilters(newFilters);
    
    // Create new search params
    const params = new URLSearchParams(searchParams);
    Object.entries(newFilters).forEach(([key, value]) => {
      if (value) {
        params.set(key, value as string);
      } else {
        params.delete(key);
      }
    });

    // Navigate with transition for smoother UX
    startTransition(() => {
      router.push(`/products?${params.toString()}`);
    });
  };

  return (
    <div className={`catalog-container ${isPending ? 'loading' : ''}`}>
      <div className="catalog-controls">
        <FilterControls 
          filters={filters}
          onFiltersChange={updateFilters}
          disabled={isPending}
        />
        
        {isPending && (
          <div className="loading-indicator">
            <span>Updating results...</span>
          </div>
        )}
      </div>

      <div className="catalog-results">
        {children}
      </div>
    </div>
  );
}

// Client Component for interactive filters
function FilterControls({
  filters,
  onFiltersChange,
  disabled,
}: {
  filters: any;
  onFiltersChange: (filters: any) => void;
  disabled: boolean;
}) {
  const handleFilterChange = (key: string, value: any) => {
    onFiltersChange({
      ...filters,
      [key]: value,
    });
  };

  return (
    <div className="filter-controls">
      <div className="filter-group">
        <label htmlFor="category">Category</label>
        <select
          id="category"
          value={filters.category || ''}
          onChange={(e) => handleFilterChange('category', e.target.value)}
          disabled={disabled}
        >
          <option value="">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="clothing">Clothing</option>
          <option value="books">Books</option>
        </select>
      </div>

      <div className="filter-group">
        <label htmlFor="price-range">Price Range</label>
        <select
          id="price-range"
          value={filters.priceRange || ''}
          onChange={(e) => handleFilterChange('priceRange', e.target.value)}
          disabled={disabled}
        >
          <option value="">Any Price</option>
          <option value="0-50">$0 - $50</option>
          <option value="50-100">$50 - $100</option>
          <option value="100+">$100+</option>
        </select>
      </div>

      <div className="filter-group">
        <label htmlFor="sort">Sort By</label>
        <select
          id="sort"
          value={filters.sort || 'relevance'}
          onChange={(e) => handleFilterChange('sort', e.target.value)}
          disabled={disabled}
        >
          <option value="relevance">Relevance</option>
          <option value="price-asc">Price: Low to High</option>
          <option value="price-desc">Price: High to Low</option>
          <option value="rating">Customer Rating</option>
        </select>
      </div>
    </div>
  );
}
```

## Concurrent Features and Performance Optimization

### Advanced Concurrent Patterns

```typescript
// hooks/useConcurrentFeatures.ts
import { 
  useState, 
  useTransition, 
  useDeferredValue, 
  useCallback, 
  useOptimistic,
  startTransition 
} from 'react';

// Hook for optimistic updates with Server Actions
export function useOptimisticCart() {
  const [cart, setCart] = useState<CartItem[]>([]);
  const [optimisticCart, addOptimisticItem] = useOptimistic(
    cart,
    (state, newItem: CartItem) => {
      const existingIndex = state.findIndex(item => item.id === newItem.id);
      if (existingIndex >= 0) {
        return state.map((item, index) => 
          index === existingIndex 
            ? { ...item, quantity: item.quantity + newItem.quantity }
            : item
        );
      }
      return [...state, newItem];
    }
  );

  const [isPending, startTransition] = useTransition();

  const addToCart = useCallback(async (item: CartItem) => {
    // Optimistically update UI immediately
    addOptimisticItem(item);

    // Perform actual server update
    startTransition(async () => {
      try {
        const updatedCart = await addItemToCartAction(item);
        setCart(updatedCart);
      } catch (error) {
        // Revert optimistic update on error
        console.error('Failed to add item to cart:', error);
        // The optimistic state will automatically revert
      }
    });
  }, [addOptimisticItem]);

  return {
    cart: optimisticCart,
    addToCart,
    isPending,
  };
}

// Hook for deferred search with performance optimization
export function useDeferredSearch() {
  const [searchQuery, setSearchQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  // Defer the expensive search operation
  const deferredQuery = useDeferredValue(searchQuery);
  
  const updateSearch = useCallback((query: string) => {
    setSearchQuery(query);
    
    // Mark search as low-priority
    startTransition(() => {
      // This will be deferred until higher priority updates complete
      performSearch(query);
    });
  }, []);

  return {
    searchQuery,
    deferredQuery,
    updateSearch,
    isPending,
  };
}

// Hook for concurrent data fetching
export function useConcurrentData<T>(
  fetcher: () => Promise<T>,
  dependencies: any[] = []
) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isPending, startTransition] = useTransition();

  const refresh = useCallback(() => {
    startTransition(async () => {
      try {
        setError(null);
        const result = await fetcher();
        setData(result);
      } catch (err) {
        setError(err instanceof Error ? err : new Error('Unknown error'));
      }
    });
  }, dependencies);

  return {
    data,
    error,
    isPending,
    refresh,
  };
}

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

async function addItemToCartAction(item: CartItem): Promise<CartItem[]> {
  // Simulate server action
  await new Promise(resolve => setTimeout(resolve, 1000));
  
  const response = await fetch('/api/cart/add', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(item),
  });
  
  if (!response.ok) {
    throw new Error('Failed to add item to cart');
  }
  
  return response.json();
}

async function performSearch(query: string): Promise<any[]> {
  if (!query.trim()) return [];
  
  const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
  return response.json();
}
```

### Performance Monitoring and Optimization

```typescript
// components/PerformanceMonitor.tsx
'use client';

import { useEffect, useCallback } from 'react';
import { getCLS, getFCP, getFID, getLCP, getTTFB } from 'web-vitals';

interface PerformanceMetrics {
  cls: number;
  fcp: number;
  fid: number;
  lcp: number;
  ttfb: number;
}

export function PerformanceMonitor() {
  const reportMetric = useCallback((metric: any) => {
    // Send to analytics service
    if (typeof window !== 'undefined' && window.gtag) {
      window.gtag('event', metric.name, {
        value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
        event_category: 'Web Vitals',
        event_label: metric.id,
        non_interaction: true,
      });
    }

    // Send to custom analytics endpoint
    fetch('/api/analytics/web-vitals', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: metric.name,
        value: metric.value,
        id: metric.id,
        delta: metric.delta,
        rating: metric.rating,
        timestamp: Date.now(),
        url: window.location.href,
        userAgent: navigator.userAgent,
      }),
    }).catch(console.error);
  }, []);

  useEffect(() => {
    // Measure Core Web Vitals
    getCLS(reportMetric);
    getFCP(reportMetric);
    getFID(reportMetric);
    getLCP(reportMetric);
    getTTFB(reportMetric);

    // Custom performance measurements
    measureCustomMetrics();
  }, [reportMetric]);

  return null; // This component doesn't render anything
}

function measureCustomMetrics() {
  // Measure React-specific performance
  if (typeof window !== 'undefined' && window.performance) {
    // Measure hydration time
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name === 'react-hydration') {
          fetch('/api/analytics/custom-metrics', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              name: 'react-hydration',
              value: entry.duration,
              timestamp: Date.now(),
            }),
          }).catch(console.error);
        }
      }
    });

    observer.observe({ entryTypes: ['measure'] });

    // Measure component render times
    const componentObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name.startsWith('⚛️')) {
          fetch('/api/analytics/component-metrics', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              component: entry.name,
              duration: entry.duration,
              timestamp: Date.now(),
            }),
          }).catch(console.error);
        }
      }
    });

    componentObserver.observe({ entryTypes: ['measure'] });
  }
}

// React DevTools Profiler integration
export function ProfiledComponent({ 
  children, 
  id,
}: { 
  children: React.ReactNode;
  id: string;
}) {
  const onRender = useCallback((
    id: string,
    phase: 'mount' | 'update',
    actualDuration: number,
    baseDuration: number,
    startTime: number,
    commitTime: number,
    interactions: Set<any>
  ) => {
    // Send profiling data to analytics
    fetch('/api/analytics/profiler', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        componentId: id,
        phase,
        actualDuration,
        baseDuration,
        startTime,
        commitTime,
        interactionCount: interactions.size,
        timestamp: Date.now(),
      }),
    }).catch(console.error);
  }, []);

  return (
    <Profiler id={id} onRender={onRender}>
      {children}
    </Profiler>
  );
}
```

## Server Actions and Form Handling

### Advanced Server Actions

```typescript
// app/actions/productActions.ts
'use server';

import { revalidateTag, revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';

// Validation schemas
const CreateProductSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  description: z.string().min(10, 'Description must be at least 10 characters'),
  price: z.number().positive('Price must be positive'),
  category: z.string().min(1, 'Category is required'),
  tags: z.array(z.string()).optional(),
  images: z.array(z.string().url()).min(1, 'At least one image is required'),
});

const UpdateProductSchema = CreateProductSchema.partial().extend({
  id: z.string().uuid(),
});

// Server action for creating products
export async function createProduct(formData: FormData) {
  try {
    // Extract and validate data
    const rawData = {
      name: formData.get('name'),
      description: formData.get('description'),
      price: Number(formData.get('price')),
      category: formData.get('category'),
      tags: formData.getAll('tags'),
      images: formData.getAll('images'),
    };

    const validatedData = CreateProductSchema.parse(rawData);

    // Check permissions
    const session = await getServerSession();
    if (!session?.user?.id) {
      throw new Error('Unauthorized');
    }

    // Create product
    const product = await fetch(`${process.env.API_BASE_URL}/products`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.API_TOKEN}`,
      },
      body: JSON.stringify({
        ...validatedData,
        userId: session.user.id,
      }),
    });

    if (!product.ok) {
      throw new Error(`Failed to create product: ${product.status}`);
    }

    const createdProduct = await product.json();

    // Revalidate related cache entries
    revalidateTag('products');
    revalidateTag(`category:${validatedData.category}`);
    revalidatePath('/products');
    revalidatePath('/admin/products');

    // Return success response
    return {
      success: true,
      product: createdProduct,
      message: 'Product created successfully',
    };

  } catch (error) {
    console.error('Create product error:', error);
    
    if (error instanceof z.ZodError) {
      return {
        success: false,
        errors: error.flatten().fieldErrors,
        message: 'Validation failed',
      };
    }

    return {
      success: false,
      message: error instanceof Error ? error.message : 'Unknown error occurred',
    };
  }
}

// Server action for updating products with optimistic updates
export async function updateProduct(formData: FormData) {
  try {
    const rawData = {
      id: formData.get('id'),
      name: formData.get('name'),
      description: formData.get('description'),
      price: formData.get('price') ? Number(formData.get('price')) : undefined,
      category: formData.get('category'),
    };

    const validatedData = UpdateProductSchema.parse(rawData);

    // Check permissions
    const session = await getServerSession();
    if (!session?.user?.id) {
      throw new Error('Unauthorized');
    }

    // Update product
    const response = await fetch(
      `${process.env.API_BASE_URL}/products/${validatedData.id}`,
      {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${process.env.API_TOKEN}`,
        },
        body: JSON.stringify(validatedData),
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to update product: ${response.status}`);
    }

    const updatedProduct = await response.json();

    // Revalidate specific product and related cache
    revalidateTag(`product:${validatedData.id}`);
    revalidateTag('products');
    if (validatedData.category) {
      revalidateTag(`category:${validatedData.category}`);
    }

    return {
      success: true,
      product: updatedProduct,
      message: 'Product updated successfully',
    };

  } catch (error) {
    console.error('Update product error:', error);
    
    if (error instanceof z.ZodError) {
      return {
        success: false,
        errors: error.flatten().fieldErrors,
        message: 'Validation failed',
      };
    }

    return {
      success: false,
      message: error instanceof Error ? error.message : 'Unknown error occurred',
    };
  }
}

// Server action for deleting products
export async function deleteProduct(productId: string) {
  try {
    // Check permissions
    const session = await getServerSession();
    if (!session?.user?.id) {
      throw new Error('Unauthorized');
    }

    // Get product details for cache invalidation
    const productResponse = await fetch(
      `${process.env.API_BASE_URL}/products/${productId}`,
      {
        headers: {
          'Authorization': `Bearer ${process.env.API_TOKEN}`,
        },
      }
    );

    if (!productResponse.ok) {
      throw new Error('Product not found');
    }

    const product = await productResponse.json();

    // Delete product
    const deleteResponse = await fetch(
      `${process.env.API_BASE_URL}/products/${productId}`,
      {
        method: 'DELETE',
        headers: {
          'Authorization': `Bearer ${process.env.API_TOKEN}`,
        },
      }
    );

    if (!deleteResponse.ok) {
      throw new Error(`Failed to delete product: ${deleteResponse.status}`);
    }

    // Revalidate cache
    revalidateTag(`product:${productId}`);
    revalidateTag('products');
    revalidateTag(`category:${product.category}`);
    revalidatePath('/products');
    revalidatePath('/admin/products');

    return {
      success: true,
      message: 'Product deleted successfully',
    };

  } catch (error) {
    console.error('Delete product error:', error);
    return {
      success: false,
      message: error instanceof Error ? error.message : 'Unknown error occurred',
    };
  }
}

// Batch operations
export async function batchUpdateProducts(updates: Array<{ id: string; data: any }>) {
  try {
    const session = await getServerSession();
    if (!session?.user?.id) {
      throw new Error('Unauthorized');
    }

    const results = await Promise.allSettled(
      updates.map(async ({ id, data }) => {
        const response = await fetch(
          `${process.env.API_BASE_URL}/products/${id}`,
          {
            method: 'PATCH',
            headers: {
              'Content-Type': 'application/json',
              'Authorization': `Bearer ${process.env.API_TOKEN}`,
            },
            body: JSON.stringify(data),
          }
        );

        if (!response.ok) {
          throw new Error(`Failed to update product ${id}: ${response.status}`);
        }

        return response.json();
      })
    );

    const successful = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;

    // Revalidate all product-related cache
    revalidateTag('products');
    revalidatePath('/products');
    revalidatePath('/admin/products');

    return {
      success: true,
      message: `Batch update completed: ${successful} successful, ${failed} failed`,
      results: {
        successful,
        failed,
        details: results,
      },
    };

  } catch (error) {
    console.error('Batch update error:', error);
    return {
      success: false,
      message: error instanceof Error ? error.message : 'Batch update failed',
    };
  }
}

// Helper function to get server session
async function getServerSession() {
  // Implementation depends on your auth provider
  // This is a placeholder
  return {
    user: {
      id: 'user-123',
      email: 'user@example.com',
    },
  };
}
```

### Progressive Enhancement Forms

```typescript
// components/ProgressiveForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { useOptimistic, useRef, useEffect } from 'react';
import { createProduct, updateProduct } from '../actions/productActions';

interface Product {
  id?: string;
  name: string;
  description: string;
  price: number;
  category: string;
  tags: string[];
  images: string[];
}

export function ProductForm({ 
  product, 
  mode = 'create' 
}: { 
  product?: Product;
  mode?: 'create' | 'edit';
}) {
  const [state, formAction] = useFormState(
    mode === 'create' ? createProduct : updateProduct,
    { success: false, message: '', errors: {} }
  );
  
  const [optimisticProduct, addOptimistic] = useOptimistic(
    product,
    (currentProduct, newProduct: Product) => ({
      ...currentProduct,
      ...newProduct,
    })
  );

  const formRef = useRef<HTMLFormElement>(null);

  // Reset form on successful submission
  useEffect(() => {
    if (state.success && mode === 'create') {
      formRef.current?.reset();
    }
  }, [state.success, mode]);

  const handleSubmit = async (formData: FormData) => {
    // Optimistic update for edit mode
    if (mode === 'edit' && product) {
      const newData = {
        id: product.id,
        name: formData.get('name') as string,
        description: formData.get('description') as string,
        price: Number(formData.get('price')),
        category: formData.get('category') as string,
        tags: formData.getAll('tags') as string[],
        images: formData.getAll('images') as string[],
      };
      
      addOptimistic(newData);
    }

    // Submit form
    formAction(formData);
  };

  return (
    <form 
      ref={formRef}
      action={handleSubmit}
      className="product-form"
      noValidate
    >
      {product?.id && (
        <input type="hidden" name="id" value={product.id} />
      )}

      <div className="form-group">
        <label htmlFor="name">Product Name</label>
        <input
          type="text"
          id="name"
          name="name"
          defaultValue={optimisticProduct?.name || ''}
          required
          aria-describedby={state.errors?.name ? 'name-error' : undefined}
          className={state.errors?.name ? 'error' : ''}
        />
        {state.errors?.name && (
          <div id="name-error" className="error-message">
            {state.errors.name[0]}
          </div>
        )}
      </div>

      <div className="form-group">
        <label htmlFor="description">Description</label>
        <textarea
          id="description"
          name="description"
          defaultValue={optimisticProduct?.description || ''}
          required
          rows={4}
          aria-describedby={state.errors?.description ? 'description-error' : undefined}
          className={state.errors?.description ? 'error' : ''}
        />
        {state.errors?.description && (
          <div id="description-error" className="error-message">
            {state.errors.description[0]}
          </div>
        )}
      </div>

      <div className="form-row">
        <div className="form-group">
          <label htmlFor="price">Price ($)</label>
          <input
            type="number"
            id="price"
            name="price"
            defaultValue={optimisticProduct?.price || ''}
            required
            min="0"
            step="0.01"
            aria-describedby={state.errors?.price ? 'price-error' : undefined}
            className={state.errors?.price ? 'error' : ''}
          />
          {state.errors?.price && (
            <div id="price-error" className="error-message">
              {state.errors.price[0]}
            </div>
          )}
        </div>

        <div className="form-group">
          <label htmlFor="category">Category</label>
          <select
            id="category"
            name="category"
            defaultValue={optimisticProduct?.category || ''}
            required
            aria-describedby={state.errors?.category ? 'category-error' : undefined}
            className={state.errors?.category ? 'error' : ''}
          >
            <option value="">Select Category</option>
            <option value="electronics">Electronics</option>
            <option value="clothing">Clothing</option>
            <option value="books">Books</option>
            <option value="home">Home & Garden</option>
          </select>
          {state.errors?.category && (
            <div id="category-error" className="error-message">
              {state.errors.category[0]}
            </div>
          )}
        </div>
      </div>

      <div className="form-group">
        <label htmlFor="tags">Tags</label>
        <TagInput
          name="tags"
          defaultValue={optimisticProduct?.tags || []}
          error={state.errors?.tags?.[0]}
        />
      </div>

      <div className="form-group">
        <label htmlFor="images">Product Images</label>
        <ImageUpload
          name="images"
          defaultValue={optimisticProduct?.images || []}
          error={state.errors?.images?.[0]}
        />
      </div>

      <div className="form-actions">
        <SubmitButton mode={mode} />
        <button type="button" className="btn-secondary">
          Cancel
        </button>
      </div>

      {state.message && (
        <div className={`message ${state.success ? 'success' : 'error'}`}>
          {state.message}
        </div>
      )}
    </form>
  );
}

function SubmitButton({ mode }: { mode: 'create' | 'edit' }) {
  const { pending } = useFormStatus();

  return (
    <button 
      type="submit" 
      disabled={pending}
      className="btn-primary"
      aria-disabled={pending}
    >
      {pending ? (
        <>
          <span className="spinner" aria-hidden="true" />
          {mode === 'create' ? 'Creating...' : 'Updating...'}
        </>
      ) : (
        mode === 'create' ? 'Create Product' : 'Update Product'
      )}
    </button>
  );
}

// Enhanced input components
function TagInput({ 
  name, 
  defaultValue = [], 
  error 
}: { 
  name: string;
  defaultValue: string[];
  error?: string;
}) {
  const [tags, setTags] = useState<string[]>(defaultValue);
  const [inputValue, setInputValue] = useState('');

  const addTag = (tag: string) => {
    const trimmedTag = tag.trim();
    if (trimmedTag && !tags.includes(trimmedTag)) {
      setTags([...tags, trimmedTag]);
    }
    setInputValue('');
  };

  const removeTag = (tagToRemove: string) => {
    setTags(tags.filter(tag => tag !== tagToRemove));
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ',') {
      e.preventDefault();
      addTag(inputValue);
    } else if (e.key === 'Backspace' && !inputValue && tags.length > 0) {
      removeTag(tags[tags.length - 1]);
    }
  };

  return (
    <div className="tag-input">
      <div className="tags-container">
        {tags.map((tag, index) => (
          <span key={index} className="tag">
            {tag}
            <button
              type="button"
              onClick={() => removeTag(tag)}
              aria-label={`Remove tag ${tag}`}
            >
              ×
            </button>
            <input type="hidden" name={name} value={tag} />
          </span>
        ))}
        <input
          type="text"
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
          onKeyDown={handleKeyDown}
          onBlur={() => {
            if (inputValue.trim()) {
              addTag(inputValue);
            }
          }}
          placeholder="Add tags..."
          className={error ? 'error' : ''}
        />
      </div>
      {error && <div className="error-message">{error}</div>}
    </div>
  );
}

function ImageUpload({ 
  name, 
  defaultValue = [], 
  error 
}: { 
  name: string;
  defaultValue: string[];
  error?: string;
}) {
  const [images, setImages] = useState<string[]>(defaultValue);
  const [uploading, setUploading] = useState(false);

  const handleFileUpload = async (files: FileList) => {
    setUploading(true);
    
    try {
      const uploadPromises = Array.from(files).map(async (file) => {
        const formData = new FormData();
        formData.append('file', file);
        
        const response = await fetch('/api/upload', {
          method: 'POST',
          body: formData,
        });
        
        if (!response.ok) {
          throw new Error('Upload failed');
        }
        
        const { url } = await response.json();
        return url;
      });

      const uploadedUrls = await Promise.all(uploadPromises);
      setImages([...images, ...uploadedUrls]);
    } catch (error) {
      console.error('Upload error:', error);
    } finally {
      setUploading(false);
    }
  };

  const removeImage = (urlToRemove: string) => {
    setImages(images.filter(url => url !== urlToRemove));
  };

  return (
    <div className="image-upload">
      <div className="images-preview">
        {images.map((url, index) => (
          <div key={index} className="image-preview">
            <img src={url} alt={`Product image ${index + 1}`} />
            <button
              type="button"
              onClick={() => removeImage(url)}
              aria-label={`Remove image ${index + 1}`}
            >
              ×
            </button>
            <input type="hidden" name={name} value={url} />
          </div>
        ))}
      </div>
      
      <div className="upload-area">
        <input
          type="file"
          id="image-upload"
          multiple
          accept="image/*"
          onChange={(e) => {
            if (e.target.files) {
              handleFileUpload(e.target.files);
            }
          }}
          disabled={uploading}
        />
        <label htmlFor="image-upload" className="upload-label">
          {uploading ? 'Uploading...' : 'Choose Images'}
        </label>
      </div>
      
      {error && <div className="error-message">{error}</div>}
    </div>
  );
}

export default ProductForm;
```

## Interview-Ready React Server Components Summary

**React Server Components Architecture:**
1. **Server-Side Rendering** - Component-level SSR with zero client-side JavaScript
2. **Streaming & Suspense** - Progressive loading with nested boundaries and error handling
3. **Client-Server Composition** - Optimal hybrid patterns with interactive boundaries
4. **Performance Optimization** - Core Web Vitals monitoring and concurrent features

**Advanced Patterns:**
- Optimistic updates with Server Actions and useOptimistic
- Deferred values and transitions for better UX
- Progressive enhancement with useFormState and useFormStatus
- Intelligent cache invalidation with revalidateTag and revalidatePath

**Enterprise Features:**
- Type-safe form validation with Zod schemas
- Batch operations and error handling
- Performance monitoring and analytics integration
- Accessibility-first form design with ARIA support

**Key Interview Topics:** Server Components vs Client Components trade-offs, streaming strategies, concurrent React features, Server Actions architecture, progressive enhancement patterns, performance optimization techniques, and modern React development practices.