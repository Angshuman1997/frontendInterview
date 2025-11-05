# Next.js Advanced Features and Architecture Patterns

Next.js has evolved significantly with App Router, Server Components, and advanced rendering strategies. This comprehensive guide covers modern Next.js patterns, performance optimization, and architectural decisions for scalable applications.

## App Router Architecture

### 1. File-based Routing with App Directory

**App Router Structure:**
```typescript
// app directory structure
app/
├── layout.tsx                 // Root layout
├── page.tsx                  // Home page
├── loading.tsx               // Loading UI
├── error.tsx                 // Error UI
├── not-found.tsx            // 404 page
├── global-error.tsx         // Global error boundary
├── (auth)/                  // Route groups
│   ├── login/
│   │   └── page.tsx
│   └── register/
│       └── page.tsx
├── dashboard/
│   ├── layout.tsx           // Nested layout
│   ├── page.tsx            // Dashboard home
│   ├── loading.tsx         // Dashboard loading
│   ├── analytics/
│   │   └── page.tsx
│   └── settings/
│       ├── page.tsx
│       └── profile/
│           └── page.tsx
├── blog/
│   ├── page.tsx            // Blog listing
│   └── [slug]/
│       └── page.tsx        // Dynamic blog post
└── api/
    ├── users/
    │   └── route.ts        // API route handlers
    └── auth/
        └── route.ts

// Root layout (app/layout.tsx)
import { Inter } from 'next/font/google';
import { ClerkProvider } from '@clerk/nextjs';
import { ThemeProvider } from '@/components/theme-provider';
import { Toaster } from '@/components/ui/toaster';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: {
    default: 'My App',
    template: '%s | My App',
  },
  description: 'A modern Next.js application',
  keywords: ['Next.js', 'React', 'TypeScript'],
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ClerkProvider>
      <html lang="en" suppressHydrationWarning>
        <body className={inter.className}>
          <ThemeProvider
            attribute="class"
            defaultTheme="system"
            enableSystem
            disableTransitionOnChange
          >
            <div className="min-h-screen bg-background">
              {children}
            </div>
            <Toaster />
          </ThemeProvider>
        </body>
      </html>
    </ClerkProvider>
  );
}

// Nested layout (app/dashboard/layout.tsx)
import { Sidebar } from '@/components/dashboard/sidebar';
import { Header } from '@/components/dashboard/header';
import { auth } from '@clerk/nextjs';
import { redirect } from 'next/navigation';

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const { userId } = auth();
  
  if (!userId) {
    redirect('/login');
  }

  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex-1 flex flex-col overflow-hidden">
        <Header />
        <main className="flex-1 overflow-auto p-6">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### 2. Server and Client Components

**Server Components (Default):**
```typescript
// app/dashboard/analytics/page.tsx - Server Component
import { getAnalyticsData } from '@/lib/analytics';
import { ChartComponent } from '@/components/chart';
import { MetricsCards } from '@/components/metrics-cards';

// This runs on the server, has access to backend resources
export default async function AnalyticsPage() {
  // Direct database access or API calls on server
  const analyticsData = await getAnalyticsData();
  const metrics = await getMetrics();

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold">Analytics Dashboard</h1>
      
      {/* Server-rendered components */}
      <MetricsCards metrics={metrics} />
      
      {/* Client component for interactivity */}
      <ChartComponent data={analyticsData} />
    </div>
  );
}

// Async data fetching in Server Components
async function getAnalyticsData() {
  // Direct database access
  const data = await db.analytics.findMany({
    where: { date: { gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) } },
    orderBy: { date: 'desc' },
  });

  return data;
}

// lib/analytics.ts - Server-side data fetching
import { cache } from 'react';

