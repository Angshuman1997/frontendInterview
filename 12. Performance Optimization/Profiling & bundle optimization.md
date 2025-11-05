# Profiling & bundle optimization

This file covers techniques for analyzing and optimizing React app performance and bundle size.
Essential for identifying bottlenecks and maintaining fast, efficient applications.

---

## Why profiling matters

Profiling helps you:

* Identify actual performance bottlenecks (not assumptions)
* Measure the impact of optimizations
* Prevent performance regressions
* Optimize for real user experiences
* Make data-driven decisions about code changes
* Understand bundle composition and size

"Premature optimization is the root of all evil" - Donald Knuth. Profile first, optimize second.

---

## React DevTools Profiler

### Basic profiling workflow

```jsx
function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <Header />
      <MainContent />
      <Footer />
    </Profiler>
  );
}

function onRenderCallback(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  console.log('Component:', id);
  console.log('Phase:', phase); // 'mount' or 'update'
  console.log('Actual duration:', actualDuration); // Time spent rendering
  console.log('Base duration:', baseDuration); // Estimated time without memoization
}
```

### Using React DevTools extension

1. **Install React DevTools** browser extension
2. **Open DevTools** → React → Profiler tab
3. **Click record** button (red circle)
4. **Interact** with your app (clicks, typing, navigation)
5. **Stop recording** and analyze results

### Interpreting profiler results

```jsx
// Example of component that would show in profiler
function ExpensiveComponent({ items }) {
  // This will show high render time in profiler
  const processedItems = items.map(item => ({
    ...item,
    processed: heavyComputation(item) // Expensive operation
  }));

  return (
    <div>
      {processedItems.map(item => (
        <div key={item.id}>{item.processed}</div>
      ))}
    </div>
  );
}

// Optimized version
function OptimizedComponent({ items }) {
  const processedItems = useMemo(() => 
    items.map(item => ({
      ...item,
      processed: heavyComputation(item)
    })), 
    [items]
  );

  return (
    <div>
      {processedItems.map(item => (
        <div key={item.id}>{item.processed}</div>
      ))}
    </div>
  );
}
```

---

## Browser Performance Tab

### Recording performance

```js
// Programmatic performance measurement
function measurePerformance() {
  performance.mark('start');
  
  // Your code here
  performExpensiveOperation();
  
  performance.mark('end');
  performance.measure('operation', 'start', 'end');
  
  const measure = performance.getEntriesByName('operation')[0];
  console.log(`Operation took ${measure.duration}ms`);
}
```

### Key metrics to monitor

```js
// Core Web Vitals measurement
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function setupPerformanceMonitoring() {
  getCLS(console.log); // Cumulative Layout Shift
  getFID(console.log); // First Input Delay
  getFCP(console.log); // First Contentful Paint
  getLCP(console.log); // Largest Contentful Paint
  getTTFB(console.log); // Time to First Byte
}
```

### Memory profiling

```js
// Monitor memory usage
function checkMemoryUsage() {
  if ('memory' in performance) {
    const memory = performance.memory;
    console.log('Used JS Heap Size:', memory.usedJSHeapSize / 1048576, 'MB');
    console.log('Total JS Heap Size:', memory.totalJSHeapSize / 1048576, 'MB');
    console.log('JS Heap Size Limit:', memory.jsHeapSizeLimit / 1048576, 'MB');
  }
}

// Check for memory leaks
setInterval(checkMemoryUsage, 5000);
```

---

## Bundle analysis tools

### Webpack Bundle Analyzer

```bash
npm install --save-dev webpack-bundle-analyzer
```

```js
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    })
  ]
};
```

```json
// package.json scripts
{
  "scripts": {
    "analyze": "npm run build && npx webpack-bundle-analyzer build/static/js/*.js"
  }
}
```

### Source Map Explorer

```bash
npm install --save-dev source-map-explorer
```

```json
{
  "scripts": {
    "analyze": "npm run build && npx source-map-explorer 'build/static/js/*.js'"
  }
}
```

### Bundle Buddy

```bash
# Generate webpack stats
npx webpack --profile --json > stats.json

# Upload to https://bundle-buddy.com/webpack
```

---

## Performance monitoring in production

### Real User Monitoring (RUM)

