# Service Workers, Web Workers, and Background Processing

Service Workers and Web Workers enable powerful background processing capabilities in web applications. This guide covers implementation strategies for offline functionality, background sync, push notifications, and performance optimization through background processing.

## Service Workers - Offline and Caching Strategies

### 1. Service Worker Fundamentals and Lifecycle
```typescript
// Service Worker Registration and Management
interface ServiceWorkerConfig {
  swPath: string;
  scope?: string;
  updateViaCache?: 'imports' | 'all' | 'none';
  enableSkipWaiting?: boolean;
  enableClientsClaim?: boolean;
}

class ServiceWorkerManager {
  private registration: ServiceWorkerRegistration | null = null;
  private config: ServiceWorkerConfig;
  private updateListeners = new Set<Function>();

  constructor(config: ServiceWorkerConfig) {
    this.config = config;
  }

  // Register service worker
  async register(): Promise<ServiceWorkerRegistration> {
    if (!('serviceWorker' in navigator)) {
      throw new Error('Service Worker not supported');
    }

    try {
      this.registration = await navigator.serviceWorker.register(
        this.config.swPath,
        {
          scope: this.config.scope || '/',
          updateViaCache: this.config.updateViaCache || 'imports',
        }
      );

      this.setupEventListeners();
      this.checkForUpdates();

      console.log('Service Worker registered:', this.registration.scope);
      return this.registration;
    } catch (error) {
      console.error('Service Worker registration failed:', error);
      throw error;
    }
  }

  private setupEventListeners(): void {
    if (!this.registration) return;

    // Listen for updates
    this.registration.addEventListener('updatefound', () => {
      const newWorker = this.registration!.installing;
      
      if (newWorker) {
        this.handleServiceWorkerUpdate(newWorker);
      }
    });

    // Listen for controller changes
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      this.notifyUpdateListeners('controllerchange');
    });

    // Listen for messages from service worker
    navigator.serviceWorker.addEventListener('message', (event) => {
      this.handleServiceWorkerMessage(event);
    });
  }

  private handleServiceWorkerUpdate(newWorker: ServiceWorker): void {
    newWorker.addEventListener('statechange', () => {
      if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
        // New service worker installed, update available
        this.notifyUpdateListeners('updateavailable', newWorker);
      }
    });
  }

  private handleServiceWorkerMessage(event: MessageEvent): void {
    const { type, payload } = event.data;

    switch (type) {
      case 'CACHE_UPDATED':
        this.notifyUpdateListeners('cacheupdate', payload);
        break;
      case 'OFFLINE_STATUS':
        this.notifyUpdateListeners('offlinestatus', payload);
        break;
      case 'SYNC_COMPLETE':
        this.notifyUpdateListeners('synccomplete', payload);
        break;
    }
  }

  // Check for service worker updates
  async checkForUpdates(): Promise<void> {
    if (!this.registration) return;

    try {
      await this.registration.update();
    } catch (error) {
      console.error('Service Worker update check failed:', error);
    }
  }

  // Skip waiting and activate new service worker
  async skipWaiting(): Promise<void> {
    if (!this.registration?.waiting) return;

    // Send skip waiting message to service worker
    this.registration.waiting.postMessage({ type: 'SKIP_WAITING' });
  }

  // Send message to service worker
  sendMessage(message: any): void {
    if (!navigator.serviceWorker.controller) {
      console.warn('No service worker controller available');
      return;
    }

    navigator.serviceWorker.controller.postMessage(message);
  }

  // Add update listener
  onUpdate(callback: (event: string, payload?: any) => void): () => void {
    this.updateListeners.add(callback);
    
    return () => {
      this.updateListeners.delete(callback);
    };
  }

  private notifyUpdateListeners(event: string, payload?: any): void {
    this.updateListeners.forEach(callback => {
      try {
        callback(event, payload);
      } catch (error) {
        console.error('Error in service worker update listener:', error);
      }
    });
  }

  // Unregister service worker
  async unregister(): Promise<boolean> {
    if (!this.registration) {
      return false;
    }

    try {
      const result = await this.registration.unregister();
      this.registration = null;
      return result;
    } catch (error) {
      console.error('Service Worker unregistration failed:', error);
      return false;
    }
  }
}

// Service Worker Implementation (sw.js)
const SW_VERSION = 'v1.0.0';
const CACHE_NAME = `app-cache-${SW_VERSION}`;
const STATIC_CACHE = `static-cache-${SW_VERSION}`;
const DYNAMIC_CACHE = `dynamic-cache-${SW_VERSION}`;

// Resources to cache immediately
const STATIC_RESOURCES = [
  '/',
  '/index.html',
  '/manifest.json',
  '/css/main.css',
  '/js/main.js',
  '/offline.html',
];

// Network-first resources
const NETWORK_FIRST_PATTERNS = [
  /\/api\//,
  /\/auth\//,
];

// Cache-first resources
const CACHE_FIRST_PATTERNS = [
  /\.(?:png|gif|jpg|jpeg|svg|webp)$/,
  /\.(?:css|js)$/,
  /\/fonts\//,
];

// Service Worker Event Handlers
self.addEventListener('install', (event: ExtendableEvent) => {
  console.log(`Service Worker ${SW_VERSION} installing...`);

  event.waitUntil(
    (async () => {
      try {
        // Cache static resources
        const staticCache = await caches.open(STATIC_CACHE);
        await staticCache.addAll(STATIC_RESOURCES);

        console.log('Static resources cached');

        // Skip waiting if configured
        if (self.skipWaiting) {
          await self.skipWaiting();
        }
      } catch (error) {
        console.error('Installation failed:', error);
        throw error;
      }
    })()
  );
});

self.addEventListener('activate', (event: ExtendableEvent) => {
  console.log(`Service Worker ${SW_VERSION} activating...`);

  event.waitUntil(
    (async () => {
      try {
        // Clean up old caches
        await cleanupOldCaches();

        // Claim all clients if configured
        if (self.clients && self.clients.claim) {
          await self.clients.claim();
        }

        console.log('Service Worker activated');
      } catch (error) {
        console.error('Activation failed:', error);
      }
    })()
  );
});

self.addEventListener('fetch', (event: FetchEvent) => {
  const { request } = event;
  const url = new URL(request.url);

  // Skip non-GET requests
  if (request.method !== 'GET') {
    return;
  }

  // Skip chrome-extension and other schemes
  if (!url.protocol.startsWith('http')) {
    return;
  }

  event.respondWith(handleFetchRequest(request));
});

// Fetch request handler with caching strategies
async function handleFetchRequest(request: Request): Promise<Response> {
  const url = new URL(request.url);

  // Network-first strategy for API calls
  if (NETWORK_FIRST_PATTERNS.some(pattern => pattern.test(url.pathname))) {
    return networkFirstStrategy(request);
  }

  // Cache-first strategy for static assets
  if (CACHE_FIRST_PATTERNS.some(pattern => pattern.test(url.pathname))) {
    return cacheFirstStrategy(request);
  }

  // Stale-while-revalidate for navigation requests
  if (request.mode === 'navigate') {
    return staleWhileRevalidateStrategy(request);
  }

  // Default to network-first
  return networkFirstStrategy(request);
}

// Network-first caching strategy
async function networkFirstStrategy(request: Request): Promise<Response> {
  try {
    const networkResponse = await fetch(request);
    
    if (networkResponse.ok) {
      // Cache successful responses
      const cache = await caches.open(DYNAMIC_CACHE);
      await cache.put(request, networkResponse.clone());
    }
    
    return networkResponse;
  } catch (error) {
    // Network failed, try cache
    const cachedResponse = await caches.match(request);
    
    if (cachedResponse) {
      return cachedResponse;
    }

    // Return offline page for navigation requests
    if (request.mode === 'navigate') {
      const offlineResponse = await caches.match('/offline.html');
      return offlineResponse || createOfflineResponse();
    }

    throw error;
  }
}

// Cache-first caching strategy
async function cacheFirstStrategy(request: Request): Promise<Response> {
  const cachedResponse = await caches.match(request);
  
  if (cachedResponse) {
    // Update cache in background
    updateCacheInBackground(request);
    return cachedResponse;
  }

  try {
    const networkResponse = await fetch(request);
    
    if (networkResponse.ok) {
      const cache = await caches.open(DYNAMIC_CACHE);
      await cache.put(request, networkResponse.clone());
    }
    
    return networkResponse;
  } catch (error) {
    console.error('Cache-first strategy failed:', error);
    throw error;
  }
}

// Stale-while-revalidate strategy
async function staleWhileRevalidateStrategy(request: Request): Promise<Response> {
  const cachedResponse = await caches.match(request);
  
  // Always try to update cache in background
  const networkPromise = updateCacheInBackground(request);

  if (cachedResponse) {
    return cachedResponse;
  }

  // If no cache, wait for network
  try {
    return await networkPromise;
  } catch (error) {
    return await caches.match('/offline.html') || createOfflineResponse();
  }
}

// Background cache update
async function updateCacheInBackground(request: Request): Promise<Response> {
  try {
    const networkResponse = await fetch(request);
    
    if (networkResponse.ok) {
      const cache = await caches.open(DYNAMIC_CACHE);
      await cache.put(request, networkResponse.clone());
    }
    
    return networkResponse;
  } catch (error) {
    console.warn('Background cache update failed:', error);
    throw error;
  }
}

// Clean up old caches
async function cleanupOldCaches(): Promise<void> {
  const currentCaches = [STATIC_CACHE, DYNAMIC_CACHE, CACHE_NAME];
  const cacheNames = await caches.keys();
  
  const deletePromises = cacheNames
    .filter(cacheName => !currentCaches.includes(cacheName))
    .map(cacheName => caches.delete(cacheName));

  await Promise.all(deletePromises);
}

// Create offline response
function createOfflineResponse(): Response {
  return new Response(
    `
    <!DOCTYPE html>
    <html>
      <head>
        <title>Offline</title>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <style>
          body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
          .offline-icon { font-size: 64px; margin-bottom: 20px; }
        </style>
      </head>
      <body>
        <div class="offline-icon">📱</div>
        <h1>You're offline</h1>
        <p>Please check your internet connection and try again.</p>
        <button onclick="window.location.reload()">Retry</button>
      </body>
    </html>
    `,
    {
      status: 200,
      headers: { 'Content-Type': 'text/html' },
    }
  );
}

// Message handling
self.addEventListener('message', (event: ExtendableMessageEvent) => {
  const { type, payload } = event.data;

  switch (type) {
    case 'SKIP_WAITING':
      self.skipWaiting();
      break;
    
    case 'GET_VERSION':
      event.ports[0]?.postMessage({ version: SW_VERSION });
      break;
    
    case 'CLEAR_CACHE':
      handleClearCache(payload);
      break;
    
    case 'PREFETCH_RESOURCES':
      handlePrefetchResources(payload);
      break;
  }
});

async function handleClearCache(cacheNames?: string[]): Promise<void> {
  try {
    if (cacheNames) {
      await Promise.all(cacheNames.map(name => caches.delete(name)));
    } else {
      const allCaches = await caches.keys();
      await Promise.all(allCaches.map(name => caches.delete(name)));
    }
    
    // Notify clients
    const clients = await self.clients.matchAll();
    clients.forEach(client => {
      client.postMessage({ type: 'CACHE_CLEARED' });
    });
  } catch (error) {
    console.error('Cache clear failed:', error);
  }
}

async function handlePrefetchResources(resources: string[]): Promise<void> {
  try {
    const cache = await caches.open(DYNAMIC_CACHE);
    await Promise.all(
      resources.map(async resource => {
        try {
          const response = await fetch(resource);
          if (response.ok) {
            await cache.put(resource, response);
          }
        } catch (error) {
          console.warn(`Failed to prefetch ${resource}:`, error);
        }
      })
    );
    
    // Notify clients
    const clients = await self.clients.matchAll();
    clients.forEach(client => {
      client.postMessage({ type: 'PREFETCH_COMPLETE', resources });
    });
  } catch (error) {
    console.error('Prefetch failed:', error);
  }
}

// React hook for Service Worker management
export function useServiceWorker(config: ServiceWorkerConfig) {
  const [isRegistered, setIsRegistered] = useState(false);
  const [hasUpdate, setHasUpdate] = useState(false);
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  const [swManager] = useState(() => new ServiceWorkerManager(config));

  useEffect(() => {
    const registerServiceWorker = async () => {
      try {
        await swManager.register();
        setIsRegistered(true);
      } catch (error) {
        console.error('Service Worker registration failed:', error);
      }
    };

    registerServiceWorker();

    // Setup update listener
    const unsubscribe = swManager.onUpdate((event, payload) => {
      switch (event) {
        case 'updateavailable':
          setHasUpdate(true);
          break;
        case 'controllerchange':
          window.location.reload();
          break;
        case 'offlinestatus':
          setIsOnline(payload.isOnline);
          break;
      }
    });

    // Listen for online/offline events
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      unsubscribe();
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, [swManager]);

  const updateServiceWorker = useCallback(async () => {
    await swManager.skipWaiting();
    setHasUpdate(false);
  }, [swManager]);

  const sendMessage = useCallback((message: any) => {
    swManager.sendMessage(message);
  }, [swManager]);

  return {
    isRegistered,
    hasUpdate,
    isOnline,
    updateServiceWorker,
    sendMessage,
    swManager,
  };
}
```

