# How would you improve initial load time

This file covers practical strategies to optimize initial page load time.
Keep it handy for interviews and as a practical guide when optimizing web applications.

---

## Why it matters

Initial load time directly impacts:

* User experience and perceived performance
* SEO rankings (Core Web Vitals)
* Conversion rates and business metrics
* Mobile users on slow networks
* Search engine crawling efficiency

---

## High-level strategies

1. **Reduce resource size** through compression, minification, and optimization.
2. **Minimize critical path** by prioritizing above-the-fold content.
3. **Leverage caching** at multiple layers (browser, CDN, service worker).
4. **Optimize network requests** by reducing count and improving delivery.
5. **Defer non-critical resources** to avoid blocking initial render.
6. **Use modern formats** and delivery methods for better compression.

---

## Techniques & patterns

### 1. Bundle optimization

Minimize and compress JavaScript and CSS bundles.

```js
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
  },
};
```

* Use tree shaking to eliminate dead code
* Split bundles (vendor, app, route-specific)
* Enable Gzip/Brotli compression on server
* Use webpack-bundle-analyzer to identify large dependencies

---

### 2. Code splitting and lazy loading

Load only what's needed for initial render.

```jsx
// Route-based code splitting
const HomePage = lazy(() => import('./pages/HomePage'));
const ProfilePage = lazy(() => import('./pages/ProfilePage'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/profile" element={<ProfilePage />} />
      </Routes>
    </Suspense>
  );
}

// Component-based lazy loading
const HeavyChart = lazy(() => import('./HeavyChart'));
```

---

### 3. Image optimization

Images often account for 60-70% of page weight.

```html
<!-- Modern formats with fallbacks -->
<picture>
  <source srcset="hero.webp" type="image/webp">
  <source srcset="hero.avif" type="image/avif">
  <img src="hero.jpg" alt="Hero image" loading="lazy">
</picture>

<!-- Responsive images -->
<img srcset="small.jpg 480w, medium.jpg 800w, large.jpg 1200w"
     sizes="(max-width: 480px) 480px, (max-width: 800px) 800px, 1200px"
     src="medium.jpg" alt="Responsive image">
```