// Cache function results across requests
export const getAnalyticsData = cache(async () => {
  const response = await fetch('https://api.analytics.com/data', {
    headers: {
      Authorization: `Bearer ${process.env.ANALYTICS_API_KEY}`,
    },
    // Next.js specific caching
    next: { 
      revalidate: 3600, // Revalidate every hour
      tags: ['analytics'] // Tag for on-demand revalidation
    },
  });

  if (!response.ok) {
    throw new Error('Failed to fetch analytics data');
  }

  return response.json();
});
```

**Client Components:**
```typescript
// components/chart.tsx - Client Component
'use client';

import { useState, useEffect } from 'react';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

interface ChartComponentProps {
  data: AnalyticsData[];
}

export function ChartComponent({ data }: ChartComponentProps) {
  const [selectedMetric, setSelectedMetric] = useState('views');
  const [chartData, setChartData] = useState(data);

  // Client-side interactivity
  const handleMetricChange = (metric: string) => {
    setSelectedMetric(metric);
    // Transform data based on selected metric
    const transformedData = data.map(item => ({
      ...item,
      value: item[metric as keyof AnalyticsData],
    }));
    setChartData(transformedData);
  };

  return (
    <div className="bg-card p-6 rounded-lg">
      <div className="flex items-center justify-between mb-4">
        <h2 className="text-xl font-semibold">Metrics Over Time</h2>
        <select
          value={selectedMetric}
          onChange={(e) => handleMetricChange(e.target.value)}
          className="border rounded px-3 py-1"
        >
          <option value="views">Page Views</option>
          <option value="users">Unique Users</option>
          <option value="sessions">Sessions</option>
        </select>
      </div>
      
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={chartData}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="date" />
          <YAxis />
          <Tooltip />
          <Line 
            type="monotone" 
            dataKey="value" 
            stroke="#8884d8" 
            strokeWidth={2}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}

// Hybrid component - Server Component that renders Client Component
// app/blog/[slug]/page.tsx
import { getBlogPost, getRelatedPosts } from '@/lib/blog';
import { CommentSection } from '@/components/comments'; // Client Component
import { ShareButtons } from '@/components/share-buttons'; // Client Component

interface BlogPostPageProps {
  params: { slug: string };
}

export default async function BlogPostPage({ params }: BlogPostPageProps) {
  // Server-side data fetching
  const [post, relatedPosts] = await Promise.all([
    getBlogPost(params.slug),
    getRelatedPosts(params.slug),
  ]);

  if (!post) {
    notFound();
  }

  return (
    <article className="max-w-4xl mx-auto px-4 py-8">
      {/* Server-rendered content */}
      <header className="mb-8">
        <h1 className="text-4xl font-bold mb-4">{post.title}</h1>
        <div className="flex items-center space-x-4 text-muted-foreground">
          <time>{post.publishedAt}</time>
          <span>{post.readingTime} min read</span>
        </div>
      </header>

      {/* Server-rendered markdown content */}
      <div 
        className="prose dark:prose-invert max-w-none"
        dangerouslySetInnerHTML={{ __html: post.content }}
      />

      {/* Client Components for interactivity */}
      <div className="mt-8 space-y-6">
        <ShareButtons url={`/blog/${params.slug}`} title={post.title} />
        <CommentSection postId={post.id} />
      </div>

      {/* Server-rendered related posts */}
      <aside className="mt-12">
        <h2 className="text-2xl font-bold mb-6">Related Posts</h2>
        <div className="grid md:grid-cols-2 lg:grid-cols-3 gap-6">
          {relatedPosts.map(relatedPost => (
            <BlogCard key={relatedPost.id} post={relatedPost} />
          ))}
        </div>
      </aside>
    </article>
  );
}
```

## Advanced Rendering Strategies

### 1. Static Site Generation (SSG) with ISR

```typescript
// Static generation with Incremental Static Regeneration
// app/products/page.tsx
import { getProducts } from '@/lib/products';
import { ProductGrid } from '@/components/product-grid';

export const revalidate = 3600; // Revalidate every hour

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div>
      <h1>Our Products</h1>
      <ProductGrid products={products} />
    </div>
  );
}