## Web Workers - Background Processing

### 1. Web Worker Implementation and Communication
```typescript
// Web Worker Manager for background processing
interface WebWorkerConfig {
  workerScript: string;
  transferable?: boolean;
  fallback?: () => any;
  timeout?: number;
}

interface WorkerTask {
  id: string;
  type: string;
  payload: any;
  transferable?: Transferable[];
  resolve: (value: any) => void;
  reject: (error: any) => void;
  timeout?: NodeJS.Timeout;
}

class WebWorkerManager {
  private worker: Worker | null = null;
  private config: WebWorkerConfig;
  private tasks = new Map<string, WorkerTask>();
  private isReady = false;
  private taskIdCounter = 0;

  constructor(config: WebWorkerConfig) {
    this.config = config;
    this.initializeWorker();
  }

  private async initializeWorker(): Promise<void> {
    try {
      if (!window.Worker) {
        throw new Error('Web Workers not supported');
      }

      this.worker = new Worker(this.config.workerScript);
      this.setupEventListeners();
      
      // Test worker readiness
      await this.executeTask('ping', {});
      this.isReady = true;
    } catch (error) {
      console.error('Web Worker initialization failed:', error);
      this.worker = null;
    }
  }

  private setupEventListeners(): void {
    if (!this.worker) return;

    this.worker.onmessage = (event) => {
      this.handleWorkerMessage(event);
    };

    this.worker.onerror = (error) => {
      console.error('Web Worker error:', error);
      this.rejectAllTasks(new Error('Worker error'));
    };

    this.worker.onmessageerror = (error) => {
      console.error('Web Worker message error:', error);
      this.rejectAllTasks(new Error('Worker message error'));
    };
  }

  private handleWorkerMessage(event: MessageEvent): void {
    const { taskId, result, error } = event.data;
    const task = this.tasks.get(taskId);

    if (!task) {
      console.warn('Received message for unknown task:', taskId);
      return;
    }

    this.tasks.delete(taskId);
    
    if (task.timeout) {
      clearTimeout(task.timeout);
    }

    if (error) {
      task.reject(new Error(error));
    } else {
      task.resolve(result);
    }
  }

  private generateTaskId(): string {
    return `task_${++this.taskIdCounter}_${Date.now()}`;
  }

  // Execute task in web worker
  async executeTask(type: string, payload: any, transferable?: Transferable[]): Promise<any> {
    if (!this.worker || !this.isReady) {
      if (this.config.fallback) {
        return this.config.fallback();
      }
      throw new Error('Web Worker not available');
    }

    return new Promise((resolve, reject) => {
      const taskId = this.generateTaskId();
      
      const task: WorkerTask = {
        id: taskId,
        type,
        payload,
        transferable,
        resolve,
        reject,
      };

      // Set timeout if configured
      if (this.config.timeout) {
        task.timeout = setTimeout(() => {
          this.tasks.delete(taskId);
          reject(new Error('Worker task timeout'));
        }, this.config.timeout);
      }

      this.tasks.set(taskId, task);

      // Send message to worker
      const message = { taskId, type, payload };
      
      if (transferable && this.config.transferable) {
        this.worker!.postMessage(message, transferable);
      } else {
        this.worker!.postMessage(message);
      }
    });
  }

  // Execute multiple tasks in parallel
  async executeParallelTasks(tasks: Array<{
    type: string;
    payload: any;
    transferable?: Transferable[];
  }>): Promise<any[]> {
    const promises = tasks.map(task =>
      this.executeTask(task.type, task.payload, task.transferable)
    );

    return Promise.all(promises);
  }

  // Execute tasks in sequence
  async executeSequentialTasks(tasks: Array<{
    type: string;
    payload: any;
    transferable?: Transferable[];
  }>): Promise<any[]> {
    const results = [];
    
    for (const task of tasks) {
      const result = await this.executeTask(task.type, task.payload, task.transferable);
      results.push(result);
    }

    return results;
  }

  private rejectAllTasks(error: Error): void {
    this.tasks.forEach(task => {
      if (task.timeout) {
        clearTimeout(task.timeout);
      }
      task.reject(error);
    });
    this.tasks.clear();
  }

  // Terminate worker
  terminate(): void {
    if (this.worker) {
      this.rejectAllTasks(new Error('Worker terminated'));
      this.worker.terminate();
      this.worker = null;
      this.isReady = false;
    }
  }

  // Check if worker is ready
  get ready(): boolean {
    return this.isReady;
  }
}

// Web Worker Script (worker.js)
/*
// Worker-side implementation
const workerTasks = {
  ping: () => 'pong',
  
  // Heavy computation example
  calculatePrimes: (data) => {
    const { start, end } = data;
    const primes = [];
    
    for (let i = start; i <= end; i++) {
      if (isPrime(i)) {
        primes.push(i);
      }
    }
    
    return primes;
  },
  
  // Image processing example
  processImage: (data) => {
    const { imageData, filter } = data;
    const { data: pixels, width, height } = imageData;
    
    switch (filter) {
      case 'grayscale':
        return applyGrayscaleFilter(pixels, width, height);
      case 'blur':
        return applyBlurFilter(pixels, width, height);
      default:
        return imageData;
    }
  },
  
  // Data processing example
  processLargeDataset: (data) => {
    const { dataset, operation } = data;
    
    switch (operation) {
      case 'sort':
        return dataset.sort((a, b) => a - b);
      case 'filter':
        return dataset.filter(item => item > 0);
      case 'map':
        return dataset.map(item => item * 2);
      default:
        return dataset;
    }
  },
  
  // Text analysis example
  analyzeText: (data) => {
    const { text } = data;
    
    const words = text.toLowerCase().split(/\W+/).filter(Boolean);
    const wordCount = words.length;
    const uniqueWords = new Set(words).size;
    const avgWordLength = words.reduce((sum, word) => sum + word.length, 0) / wordCount;
    
    const wordFrequency = {};
    words.forEach(word => {
      wordFrequency[word] = (wordFrequency[word] || 0) + 1;
    });
    
    return {
      wordCount,
      uniqueWords,
      avgWordLength,
      wordFrequency,
    };
  },
};

// Helper functions
function isPrime(num) {
  if (num < 2) return false;
  for (let i = 2; i <= Math.sqrt(num); i++) {
    if (num % i === 0) return false;
  }
  return true;
}

function applyGrayscaleFilter(pixels, width, height) {
  const result = new Uint8ClampedArray(pixels);
  
  for (let i = 0; i < result.length; i += 4) {
    const gray = 0.299 * result[i] + 0.587 * result[i + 1] + 0.114 * result[i + 2];
    result[i] = gray;     // Red
    result[i + 1] = gray; // Green
    result[i + 2] = gray; // Blue
    // Alpha channel (i + 3) remains unchanged
  }
  
  return new ImageData(result, width, height);
}

function applyBlurFilter(pixels, width, height) {
  // Simple box blur implementation
  const result = new Uint8ClampedArray(pixels);
  const radius = 2;
  
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      let r = 0, g = 0, b = 0, a = 0;
      let count = 0;
      
      for (let dy = -radius; dy <= radius; dy++) {
        for (let dx = -radius; dx <= radius; dx++) {
          const nx = x + dx;
          const ny = y + dy;
          
          if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
            const index = (ny * width + nx) * 4;
            r += pixels[index];
            g += pixels[index + 1];
            b += pixels[index + 2];
            a += pixels[index + 3];
            count++;
          }
        }
      }
      
      const index = (y * width + x) * 4;
      result[index] = r / count;
      result[index + 1] = g / count;
      result[index + 2] = b / count;
      result[index + 3] = a / count;
    }
  }
  
  return new ImageData(result, width, height);
}

// Message handler
self.onmessage = function(event) {
  const { taskId, type, payload } = event.data;
  
  try {
    const taskFunction = workerTasks[type];
    
    if (!taskFunction) {
      throw new Error(`Unknown task type: ${type}`);
    }
    
    const result = taskFunction(payload);
    
    self.postMessage({ taskId, result });
  } catch (error) {
    self.postMessage({ taskId, error: error.message });
  }
};
*/

// React hook for Web Worker management
export function useWebWorker(config: WebWorkerConfig) {
  const [workerManager] = useState(() => new WebWorkerManager(config));
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    const checkReady = () => {
      setIsReady(workerManager.ready);
    };

    // Check readiness periodically
    const interval = setInterval(checkReady, 100);
    checkReady(); // Initial check

    return () => {
      clearInterval(interval);
      workerManager.terminate();
    };
  }, [workerManager]);

  const executeTask = useCallback(async (type: string, payload: any, transferable?: Transferable[]) => {
    return workerManager.executeTask(type, payload, transferable);
  }, [workerManager]);

  const executeParallelTasks = useCallback(async (tasks: any[]) => {
    return workerManager.executeParallelTasks(tasks);
  }, [workerManager]);

  const executeSequentialTasks = useCallback(async (tasks: any[]) => {
    return workerManager.executeSequentialTasks(tasks);
  }, [workerManager]);

  return {
    isReady,
    executeTask,
    executeParallelTasks,
    executeSequentialTasks,
    terminate: () => workerManager.terminate(),
  };
}

// Background Processing Hook
export function useBackgroundProcessing() {
  const workerConfig = useMemo(() => ({
    workerScript: '/workers/background-processor.js',
    transferable: true,
    timeout: 30000, // 30 seconds
    fallback: () => console.warn('Web Worker fallback not implemented'),
  }), []);

  const { isReady, executeTask, executeParallelTasks } = useWebWorker(workerConfig);

  // Process large dataset
  const processLargeDataset = useCallback(async (dataset: any[], operation: string) => {
    if (!isReady) {
      throw new Error('Background processor not ready');
    }

    const chunkSize = Math.ceil(dataset.length / 4); // Split into 4 chunks
    const chunks = [];
    
    for (let i = 0; i < dataset.length; i += chunkSize) {
      chunks.push(dataset.slice(i, i + chunkSize));
    }

    const tasks = chunks.map(chunk => ({
      type: 'processLargeDataset',
      payload: { dataset: chunk, operation },
    }));

    const results = await executeParallelTasks(tasks);
    
    // Merge results
    return results.flat();
  }, [isReady, executeParallelTasks]);

  // Process image
  const processImage = useCallback(async (imageData: ImageData, filter: string) => {
    if (!isReady) {
      throw new Error('Background processor not ready');
    }

    return executeTask('processImage', { imageData, filter }, [imageData.data.buffer]);
  }, [isReady, executeTask]);

  // Analyze text
  const analyzeText = useCallback(async (text: string) => {
    if (!isReady) {
      throw new Error('Background processor not ready');
    }

    return executeTask('analyzeText', { text });
  }, [isReady, executeTask]);

  return {
    isReady,
    processLargeDataset,
    processImage,
    analyzeText,
  };
}
```