* Use WebP/AVIF formats (smaller file sizes)
* Implement lazy loading for below-the-fold images
* Serve responsive images based on device/viewport
* Optimize image dimensions (don't serve 4K for mobile)

---

### 4. Critical CSS and resource hints

Inline critical CSS and defer non-critical styles.

```html
<!-- Inline critical CSS for above-the-fold -->
<style>
  .hero { background: #f0f0f0; height: 100vh; }
  .nav { position: fixed; top: 0; }
</style>

<!-- Preload critical resources -->
<link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/hero.webp" as="image">

<!-- Defer non-critical CSS -->
<link rel="preload" href="/styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
```

---

### 5. Font optimization

Fonts can block rendering if not optimized.

```css
/* Use font-display for faster text rendering */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap; /* Shows fallback font while loading */
}

/* Preload critical fonts */
```

```html
<link rel="preload" href="/fonts/critical.woff2" as="font" type="font/woff2" crossorigin>
```

* Use `font-display: swap` to avoid invisible text
* Preload critical fonts
* Subset fonts to include only needed characters
* Use system fonts when possible for instant rendering

---

### 6. Server-Side Rendering (SSR) / Static Generation

Pre-render content on the server for faster initial paint.

```jsx
// Next.js static generation
export async function getStaticProps() {
  const data = await fetchData();
  return { props: { data } };
}

// Next.js server-side rendering
export async function getServerSideProps() {
  const data = await fetchData();
  return { props: { data } };
}
```

* Use SSG for static content (blogs, marketing pages)
* Use SSR for dynamic, personalized content
* Implement incremental static regeneration (ISR) for hybrid approach
* Consider edge-side rendering for global performance

---

### 7. CDN and caching strategies

Serve static assets from geographically distributed servers.

```js
// Service Worker caching strategy
self.addEventListener('fetch', event => {
  if (event.request.destination === 'image') {
    event.respondWith(
      caches.open('images').then(cache =>
        cache.match(event.request).then(response =>
          response || fetch(event.request).then(fetchResponse => {
            cache.put(event.request, fetchResponse.clone());
            return fetchResponse;
          })
        )
      )
    );
  }
});
```

* Use CDN for static assets (images, CSS, JS)
* Implement proper cache headers (`Cache-Control`, `ETag`)
* Use service workers for offline-first caching
* Leverage browser cache with versioned assets

---

### 8. Third-party script optimization

Third-party scripts can significantly impact load time.

```html
<!-- Defer non-critical scripts -->
<script src="/analytics.js" defer></script>

<!-- Async load for independent scripts -->
<script src="/chat-widget.js" async></script>

<!-- Lazy load heavy widgets -->
<script>
  // Load script only when needed
  function loadChatWidget() {
    const script = document.createElement('script');
    script.src = '/chat.js';
    document.head.appendChild(script);
  }
  
  // Trigger on user interaction
  document.addEventListener('scroll', loadChatWidget, { once: true });
</script>
```

---

### 9. API and data optimization

Optimize data fetching patterns.

```jsx
// Fetch critical data early
function App() {
  const { data: criticalData } = useSWR('/api/critical', { suspense: true });
  const { data: secondaryData } = useSWR('/api/secondary'); // No suspense
  
  return (
    <div>
      <CriticalComponent data={criticalData} />
      {secondaryData && <SecondaryComponent data={secondaryData} />}
    </div>
  );
}

// GraphQL query optimization
const HOMEPAGE_QUERY = gql`
  query Homepage {
    hero { title, image }
    featuredPosts(limit: 3) { title, excerpt }
  }
`;
```

* Fetch only necessary data for initial render
* Use GraphQL to avoid over-fetching
* Implement proper error boundaries and loading states
* Consider streaming responses for large datasets

---

### 10. Modern build tools and techniques

Use tools that optimize for performance by default.

```js
// Vite config for optimization
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash', 'date-fns'],
        },
      },
    },
  },
};
```

* Use Vite or modern bundlers for faster builds
* Enable module preloading for critical chunks
* Use import() for dynamic imports and code splitting
* Leverage HTTP/2 push for critical resources

---

## Performance budgets

Set measurable targets to maintain performance.

```js
// lighthouse-ci.js
module.exports = {
  ci: {
    assert: {
      assertions: {
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'speed-index': ['error', { maxNumericValue: 3000 }],
      },
    },
  },
};
```

* Set FCP target < 1.8s, LCP < 2.5s, CLS < 0.1
* Monitor bundle size (aim for < 200KB initial JS)
* Track Core Web Vitals in production
* Use automated performance testing in CI/CD

---

## Quick wins checklist

* [ ] Enable Gzip/Brotli compression
* [ ] Optimize images (WebP, lazy loading, responsive)
* [ ] Defer non-critical JavaScript
* [ ] Inline critical CSS
* [ ] Preload critical resources
* [ ] Use CDN for static assets
* [ ] Implement proper caching headers
* [ ] Remove unused CSS/JS (tree shaking)
* [ ] Split bundles (vendor, app, routes)
* [ ] Use modern image formats and font-display: swap

---

## Measuring and monitoring

```js
// Core Web Vitals measurement
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

getCLS(console.log);
getFID(console.log);
getFCP(console.log);
getLCP(console.log);
getTTFB(console.log);
```

* Use Lighthouse for auditing
* Monitor Real User Metrics (RUM) with tools like web-vitals
* Set up performance monitoring (PageSpeed Insights API)
* Track performance regressions in CI/CD

---

## Common pitfalls to avoid

* Loading all JavaScript upfront instead of code splitting
* Not optimizing images (largest contributor to slow loads)
* Blocking render with synchronous scripts
* Not leveraging browser caching
* Over-relying on client-side rendering for static content
* Loading third-party scripts without considering impact
* Not measuring actual user experience (only synthetic tests)

---

## Example: comprehensive optimization

```jsx
// app.js - Optimized application structure
import { lazy, Suspense } from 'react';
import { preloadRoute } from './utils/preloader';

// Code splitting with preloading
const HomePage = lazy(() => import('./pages/HomePage'));
const ProductPage = lazy(() => import('./pages/ProductPage'));

function App() {
  // Preload routes on hover/focus
  const handleLinkHover = (route) => preloadRoute(route);
  
  return (
    <div className="app">
      {/* Critical CSS inlined */}
      <header className="header">
        <nav>
          <a href="/" onMouseEnter={() => handleLinkHover('home')}>Home</a>
          <a href="/products" onMouseEnter={() => handleLinkHover('products')}>Products</a>
        </nav>
      </header>
      
      <main>
        <Suspense fallback={<div className="loading">Loading...</div>}>
          <Routes>
            <Route path="/" element={<HomePage />} />
            <Route path="/products" element={<ProductPage />} />
          </Routes>
        </Suspense>
      </main>
    </div>
  );
}
```

```html
<!-- index.html - Optimized HTML -->
<!DOCTYPE html>
<html lang="en">
<head>
  <!-- Critical CSS inlined -->
  <style>/* Critical styles here */</style>
  
  <!-- Preload critical resources -->
  <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="/hero.webp" as="image">
  
  <!-- Defer non-critical CSS -->
  <link rel="preload" href="/styles.css" as="style" onload="this.rel='stylesheet'">
</head>
<body>
  <div id="root"></div>
  <!-- Main bundle with defer -->
  <script src="/main.js" defer></script>
</body>
</html>
```

---

## Interview-ready answer

To improve initial load time: optimize bundle size through code splitting and tree shaking, implement lazy loading for non-critical resources, optimize images with modern formats and responsive loading, use SSR/SSG for faster first paint, leverage CDN and proper caching strategies, defer third-party scripts, inline critical CSS, and preload essential resources. Set performance budgets, monitor Core Web Vitals, and measure real user metrics to maintain optimizations over time.