// Dynamic pages with ISR
// app/products/[id]/page.tsx
import { getProduct, getProducts } from '@/lib/products';
import { ProductDetails } from '@/components/product-details';
import { notFound } from 'next/navigation';

interface ProductPageProps {
  params: { id: string };
}

// Generate static paths at build time
export async function generateStaticParams() {
  const products = await getProducts();
  
  return products.map((product) => ({
    id: product.id,
  }));
}

// Metadata generation
export async function generateMetadata({ params }: ProductPageProps) {
  const product = await getProduct(params.id);
  
  if (!product) {
    return {
      title: 'Product Not Found',
    };
  }

  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      description: product.description,
      images: [product.image],
    },
  };
}

export default async function ProductPage({ params }: ProductPageProps) {
  const product = await getProduct(params.id);

  if (!product) {
    notFound();
  }

  return <ProductDetails product={product} />;
}

// On-demand revalidation
// app/api/revalidate/route.ts
import { revalidateTag, revalidatePath } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const secret = request.nextUrl.searchParams.get('secret');
  
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ message: 'Invalid secret' }, { status: 401 });
  }

  const body = await request.json();
  const { type, slug, tag } = body;

  try {
    if (type === 'path') {
      // Revalidate specific path
      revalidatePath(slug);
    } else if (type === 'tag') {
      // Revalidate all pages with specific tag
      revalidateTag(tag);
    }

    return NextResponse.json({ revalidated: true, now: Date.now() });
  } catch (error) {
    return NextResponse.json(
      { message: 'Error revalidating' },
      { status: 500 }
    );
  }
}

// Usage: POST /api/revalidate?secret=xxx
// Body: { "type": "path", "slug": "/products/123" }
```

### 2. Streaming and Suspense

```typescript
// Streaming with Suspense boundaries
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { UserMetrics } from '@/components/user-metrics';
import { RecentActivity } from '@/components/recent-activity';
import { QuickActions } from '@/components/quick-actions';
import { AnalyticsChart } from '@/components/analytics-chart';

export default function DashboardPage() {
  return (
    <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
      {/* Fast loading component */}
      <QuickActions />
      
      {/* Slow loading components with Suspense */}
      <Suspense fallback={<MetricsSkeleton />}>
        <UserMetrics />
      </Suspense>
      
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />
      </Suspense>
      
      <div className="md:col-span-2 lg:col-span-3">
        <Suspense fallback={<ChartSkeleton />}>
          <AnalyticsChart />
        </Suspense>
      </div>
    </div>
  );
}

// Individual components can have their own data fetching
// components/user-metrics.tsx
async function getUserMetrics() {
  // Simulate slow API call
  await new Promise(resolve => setTimeout(resolve, 2000));
  
  const response = await fetch('https://api.example.com/metrics', {
    next: { revalidate: 300 } // Cache for 5 minutes
  });
  
  return response.json();
}

export async function UserMetrics() {
  const metrics = await getUserMetrics();

  return (
    <div className="bg-card p-6 rounded-lg">
      <h2 className="text-lg font-semibold mb-4">User Metrics</h2>
      <div className="space-y-2">
        <div className="flex justify-between">
          <span>Active Users</span>
          <span className="font-bold">{metrics.activeUsers}</span>
        </div>
        <div className="flex justify-between">
          <span>New Signups</span>
          <span className="font-bold">{metrics.newSignups}</span>
        </div>
        <div className="flex justify-between">
          <span>Conversion Rate</span>
          <span className="font-bold">{metrics.conversionRate}%</span>
        </div>
      </div>
    </div>
  );
}

// Loading components
function MetricsSkeleton() {
  return (
    <div className="bg-card p-6 rounded-lg animate-pulse">
      <div className="h-4 bg-muted rounded w-24 mb-4"></div>
      <div className="space-y-2">
        {[...Array(3)].map((_, i) => (
          <div key={i} className="flex justify-between">
            <div className="h-3 bg-muted rounded w-20"></div>
            <div className="h-3 bg-muted rounded w-12"></div>
          </div>
        ))}
      </div>
    </div>
  );
}