### 2. Background Sync and Push Notifications
```typescript
// Background Sync Implementation
interface BackgroundSyncConfig {
  syncTag: string;
  fallbackInterval?: number;
  maxRetries?: number;
}

class BackgroundSyncManager {
  private registration: ServiceWorkerRegistration | null = null;
  private config: BackgroundSyncConfig;
  private pendingTasks = new Map<string, any>();

  constructor(registration: ServiceWorkerRegistration, config: BackgroundSyncConfig) {
    this.registration = registration;
    this.config = config;
  }

  // Register background sync
  async registerSync(tag: string, data?: any): Promise<void> {
    if (!this.registration) {
      throw new Error('Service Worker registration required');
    }

    if ('serviceWorker' in navigator && 'sync' in window.ServiceWorkerRegistration.prototype) {
      try {
        // Store task data for later use
        if (data) {
          this.pendingTasks.set(tag, data);
          localStorage.setItem(`sync_task_${tag}`, JSON.stringify(data));
        }

        await (this.registration as any).sync.register(tag);
        console.log(`Background sync registered: ${tag}`);
      } catch (error) {
        console.error('Background sync registration failed:', error);
        
        // Fallback to interval-based sync
        this.setupFallbackSync(tag, data);
      }
    } else {
      console.warn('Background sync not supported');
      this.setupFallbackSync(tag, data);
    }
  }

  private setupFallbackSync(tag: string, data?: any): void {
    const interval = this.config.fallbackInterval || 60000; // Default 1 minute
    
    const fallbackSync = setInterval(async () => {
      try {
        const success = await this.executeSyncTask(tag, data);
        
        if (success) {
          clearInterval(fallbackSync);
          this.pendingTasks.delete(tag);
          localStorage.removeItem(`sync_task_${tag}`);
        }
      } catch (error) {
        console.error('Fallback sync failed:', error);
      }
    }, interval);

    // Store interval ID for cleanup
    this.pendingTasks.set(`${tag}_interval`, fallbackSync);
  }

  private async executeSyncTask(tag: string, data?: any): Promise<boolean> {
    // This would typically be handled in the service worker
    // For demonstration, we'll handle it here
    try {
      switch (tag) {
        case 'background-upload':
          return await this.handleBackgroundUpload(data);
        case 'offline-actions':
          return await this.handleOfflineActions(data);
        case 'data-sync':
          return await this.handleDataSync(data);
        default:
          console.warn(`Unknown sync tag: ${tag}`);
          return false;
      }
    } catch (error) {
      console.error(`Sync task failed for ${tag}:`, error);
      return false;
    }
  }

  private async handleBackgroundUpload(data: any): Promise<boolean> {
    if (!navigator.onLine) {
      return false;
    }

    try {
      const response = await fetch('/api/upload', {
        method: 'POST',
        body: JSON.stringify(data),
        headers: {
          'Content-Type': 'application/json',
        },
      });

      return response.ok;
    } catch (error) {
      return false;
    }
  }

  private async handleOfflineActions(data: any): Promise<boolean> {
    if (!navigator.onLine) {
      return false;
    }

    try {
      // Process queued offline actions
      const actions = data.actions || [];
      
      for (const action of actions) {
        const response = await fetch(action.url, {
          method: action.method,
          body: action.body,
          headers: action.headers,
        });

        if (!response.ok) {
          throw new Error(`Action failed: ${action.type}`);
        }
      }

      return true;
    } catch (error) {
      return false;
    }
  }

  private async handleDataSync(data: any): Promise<boolean> {
    if (!navigator.onLine) {
      return false;
    }

    try {
      // Sync local data with server
      const localData = data.localData || {};
      
      const response = await fetch('/api/sync', {
        method: 'POST',
        body: JSON.stringify(localData),
        headers: {
          'Content-Type': 'application/json',
        },
      });

      if (response.ok) {
        const serverData = await response.json();
        
        // Update local storage with server data
        Object.keys(serverData).forEach(key => {
          localStorage.setItem(key, JSON.stringify(serverData[key]));
        });

        return true;
      }

      return false;
    } catch (error) {
      return false;
    }
  }

  // Get pending sync tasks
  getPendingTasks(): Map<string, any> {
    return new Map(this.pendingTasks);
  }

  // Clear pending task
  clearPendingTask(tag: string): void {
    const intervalId = this.pendingTasks.get(`${tag}_interval`);
    if (intervalId) {
      clearInterval(intervalId);
      this.pendingTasks.delete(`${tag}_interval`);
    }
    
    this.pendingTasks.delete(tag);
    localStorage.removeItem(`sync_task_${tag}`);
  }
}

// Push Notification Manager
interface PushNotificationConfig {
  vapidPublicKey: string;
  serverEndpoint: string;
  notificationOptions?: NotificationOptions;
}

class PushNotificationManager {
  private registration: ServiceWorkerRegistration | null = null;
  private config: PushNotificationConfig;
  private subscription: PushSubscription | null = null;

  constructor(registration: ServiceWorkerRegistration, config: PushNotificationConfig) {
    this.registration = registration;
    this.config = config;
  }

  // Request notification permission
  async requestPermission(): Promise<NotificationPermission> {
    if (!('Notification' in window)) {
      throw new Error('Notifications not supported');
    }

    let permission = Notification.permission;

    if (permission === 'default') {
      permission = await Notification.requestPermission();
    }

    return permission;
  }

  // Subscribe to push notifications
  async subscribe(): Promise<PushSubscription> {
    if (!this.registration) {
      throw new Error('Service Worker registration required');
    }

    const permission = await this.requestPermission();
    
    if (permission !== 'granted') {
      throw new Error('Notification permission denied');
    }

    try {
      this.subscription = await this.registration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: this.urlBase64ToUint8Array(this.config.vapidPublicKey),
      });

      // Send subscription to server
      await this.sendSubscriptionToServer(this.subscription);

      return this.subscription;
    } catch (error) {
      console.error('Push subscription failed:', error);
      throw error;
    }
  }

  // Unsubscribe from push notifications
  async unsubscribe(): Promise<boolean> {
    if (!this.subscription) {
      return true;
    }

    try {
      const success = await this.subscription.unsubscribe();
      
      if (success) {
        // Notify server about unsubscription
        await this.removeSubscriptionFromServer(this.subscription);
        this.subscription = null;
      }

      return success;
    } catch (error) {
      console.error('Push unsubscription failed:', error);
      return false;
    }
  }

  // Get current subscription
  async getSubscription(): Promise<PushSubscription | null> {
    if (!this.registration) {
      return null;
    }

    try {
      this.subscription = await this.registration.pushManager.getSubscription();
      return this.subscription;
    } catch (error) {
      console.error('Failed to get push subscription:', error);
      return null;
    }
  }

  // Send test notification
  async sendTestNotification(): Promise<void> {
    if (!this.subscription) {
      throw new Error('No active push subscription');
    }

    try {
      await fetch(`${this.config.serverEndpoint}/send-notification`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          subscription: this.subscription,
          notification: {
            title: 'Test Notification',
            body: 'This is a test push notification',
            icon: '/icons/icon-192x192.png',
            badge: '/icons/badge-72x72.png',
            ...this.config.notificationOptions,
          },
        }),
      });
    } catch (error) {
      console.error('Failed to send test notification:', error);
      throw error;
    }
  }

  private urlBase64ToUint8Array(base64String: string): Uint8Array {
    const padding = '='.repeat((4 - base64String.length % 4) % 4);
    const base64 = (base64String + padding)
      .replace(/\-/g, '+')
      .replace(/_/g, '/');

    const rawData = window.atob(base64);
    const outputArray = new Uint8Array(rawData.length);

    for (let i = 0; i < rawData.length; ++i) {
      outputArray[i] = rawData.charCodeAt(i);
    }
    
    return outputArray;
  }

  private async sendSubscriptionToServer(subscription: PushSubscription): Promise<void> {
    try {
      await fetch(`${this.config.serverEndpoint}/subscribe`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(subscription),
      });
    } catch (error) {
      console.error('Failed to send subscription to server:', error);
      throw error;
    }
  }

  private async removeSubscriptionFromServer(subscription: PushSubscription): Promise<void> {
    try {
      await fetch(`${this.config.serverEndpoint}/unsubscribe`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(subscription),
      });
    } catch (error) {
      console.error('Failed to remove subscription from server:', error);
    }
  }
}

// React hooks for background sync and push notifications
export function useBackgroundSync(config: BackgroundSyncConfig) {
  const [syncManager, setSyncManager] = useState<BackgroundSyncManager | null>(null);
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    const initializeSync = async () => {
      if ('serviceWorker' in navigator) {
        try {
          const registration = await navigator.serviceWorker.ready;
          const manager = new BackgroundSyncManager(registration, config);
          setSyncManager(manager);
          setIsReady(true);
        } catch (error) {
          console.error('Background sync initialization failed:', error);
        }
      }
    };

    initializeSync();
  }, [config]);

  const registerSync = useCallback(async (tag: string, data?: any) => {
    if (!syncManager) {
      throw new Error('Background sync not ready');
    }

    return syncManager.registerSync(tag, data);
  }, [syncManager]);

  return {
    isReady,
    registerSync,
    getPendingTasks: () => syncManager?.getPendingTasks() || new Map(),
    clearPendingTask: (tag: string) => syncManager?.clearPendingTask(tag),
  };
}

export function usePushNotifications(config: PushNotificationConfig) {
  const [notificationManager, setNotificationManager] = useState<PushNotificationManager | null>(null);
  const [isSubscribed, setIsSubscribed] = useState(false);
  const [permission, setPermission] = useState<NotificationPermission>('default');

  useEffect(() => {
    const initializeNotifications = async () => {
      if ('serviceWorker' in navigator && 'PushManager' in window) {
        try {
          const registration = await navigator.serviceWorker.ready;
          const manager = new PushNotificationManager(registration, config);
          setNotificationManager(manager);

          // Check current subscription
          const subscription = await manager.getSubscription();
          setIsSubscribed(!!subscription);
          
          // Check permission
          setPermission(Notification.permission);
        } catch (error) {
          console.error('Push notification initialization failed:', error);
        }
      }
    };

    initializeNotifications();
  }, [config]);

  const subscribe = useCallback(async () => {
    if (!notificationManager) {
      throw new Error('Push notification manager not ready');
    }

    try {
      await notificationManager.subscribe();
      setIsSubscribed(true);
      setPermission('granted');
    } catch (error) {
      console.error('Push subscription failed:', error);
      throw error;
    }
  }, [notificationManager]);

  const unsubscribe = useCallback(async () => {
    if (!notificationManager) {
      return false;
    }

    try {
      const success = await notificationManager.unsubscribe();
      setIsSubscribed(false);
      return success;
    } catch (error) {
      console.error('Push unsubscription failed:', error);
      return false;
    }
  }, [notificationManager]);

  const sendTestNotification = useCallback(async () => {
    if (!notificationManager) {
      throw new Error('Push notification manager not ready');
    }

    return notificationManager.sendTestNotification();
  }, [notificationManager]);

  return {
    isSubscribed,
    permission,
    subscribe,
    unsubscribe,
    sendTestNotification,
  };
}
```

## Interview-Ready Summary

**Service Workers and Web Workers enable:**

1. **Service Workers** - Offline functionality, caching strategies, background sync, push notifications, app shell architecture
2. **Web Workers** - Background processing, heavy computations, image processing, data analysis without blocking UI
3. **Background Sync** - Offline action queuing, data synchronization, retry mechanisms, fallback strategies
4. **Push Notifications** - Real-time communication, user engagement, permission management, VAPID integration

**Key capabilities:**
- **Offline-First** - Cache-first, network-first, stale-while-revalidate strategies
- **Background Processing** - CPU-intensive tasks, parallel processing, transferable objects
- **Background Sync** - Queue offline actions, sync when online, retry failed requests
- **Push Messaging** - Server-initiated notifications, subscription management, payload delivery

**Service Worker strategies:** Install/activate lifecycle, cache management, fetch interception, message handling, version updates.

**Web Worker patterns:** Task queuing, parallel processing, data transfer optimization, error handling, fallback strategies.

**Best practices:** Implement progressive enhancement, handle offline gracefully, optimize cache strategies, manage worker lifecycle, handle permissions properly, provide fallbacks, monitor performance impact.