```js
// Custom performance tracking
class PerformanceTracker {
  constructor() {
    this.metrics = {};
    this.setupObservers();
  }

  setupObservers() {
    // Long task observer
    if ('PerformanceObserver' in window) {
      const longTaskObserver = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          console.warn('Long task detected:', entry.duration, 'ms');
          this.trackMetric('longTask', entry.duration);
        }
      });
      longTaskObserver.observe({ entryTypes: ['longtask'] });

      // Layout shift observer
      const clsObserver = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          if (!entry.hadRecentInput) {
            this.trackMetric('cls', entry.value);
          }
        }
      });
      clsObserver.observe({ entryTypes: ['layout-shift'] });
    }
  }

  trackMetric(name, value) {
    this.metrics[name] = this.metrics[name] || [];
    this.metrics[name].push({
      value,
      timestamp: performance.now()
    });
    
    // Send to analytics service
    this.sendToAnalytics(name, value);
  }

  sendToAnalytics(metric, value) {
    // Send to your analytics service
    fetch('/api/metrics', {
      method: 'POST',
      body: JSON.stringify({ metric, value, timestamp: Date.now() })
    });
  }
}

const tracker = new PerformanceTracker();
```

### Error boundary with performance tracking

```jsx
class PerformanceErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // Track performance when errors occur
    const performanceEntries = performance.getEntriesByType('navigation');
    const timing = performanceEntries[0];
    
    console.error('Error with performance context:', {
      error: error.message,
      loadTime: timing.loadEventEnd - timing.loadEventStart,
      domContentLoaded: timing.domContentLoadedEventEnd - timing.domContentLoadedEventStart,
      componentStack: errorInfo.componentStack
    });
  }

  render() {
    if (this.state.hasError) {
      return <div>Something went wrong. Please refresh the page.</div>;
    }

    return this.props.children;
  }
}
```

---

## React performance optimization patterns

### Component performance audit

```jsx
// Higher-order component for performance tracking
function withPerformanceTracking(WrappedComponent, componentName) {
  return function PerformanceTrackedComponent(props) {
    const renderStart = useRef();
    const renderCount = useRef(0);

    renderStart.current = performance.now();
    renderCount.current += 1;

    useLayoutEffect(() => {
      const renderTime = performance.now() - renderStart.current;
      console.log(`${componentName} render #${renderCount.current}: ${renderTime.toFixed(2)}ms`);
      
      if (renderTime > 16) { // More than one frame
        console.warn(`${componentName} slow render detected!`);
      }
    });

    return <WrappedComponent {...props} />;
  };
}

// Usage
const TrackedComponent = withPerformanceTracking(MyComponent, 'MyComponent');
```

### Render optimization checklist

```jsx
function OptimizedComponent({ items, filter, onItemClick }) {
  // ✅ Memoize expensive computations
  const filteredItems = useMemo(() => 
    items.filter(item => item.name.includes(filter)), 
    [items, filter]
  );

  // ✅ Stable callback references
  const handleItemClick = useCallback((id) => {
    onItemClick(id);
  }, [onItemClick]);

  // ✅ Memoize component rendering
  const ItemList = useMemo(() => 
    filteredItems.map(item => (
      <Item 
        key={item.id} 
        item={item} 
        onClick={handleItemClick} 
      />
    )), 
    [filteredItems, handleItemClick]
  );

  return <div>{ItemList}</div>;
}

// ✅ Memoize child components
const Item = React.memo(function Item({ item, onClick }) {
  return (
    <div onClick={() => onClick(item.id)}>
      {item.name}
    </div>
  );
});
```

---

## Bundle optimization strategies

### Code splitting analysis

```js
// Analyze chunk sizes
const ChunkAnalyzer = {
  analyzeChunks() {
    const chunks = performance.getEntriesByType('navigation');
    const resources = performance.getEntriesByType('resource');
    
    const jsChunks = resources.filter(resource => 
      resource.name.includes('.js') && 
      resource.name.includes('chunk')
    );

    jsChunks.forEach(chunk => {
      console.log(`Chunk: ${chunk.name}`);
      console.log(`Size: ${chunk.transferSize} bytes`);
      console.log(`Load time: ${chunk.responseEnd - chunk.requestStart}ms`);
    });
  },

  trackChunkLoading() {
    // Track dynamic imports
    const originalImport = window.__webpack_require__.e;
    window.__webpack_require__.e = function(chunkId) {
      const start = performance.now();
      return originalImport.call(this, chunkId).then(
        (result) => {
          const loadTime = performance.now() - start;
          console.log(`Chunk ${chunkId} loaded in ${loadTime}ms`);
          return result;
        }
      );
    };
  }
};
```

### Dependency analysis

```js
// package.json analyzer script
const fs = require('fs');
const path = require('path');