// Custom loading UI
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
      {[...Array(6)].map((_, i) => (
        <div key={i} className="bg-card p-6 rounded-lg animate-pulse">
          <div className="h-4 bg-muted rounded w-24 mb-4"></div>
          <div className="space-y-2">
            <div className="h-3 bg-muted rounded"></div>
            <div className="h-3 bg-muted rounded w-2/3"></div>
          </div>
        </div>
      ))}
    </div>
  );
}
```

## Data Fetching Patterns

### 1. Server Actions

```typescript
// Server Actions for form handling and mutations
// app/todos/page.tsx
import { createTodo, deleteTodo, toggleTodo } from '@/lib/actions';
import { getTodos } from '@/lib/data';
import { TodoForm } from '@/components/todo-form';
import { TodoList } from '@/components/todo-list';

export default async function TodosPage() {
  const todos = await getTodos();

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Todo List</h1>
      
      <TodoForm createTodo={createTodo} />
      <TodoList 
        todos={todos} 
        onToggle={toggleTodo}
        onDelete={deleteTodo}
      />
    </div>
  );
}

// lib/actions.ts - Server Actions
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';

const CreateTodoSchema = z.object({
  title: z.string().min(1, 'Title is required'),
  description: z.string().optional(),
});

export async function createTodo(formData: FormData) {
  const validatedFields = CreateTodoSchema.safeParse({
    title: formData.get('title'),
    description: formData.get('description'),
  });

  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors,
    };
  }

  const { title, description } = validatedFields.data;

  try {
    await db.todo.create({
      data: {
        title,
        description,
        completed: false,
      },
    });
  } catch (error) {
    return {
      message: 'Database Error: Failed to create todo.',
    };
  }

  revalidatePath('/todos');
  redirect('/todos');
}

export async function toggleTodo(id: string) {
  try {
    const todo = await db.todo.findUnique({ where: { id } });
    
    if (!todo) {
      throw new Error('Todo not found');
    }

    await db.todo.update({
      where: { id },
      data: { completed: !todo.completed },
    });
  } catch (error) {
    throw new Error('Failed to toggle todo');
  }

  revalidatePath('/todos');
}

export async function deleteTodo(id: string) {
  try {
    await db.todo.delete({ where: { id } });
  } catch (error) {
    throw new Error('Failed to delete todo');
  }

  revalidatePath('/todos');
}

// Form component using Server Actions
// components/todo-form.tsx
'use client';

import { useFormState } from 'react-dom';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';

interface TodoFormProps {
  createTodo: (formData: FormData) => Promise<{ errors?: any; message?: string }>;
}

export function TodoForm({ createTodo }: TodoFormProps) {
  const [state, dispatch] = useFormState(createTodo, null);

  return (
    <form action={dispatch} className="space-y-4 mb-8">
      <div>
        <Input
          name="title"
          placeholder="Todo title"
          required
        />
        {state?.errors?.title && (
          <p className="text-sm text-destructive mt-1">
            {state.errors.title[0]}
          </p>
        )}
      </div>
      
      <div>
        <Textarea
          name="description"
          placeholder="Optional description"
          rows={3}
        />
      </div>
      
      <Button type="submit">Add Todo</Button>
      
      {state?.message && (
        <p className="text-sm text-destructive">{state.message}</p>
      )}
    </form>
  );
}

// Optimistic updates with useOptimistic
// components/todo-list.tsx
'use client';

import { useOptimistic, useTransition } from 'react';
import { Button } from '@/components/ui/button';
import { Checkbox } from '@/components/ui/checkbox';

interface TodoListProps {
  todos: Todo[];
  onToggle: (id: string) => Promise<void>;
  onDelete: (id: string) => Promise<void>;
}

export function TodoList({ todos, onToggle, onDelete }: TodoListProps) {
  const [isPending, startTransition] = useTransition();
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, action: { type: 'toggle' | 'delete'; id: string }) => {
      switch (action.type) {
        case 'toggle':
          return state.map(todo =>
            todo.id === action.id
              ? { ...todo, completed: !todo.completed }
              : todo
          );
        case 'delete':
          return state.filter(todo => todo.id !== action.id);
        default:
          return state;
      }
    }
  );

  const handleToggle = (id: string) => {
    startTransition(async () => {
      addOptimisticTodo({ type: 'toggle', id });
      await onToggle(id);
    });
  };

  const handleDelete = (id: string) => {
    startTransition(async () => {
      addOptimisticTodo({ type: 'delete', id });
      await onDelete(id);
    });
  };

  return (
    <div className="space-y-2">
      {optimisticTodos.map(todo => (
        <div
          key={todo.id}
          className={`flex items-center space-x-3 p-3 border rounded-lg ${
            todo.completed ? 'opacity-50' : ''
          }`}
        >
          <Checkbox
            checked={todo.completed}
            onCheckedChange={() => handleToggle(todo.id)}
            disabled={isPending}
          />
          <div className="flex-1">
            <h3 className={`font-medium ${todo.completed ? 'line-through' : ''}`}>
              {todo.title}
            </h3>
            {todo.description && (
              <p className="text-sm text-muted-foreground">
                {todo.description}
              </p>
            )}
          </div>
          <Button
            variant="destructive"
            size="sm"
            onClick={() => handleDelete(todo.id)}
            disabled={isPending}
          >
            Delete
          </Button>
        </div>
      ))}
    </div>
  );
}
```

### 2. Route Handlers (API Routes)

```typescript
// Modern API routes with App Router
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { auth } from '@/lib/auth';
import { z } from 'zod';

const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['user', 'admin']).optional().default('user'),
});

// GET /api/users
export async function GET(request: NextRequest) {
  try {
    const { userId } = await auth();
    
    if (!userId) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const { searchParams } = new URL(request.url);
    const page = parseInt(searchParams.get('page') || '1');
    const limit = parseInt(searchParams.get('limit') || '10');
    const search = searchParams.get('search');

    const users = await db.user.findMany({
      where: search ? {
        OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { email: { contains: search, mode: 'insensitive' } },
        ],
      } : {},
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { createdAt: 'desc' },
    });

    const total = await db.user.count();

    return NextResponse.json({
      users,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit),
      },
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}