function analyzeDependencies() {
  const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf8'));
  const { dependencies, devDependencies } = packageJson;
  
  const allDeps = { ...dependencies, ...devDependencies };
  
  // Find large dependencies
  const largeDeps = Object.keys(allDeps).filter(dep => {
    try {
      const depPackageJson = require(`${dep}/package.json`);
      const depPath = require.resolve(dep);
      const stats = fs.statSync(depPath);
      return stats.size > 100000; // > 100KB
    } catch (e) {
      return false;
    }
  });
  
  console.log('Large dependencies:', largeDeps);
}
```

---

## Advanced profiling techniques

### Custom performance hooks

```jsx
// Hook for measuring component performance
function usePerformanceMonitor(componentName) {
  const renderCount = useRef(0);
  const totalRenderTime = useRef(0);
  const lastRenderTime = useRef(0);

  useLayoutEffect(() => {
    const renderTime = performance.now() - lastRenderTime.current;
    renderCount.current += 1;
    totalRenderTime.current += renderTime;
    
    const averageRenderTime = totalRenderTime.current / renderCount.current;
    
    if (renderTime > 16) {
      console.warn(`${componentName} slow render: ${renderTime.toFixed(2)}ms`);
    }
    
    if (renderCount.current % 10 === 0) {
      console.log(`${componentName} average render time: ${averageRenderTime.toFixed(2)}ms`);
    }
  });

  lastRenderTime.current = performance.now();
}

// Usage
function MyComponent({ data }) {
  usePerformanceMonitor('MyComponent');
  
  return <div>{data.map(item => <Item key={item.id} {...item} />)}</div>;
}
```

### Network performance tracking

```js
// Monitor API call performance
class APIPerformanceTracker {
  constructor() {
    this.interceptFetch();
  }

  interceptFetch() {
    const originalFetch = window.fetch;
    
    window.fetch = async (...args) => {
      const start = performance.now();
      const url = args[0];
      
      try {
        const response = await originalFetch(...args);
        const duration = performance.now() - start;
        
        this.trackAPICall(url, duration, response.status, 'success');
        return response;
      } catch (error) {
        const duration = performance.now() - start;
        this.trackAPICall(url, duration, 0, 'error');
        throw error;
      }
    };
  }

  trackAPICall(url, duration, status, result) {
    console.log(`API Call: ${url}`);
    console.log(`Duration: ${duration.toFixed(2)}ms`);
    console.log(`Status: ${status}`);
    console.log(`Result: ${result}`);
    
    if (duration > 1000) {
      console.warn('Slow API call detected!');
    }
  }
}

new APIPerformanceTracker();
```

---

## Performance budgets and CI/CD

### Lighthouse CI configuration

```js
// lighthouse-ci.js
module.exports = {
  ci: {
    collect: {
      startServerCommand: 'npm run serve',
      url: ['http://localhost:3000'],
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.8 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

### Bundle size monitoring

```js
// scripts/check-bundle-size.js
const fs = require('fs');
const path = require('path');

const BUNDLE_SIZE_LIMIT = 500 * 1024; // 500KB

function checkBundleSize() {
  const buildPath = path.join(__dirname, '../build/static/js');
  const files = fs.readdirSync(buildPath);
  
  const mainBundle = files.find(file => file.startsWith('main.') && file.endsWith('.js'));
  
  if (mainBundle) {
    const bundlePath = path.join(buildPath, mainBundle);
    const stats = fs.statSync(bundlePath);
    
    console.log(`Main bundle size: ${(stats.size / 1024).toFixed(2)} KB`);
    
    if (stats.size > BUNDLE_SIZE_LIMIT) {
      console.error(`Bundle size exceeds limit of ${(BUNDLE_SIZE_LIMIT / 1024).toFixed(2)} KB`);
      process.exit(1);
    }
  }
}

checkBundleSize();
```

---

## Best practices checklist

* [ ] Set up React DevTools Profiler for regular performance auditing
* [ ] Configure webpack-bundle-analyzer for bundle composition analysis
* [ ] Implement Core Web Vitals monitoring in production
* [ ] Set performance budgets in CI/CD pipeline
* [ ] Profile before and after major changes
* [ ] Monitor bundle size growth over time
* [ ] Track long tasks and layout shifts
* [ ] Use Performance API for custom metrics
* [ ] Profile on low-end devices and slow networks
* [ ] Implement error tracking with performance context

---

## Interview-ready summary

Profiling identifies real performance bottlenecks through React DevTools Profiler, browser Performance tab, and production monitoring. Bundle optimization uses webpack-bundle-analyzer and source-map-explorer to identify large dependencies and optimize code splitting. Key metrics include Core Web Vitals (LCP, FID, CLS), render times, and bundle sizes. Implement performance budgets in CI/CD, track long tasks with PerformanceObserver, and use Real User Monitoring for production insights. Always profile before optimizing and measure the impact of changes.