// POST /api/users
export async function POST(request: NextRequest) {
  try {
    const { userId } = await auth();
    
    if (!userId) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const body = await request.json();
    const validatedData = CreateUserSchema.safeParse(body);

    if (!validatedData.success) {
      return NextResponse.json(
        { error: 'Validation failed', details: validatedData.error.issues },
        { status: 400 }
      );
    }

    const { name, email, role } = validatedData.data;

    // Check if user already exists
    const existingUser = await db.user.findUnique({ where: { email } });
    
    if (existingUser) {
      return NextResponse.json(
        { error: 'User already exists' },
        { status: 409 }
      );
    }

    const user = await db.user.create({
      data: { name, email, role },
      select: { id: true, name: true, email: true, role: true, createdAt: true },
    });

    return NextResponse.json(user, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}

// Dynamic route: app/api/users/[id]/route.ts
interface RouteParams {
  params: { id: string };
}

export async function GET(request: NextRequest, { params }: RouteParams) {
  try {
    const { userId } = await auth();
    
    if (!userId) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const user = await db.user.findUnique({
      where: { id: params.id },
      select: { id: true, name: true, email: true, role: true, createdAt: true },
    });

    if (!user) {
      return NextResponse.json({ error: 'User not found' }, { status: 404 });
    }

    return NextResponse.json(user);
  } catch (error) {
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}

export async function PATCH(request: NextRequest, { params }: RouteParams) {
  try {
    const { userId } = await auth();
    
    if (!userId) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const body = await request.json();
    const validatedData = CreateUserSchema.partial().safeParse(body);

    if (!validatedData.success) {
      return NextResponse.json(
        { error: 'Validation failed', details: validatedData.error.issues },
        { status: 400 }
      );
    }

    const user = await db.user.update({
      where: { id: params.id },
      data: validatedData.data,
      select: { id: true, name: true, email: true, role: true, updatedAt: true },
    });

    return NextResponse.json(user);
  } catch (error) {
    if (error.code === 'P2025') {
      return NextResponse.json({ error: 'User not found' }, { status: 404 });
    }
    
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}

export async function DELETE(request: NextRequest, { params }: RouteParams) {
  try {
    const { userId } = await auth();
    
    if (!userId) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    await db.user.delete({
      where: { id: params.id },
    });

    return NextResponse.json({ message: 'User deleted successfully' });
  } catch (error) {
    if (error.code === 'P2025') {
      return NextResponse.json({ error: 'User not found' }, { status: 404 });
    }
    
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}
```

## Performance Optimization

### 1. Image Optimization

```typescript
// Next.js Image component with optimizations
import Image from 'next/image';

// Basic usage
export function ProductImage({ product }: { product: Product }) {
  return (
    <Image
      src={product.imageUrl}
      alt={product.name}
      width={400}
      height={300}
      priority={product.featured} // Load above-the-fold images immediately
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD..." // Low-quality placeholder
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      className="rounded-lg object-cover"
    />
  );
}

// Responsive images with different breakpoints
export function HeroImage() {
  return (
    <Image
      src="/hero-image.jpg"
      alt="Hero image"
      fill
      priority
      sizes="100vw"
      style={{
        objectFit: 'cover',
      }}
    />
  );
}

// Gallery with optimized loading
export function ImageGallery({ images }: { images: string[] }) {
  return (
    <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
      {images.map((image, index) => (
        <div key={index} className="relative aspect-square">
          <Image
            src={image}
            alt={`Gallery image ${index + 1}`}
            fill
            sizes="(max-width: 768px) 50vw, (max-width: 1200px) 33vw, 25vw"
            className="object-cover rounded-lg"
            loading={index < 4 ? 'eager' : 'lazy'} // Load first 4 images immediately
          />
        </div>
      ))}
    </div>
  );
}

// Custom image loader for external CDNs
const cloudinaryLoader = ({ src, width, quality }: any) => {
  const params = ['f_auto', 'c_limit', `w_${width}`, `q_${quality || 'auto'}`];
  return `https://res.cloudinary.com/demo/image/fetch/${params.join(',')}/${src}`;
};

export function CloudinaryImage({ src, alt, ...props }: any) {
  return (
    <Image
      loader={cloudinaryLoader}
      src={src}
      alt={alt}
      {...props}
    />
  );
}
```

### 2. Font Optimization

```typescript
// next/font optimization
import { Inter, Roboto_Mono, Playfair_Display } from 'next/font/google';
import localFont from 'next/font/local';

// Primary font
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
});

// Monospace font for code
const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-mono',
});

// Display font for headings
const playfair = Playfair_Display({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-display',
});

// Local custom font
const customFont = localFont({
  src: [
    {
      path: './fonts/CustomFont-Regular.woff2',
      weight: '400',
      style: 'normal',
    },
    {
      path: './fonts/CustomFont-Bold.woff2',
      weight: '700',
      style: 'normal',
    },
  ],
  variable: '--font-custom',
  display: 'swap',
});

// Usage in layout
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html 
      lang="en" 
      className={`${inter.variable} ${robotoMono.variable} ${playfair.variable} ${customFont.variable}`}
    >
      <body className={inter.className}>
        {children}
      </body>
    </html>
  );
}

// Tailwind CSS configuration for font variables
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      fontFamily: {
        sans: ['var(--font-inter)'],
        mono: ['var(--font-mono)'],
        display: ['var(--font-display)'],
        custom: ['var(--font-custom)'],
      },
    },
  },
};

// Usage in components
export function Typography() {
  return (
    <div>
      <h1 className="font-display text-4xl font-bold">Display Heading</h1>
      <p className="font-sans text-base">Body text in Inter</p>
      <code className="font-mono text-sm">Code in Roboto Mono</code>
      <span className="font-custom">Custom brand font</span>
    </div>
  );
}
```

### 3. Bundle Analysis and Optimization

```typescript
// next.config.js with bundle analysis
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

/** @type {import('next').NextConfig} */
const nextConfig = {
  // Experimental features
  experimental: {
    appDir: true,
    serverActions: true,
    serverComponentsExternalPackages: ['prisma'],
  },

  // Compression
  compress: true,

  // Image domains
  images: {
    domains: ['example.com', 'res.cloudinary.com'],
    formats: ['image/webp', 'image/avif'],
  },

  // Headers for security and caching
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'Referrer-Policy',
            value: 'origin-when-cross-origin',
          },
        ],
      },
      {
        source: '/api/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'no-cache, no-store, must-revalidate',
          },
        ],
      },
    ];
  },

  // Redirects
  async redirects() {
    return [
      {
        source: '/old-page',
        destination: '/new-page',
        permanent: true,
      },
    ];
  },

  // Webpack configuration
  webpack: (config, { buildId, dev, isServer, defaultLoaders, webpack }) => {
    // Custom webpack configurations
    config.module.rules.push({
      test: /\.svg$/,
      use: ['@svgr/webpack'],
    });

    // Reduce bundle size
    if (!dev && !isServer) {
      config.optimization.splitChunks.chunks = 'all';
      config.optimization.splitChunks.cacheGroups = {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      };
    }

    return config;
  },
};

module.exports = withBundleAnalyzer(nextConfig);

// Package.json scripts for bundle analysis
{
  "scripts": {
    "analyze": "ANALYZE=true npm run build",
    "build:analyze": "npm run build && npx @next/bundle-analyzer"
  }
}

// Dynamic imports for code splitting
import dynamic from 'next/dynamic';

// Lazy load heavy components
const HeavyChart = dynamic(() => import('@/components/heavy-chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // Disable server-side rendering for this component
});

const VideoPlayer = dynamic(() => import('@/components/video-player'), {
  loading: () => <div>Loading video...</div>,
});

// Conditional imports
const AdminPanel = dynamic(() => import('@/components/admin-panel'), {
  loading: () => <div>Loading admin panel...</div>,
});

export function Dashboard({ user }: { user: User }) {
  return (
    <div>
      <h1>Dashboard</h1>
      
      {/* Always rendered */}
      <QuickStats />
      
      {/* Lazy loaded */}
      <HeavyChart data={data} />
      
      {/* Conditionally rendered and lazy loaded */}
      {user.role === 'admin' && <AdminPanel />}
    </div>
  );
}
```

## Interview-Ready Summary

**Next.js App Router provides:**

1. **File-based Routing** - Nested layouts, loading states, error boundaries, route groups
2. **Server Components** - Default server rendering, direct database access, automatic caching
3. **Client Components** - Explicit 'use client' directive for interactivity and browser APIs
4. **Advanced Rendering** - SSG with ISR, streaming with Suspense, on-demand revalidation
5. **Data Fetching** - Server Actions for mutations, Route Handlers for APIs, optimistic updates

**Key patterns:**
- **Composition over configuration** - Layouts and components naturally compose
- **Progressive enhancement** - Server Components by default, Client Components when needed
- **Streaming architecture** - Render and stream components as data becomes available
- **Cache optimization** - Multiple caching layers with granular control

**Performance optimizations:** Image optimization with next/image, font optimization with next/font, bundle splitting with dynamic imports, comprehensive caching strategies.

**Migration strategy:** Can be adopted incrementally alongside Pages Router, provides clear upgrade path from traditional React SPAs.