# Edge Computing and CDN Optimization

Edge computing and Content Delivery Network (CDN) optimization are critical for delivering high-performance frontend applications globally. This comprehensive guide covers edge computing architectures, advanced CDN strategies, edge functions, service workers, and performance optimization techniques for modern web applications.

## Edge Computing Architecture Patterns

### Multi-Tier Edge Computing Strategy

```typescript
// edge/edge-architecture.ts
interface EdgeNode {
  id: string;
  location: {
    country: string;
    region: string;
    city: string;
    coordinates: [number, number];
  };
  capabilities: {
    compute: boolean;
    storage: boolean;
    caching: boolean;
    streaming: boolean;
  };
  resources: {
    cpu: number;
    memory: number;
    storage: number;
    bandwidth: number;
  };
  latency: {
    toOrigin: number;
    toUser: number;
  };
  load: {
    current: number;
    capacity: number;
  };
}

interface EdgeRequest {
  id: string;
  userId: string;
  path: string;
  method: string;
  headers: Record<string, string>;
  body?: any;
  metadata: {
    userLocation: [number, number];
    deviceType: 'mobile' | 'desktop' | 'tablet';
    connection: '2g' | '3g' | '4g' | '5g' | 'wifi' | 'ethernet';
    timestamp: number;
  };
}

interface EdgeResponse {
  data: any;
  cached: boolean;
  processed: boolean;
  source: 'edge' | 'origin' | 'cache';
  latency: number;
  headers: Record<string, string>;
}

class EdgeComputingManager {
  private nodes: Map<string, EdgeNode> = new Map();
  private cache: Map<string, any> = new Map();
  private analytics: EdgeAnalytics;

  constructor() {
    this.analytics = new EdgeAnalytics();
    this.initializeNodes();
  }

  private initializeNodes(): void {
    // Initialize edge nodes across global locations
    const nodeConfigs: Partial<EdgeNode>[] = [
      {
        id: 'us-east-1',
        location: { country: 'US', region: 'East', city: 'New York', coordinates: [40.7128, -74.0060] },
        capabilities: { compute: true, storage: true, caching: true, streaming: true },
        resources: { cpu: 16, memory: 32, storage: 1000, bandwidth: 10000 }
      },
      {
        id: 'us-west-1',
        location: { country: 'US', region: 'West', city: 'San Francisco', coordinates: [37.7749, -122.4194] },
        capabilities: { compute: true, storage: true, caching: true, streaming: true },
        resources: { cpu: 16, memory: 32, storage: 1000, bandwidth: 10000 }
      },
      {
        id: 'eu-west-1',
        location: { country: 'UK', region: 'West', city: 'London', coordinates: [51.5074, -0.1278] },
        capabilities: { compute: true, storage: true, caching: true, streaming: true },
        resources: { cpu: 12, memory: 24, storage: 800, bandwidth: 8000 }
      },
      {
        id: 'ap-southeast-1',
        location: { country: 'SG', region: 'Southeast', city: 'Singapore', coordinates: [1.3521, 103.8198] },
        capabilities: { compute: true, storage: true, caching: true, streaming: true },
        resources: { cpu: 12, memory: 24, storage: 800, bandwidth: 8000 }
      }
    ];

    nodeConfigs.forEach(config => {
      const node: EdgeNode = {
        ...config,
        latency: { toOrigin: 50, toUser: 10 },
        load: { current: 0, capacity: 100 }
      } as EdgeNode;
      
      this.nodes.set(node.id, node);
    });
  }

  public async processRequest(request: EdgeRequest): Promise<EdgeResponse> {
    const startTime = performance.now();
    
    // 1. Select optimal edge node
    const selectedNode = this.selectOptimalNode(request);
    
    // 2. Check cache first
    const cacheKey = this.generateCacheKey(request);
    let response = await this.checkCache(cacheKey, selectedNode);
    
    if (response) {
      response.cached = true;
      response.source = 'cache';
    } else {
      // 3. Process at edge or forward to origin
      if (this.canProcessAtEdge(request, selectedNode)) {
        response = await this.processAtEdge(request, selectedNode);
        response.source = 'edge';
        response.processed = true;
      } else {
        response = await this.forwardToOrigin(request, selectedNode);
        response.source = 'origin';
      }
      
      // 4. Cache the response
      await this.cacheResponse(cacheKey, response, selectedNode);
    }
    
    response.latency = performance.now() - startTime;
    
    // 5. Track analytics
    this.analytics.trackRequest(request, response, selectedNode);
    
    return response;
  }

  private selectOptimalNode(request: EdgeRequest): EdgeNode {
    const userLocation = request.metadata.userLocation;
    let bestNode: EdgeNode | null = null;
    let bestScore = Infinity;

    for (const node of this.nodes.values()) {
      // Calculate distance-based score
      const distance = this.calculateDistance(userLocation, node.location.coordinates);
      
      // Factor in current load
      const loadFactor = node.load.current / node.load.capacity;
      
      // Factor in capabilities
      const capabilityScore = this.calculateCapabilityScore(request, node);
      
      // Composite score (lower is better)
      const score = distance * 0.4 + loadFactor * 100 * 0.3 + (1 - capabilityScore) * 100 * 0.3;
      
      if (score < bestScore) {
        bestScore = score;
        bestNode = node;
      }
    }

    return bestNode || Array.from(this.nodes.values())[0];
  }

  private calculateDistance(point1: [number, number], point2: [number, number]): number {
    const [lat1, lon1] = point1;
    const [lat2, lon2] = point2;
    
    const R = 6371; // Earth's radius in kilometers
    const dLat = this.toRadians(lat2 - lat1);
    const dLon = this.toRadians(lon2 - lon1);
    
    const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
              Math.cos(this.toRadians(lat1)) * Math.cos(this.toRadians(lat2)) *
              Math.sin(dLon / 2) * Math.sin(dLon / 2);
    
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
  }

  private toRadians(degrees: number): number {
    return degrees * (Math.PI / 180);
  }

  private calculateCapabilityScore(request: EdgeRequest, node: EdgeNode): number {
    let score = 0;
    
    // Check if node can handle the request type
    if (request.path.includes('/api/') && node.capabilities.compute) score += 0.3;
    if (request.path.includes('/stream/') && node.capabilities.streaming) score += 0.3;
    if (request.method === 'GET' && node.capabilities.caching) score += 0.2;
    if (node.capabilities.storage) score += 0.2;
    
    return Math.min(score, 1);
  }

  private generateCacheKey(request: EdgeRequest): string {
    const keyData = {
      path: request.path,
      method: request.method,
      userId: request.headers.authorization ? 'authenticated' : 'anonymous',
      deviceType: request.metadata.deviceType,
      connection: request.metadata.connection
    };
    
    return `cache:${Buffer.from(JSON.stringify(keyData)).toString('base64')}`;
  }

  private async checkCache(cacheKey: string, node: EdgeNode): Promise<EdgeResponse | null> {
    if (!node.capabilities.caching) return null;
    
    const cached = this.cache.get(cacheKey);
    if (!cached) return null;
    
    // Check if cache is still valid
    const now = Date.now();
    if (now - cached.timestamp > cached.ttl) {
      this.cache.delete(cacheKey);
      return null;
    }
    
    return {
      data: cached.data,
      cached: true,
      processed: false,
      source: 'cache',
      latency: 0,
      headers: cached.headers
    };
  }

  private async cacheResponse(cacheKey: string, response: EdgeResponse, node: EdgeNode): Promise<void> {
    if (!node.capabilities.caching || response.cached) return;
    
    // Determine TTL based on content type and cache headers
    const ttl = this.determineTTL(response);
    
    this.cache.set(cacheKey, {
      data: response.data,
      headers: response.headers,
      timestamp: Date.now(),
      ttl
    });
  }

  private determineTTL(response: EdgeResponse): number {
    const cacheControl = response.headers['cache-control'];
    if (cacheControl) {
      const maxAgeMatch = cacheControl.match(/max-age=(\d+)/);
      if (maxAgeMatch) {
        return parseInt(maxAgeMatch[1]) * 1000; // Convert to milliseconds
      }
    }
    
    // Default TTL based on content type
    const contentType = response.headers['content-type'] || '';
    if (contentType.includes('text/html')) return 5 * 60 * 1000; // 5 minutes
    if (contentType.includes('application/json')) return 1 * 60 * 1000; // 1 minute
    if (contentType.includes('image/') || contentType.includes('text/css')) return 60 * 60 * 1000; // 1 hour
    
    return 10 * 60 * 1000; // 10 minutes default
  }

  private canProcessAtEdge(request: EdgeRequest, node: EdgeNode): boolean {
    if (!node.capabilities.compute) return false;
    
    // Check if this is a request that can be processed at the edge
    const edgeProcessiblePaths = [
      '/api/search',
      '/api/recommendations', 
      '/api/personalization',
      '/api/analytics',
      '/api/user-preferences'
    ];
    
    return edgeProcessiblePaths.some(path => request.path.startsWith(path));
  }

  private async processAtEdge(request: EdgeRequest, node: EdgeNode): Promise<EdgeResponse> {
    // Simulate edge processing
    await this.updateNodeLoad(node.id, 10);
    
    let data: any;
    
    try {
      if (request.path.startsWith('/api/search')) {
        data = await this.processSearchAtEdge(request);
      } else if (request.path.startsWith('/api/recommendations')) {
        data = await this.processRecommendationsAtEdge(request);
      } else if (request.path.startsWith('/api/personalization')) {
        data = await this.processPersonalizationAtEdge(request);
      } else {
        throw new Error('Cannot process at edge');
      }
      
      return {
        data,
        cached: false,
        processed: true,
        source: 'edge',
        latency: 0,
        headers: {
          'content-type': 'application/json',
          'cache-control': 'max-age=300',
          'x-processed-at': node.id
        }
      };
    } finally {
      await this.updateNodeLoad(node.id, -10);
    }
  }

  private async processSearchAtEdge(request: EdgeRequest): Promise<any> {
    // Simulate search processing with local cache/index
    const query = new URL(`http://example.com${request.path}`).searchParams.get('q');
    
    return {
      query,
      results: [
        { id: 1, title: 'Search Result 1', score: 0.95 },
        { id: 2, title: 'Search Result 2', score: 0.87 },
      ],
      total: 2,
      processingTime: 15,
      source: 'edge-cache'
    };
  }

  private async processRecommendationsAtEdge(request: EdgeRequest): Promise<any> {
    // Simulate ML-based recommendations at edge
    const userId = request.headers['x-user-id'];
    
    return {
      userId,
      recommendations: [
        { id: 'item1', score: 0.92, reason: 'Similar users liked' },
        { id: 'item2', score: 0.85, reason: 'Based on history' },
      ],
      algorithm: 'edge-collaborative-filtering',
      confidence: 0.88
    };
  }

  private async processPersonalizationAtEdge(request: EdgeRequest): Promise<any> {
    // Simulate personalization at edge
    const deviceType = request.metadata.deviceType;
    const connection = request.metadata.connection;
    
    return {
      layout: deviceType === 'mobile' ? 'mobile-optimized' : 'desktop',
      features: {
        highQualityImages: ['4g', '5g', 'wifi', 'ethernet'].includes(connection),
        autoplayVideo: connection === '5g' || connection === 'wifi',
        prefetchContent: connection === 'wifi' || connection === 'ethernet'
      },
      theme: 'adaptive',
      optimizations: {
        bundleSize: deviceType === 'mobile' ? 'minimal' : 'full',
        imageFormat: 'webp',
        compressionLevel: connection === '2g' ? 'high' : 'standard'
      }
    };
  }

  private async forwardToOrigin(request: EdgeRequest, node: EdgeNode): Promise<EdgeResponse> {
    // Simulate forwarding to origin server
    const latency = node.latency.toOrigin;
    
    await new Promise(resolve => setTimeout(resolve, latency));
    
    return {
      data: { message: 'Response from origin server', requestId: request.id },
      cached: false,
      processed: false,
      source: 'origin',
      latency,
      headers: {
        'content-type': 'application/json',
        'cache-control': 'max-age=3600',
        'x-origin-server': 'main-api'
      }
    };
  }

  private async updateNodeLoad(nodeId: string, delta: number): Promise<void> {
    const node = this.nodes.get(nodeId);
    if (node) {
      node.load.current = Math.max(0, Math.min(node.load.capacity, node.load.current + delta));
    }
  }

  public getNodeStatus(): EdgeNode[] {
    return Array.from(this.nodes.values());
  }

  public getAnalytics(): any {
    return this.analytics.getReport();
  }
}

class EdgeAnalytics {
  private requests: any[] = [];
  private performance: Map<string, number[]> = new Map();

  public trackRequest(request: EdgeRequest, response: EdgeResponse, node: EdgeNode): void {
    const record = {
      timestamp: Date.now(),
      requestId: request.id,
      path: request.path,
      method: request.method,
      nodeId: node.id,
      latency: response.latency,
      source: response.source,
      cached: response.cached,
      processed: response.processed,
      deviceType: request.metadata.deviceType,
      connection: request.metadata.connection,
      userLocation: request.metadata.userLocation
    };

    this.requests.push(record);
    
    // Track performance metrics
    const key = `${node.id}:${response.source}`;
    if (!this.performance.has(key)) {
      this.performance.set(key, []);
    }
    this.performance.get(key)!.push(response.latency);
    
    // Keep only recent records
    if (this.requests.length > 10000) {
      this.requests = this.requests.slice(-5000);
    }
  }

  public getReport(): any {
    const now = Date.now();
    const oneHourAgo = now - 60 * 60 * 1000;
    
    const recentRequests = this.requests.filter(r => r.timestamp > oneHourAgo);
    
    const metrics = {
      totalRequests: recentRequests.length,
      cacheHitRate: recentRequests.filter(r => r.cached).length / recentRequests.length,
      edgeProcessingRate: recentRequests.filter(r => r.processed).length / recentRequests.length,
      averageLatency: recentRequests.reduce((sum, r) => sum + r.latency, 0) / recentRequests.length,
      
      bySource: this.groupBy(recentRequests, 'source'),
      byNode: this.groupBy(recentRequests, 'nodeId'),
      byDevice: this.groupBy(recentRequests, 'deviceType'),
      byConnection: this.groupBy(recentRequests, 'connection'),
      
      performanceByNode: this.calculatePerformanceMetrics()
    };

    return metrics;
  }

  private groupBy(array: any[], key: string): Record<string, any> {
    return array.reduce((result, item) => {
      const group = item[key];
      if (!result[group]) {
        result[group] = { count: 0, totalLatency: 0 };
      }
      result[group].count++;
      result[group].totalLatency += item.latency;
      result[group].averageLatency = result[group].totalLatency / result[group].count;
      return result;
    }, {});
  }

  private calculatePerformanceMetrics(): Record<string, any> {
    const metrics: Record<string, any> = {};
    
    for (const [key, latencies] of this.performance.entries()) {
      if (latencies.length === 0) continue;
      
      latencies.sort((a, b) => a - b);
      const len = latencies.length;
      
      metrics[key] = {
        count: len,
        min: latencies[0],
        max: latencies[len - 1],
        avg: latencies.reduce((a, b) => a + b, 0) / len,
        p50: latencies[Math.floor(len * 0.5)],
        p95: latencies[Math.floor(len * 0.95)],
        p99: latencies[Math.floor(len * 0.99)]
      };
    }
    
    return metrics;
  }
}

export { EdgeComputingManager, EdgeNode, EdgeRequest, EdgeResponse };
```

## Advanced CDN Strategies

### Intelligent Cache Management

```typescript
// cdn/intelligent-cache-manager.ts
interface CacheConfig {
  ttl: number;
  maxSize: number;
  compressionLevel: number;
  purgeStrategy: 'lru' | 'lfu' | 'fifo' | 'adaptive';
  revalidationStrategy: 'stale-while-revalidate' | 'background-refresh' | 'immediate';
}

interface CacheEntry {
  key: string;
  data: any;
  headers: Record<string, string>;
  timestamp: number;
  ttl: number;
  accessCount: number;
  lastAccessed: number;
  size: number;
  compressed: boolean;
  tags: string[];
  metadata: {
    contentType: string;
    userAgent?: string;
    deviceType?: string;
    location?: string;
  };
}

interface CacheStats {
  hitRate: number;
  missRate: number;
  totalRequests: number;
  totalHits: number;
  totalMisses: number;
  averageResponseTime: number;
  cacheSizeUtilization: number;
  compressionRatio: number;
  purgeEvents: number;
}

class IntelligentCacheManager {
  private cache: Map<string, CacheEntry> = new Map();
  private config: CacheConfig;
  private stats: CacheStats;
  private accessLog: Array<{ key: string; timestamp: number; hit: boolean }> = [];
  private mlPredictor: CachePredictionModel;

  constructor(config: CacheConfig) {
    this.config = config;
    this.stats = this.initializeStats();
    this.mlPredictor = new CachePredictionModel();
    
    // Periodic cleanup
    setInterval(() => this.performMaintenance(), 60000); // Every minute
  }

  private initializeStats(): CacheStats {
    return {
      hitRate: 0,
      missRate: 0,
      totalRequests: 0,
      totalHits: 0,
      totalMisses: 0,
      averageResponseTime: 0,
      cacheSizeUtilization: 0,
      compressionRatio: 0,
      purgeEvents: 0
    };
  }

  public async get(key: string, metadata?: any): Promise<CacheEntry | null> {
    const startTime = performance.now();
    this.stats.totalRequests++;

    const entry = this.cache.get(key);
    
    if (entry && this.isValidEntry(entry)) {
      // Update access statistics
      entry.accessCount++;
      entry.lastAccessed = Date.now();
      
      this.stats.totalHits++;
      this.logAccess(key, true);
      
      // Check if we need background revalidation
      if (this.shouldRevalidate(entry)) {
        this.scheduleBackgroundRevalidation(key, metadata);
      }
      
      this.updateResponseTimeStats(performance.now() - startTime);
      return entry;
    }

    this.stats.totalMisses++;
    this.logAccess(key, false);
    this.updateResponseTimeStats(performance.now() - startTime);
    
    return null;
  }

  public async set(
    key: string,
    data: any,
    headers: Record<string, string> = {},
    options: {
      ttl?: number;
      tags?: string[];
      metadata?: any;
      compress?: boolean;
    } = {}
  ): Promise<void> {
    const ttl = options.ttl || this.config.ttl;
    const shouldCompress = options.compress !== false && this.shouldCompress(data);
    
    let processedData = data;
    let size = this.calculateSize(data);
    
    if (shouldCompress) {
      processedData = await this.compress(data);
      size = this.calculateSize(processedData);
    }

    const entry: CacheEntry = {
      key,
      data: processedData,
      headers,
      timestamp: Date.now(),
      ttl,
      accessCount: 0,
      lastAccessed: Date.now(),
      size,
      compressed: shouldCompress,
      tags: options.tags || [],
      metadata: {
        contentType: headers['content-type'] || 'application/octet-stream',
        ...options.metadata
      }
    };

    // Check if we need to make space
    while (this.getCurrentCacheSize() + size > this.config.maxSize) {
      await this.evictEntry();
    }

    this.cache.set(key, entry);
    
    // Train ML model with new cache entry
    this.mlPredictor.train(key, entry, this.getAccessPattern(key));
  }

  public async invalidate(key: string): Promise<boolean> {
    return this.cache.delete(key);
  }

  public async invalidateByTag(tag: string): Promise<number> {
    let count = 0;
    for (const [key, entry] of this.cache.entries()) {
      if (entry.tags.includes(tag)) {
        this.cache.delete(key);
        count++;
      }
    }
    return count;
  }

  public async invalidateByPattern(pattern: RegExp): Promise<number> {
    let count = 0;
    for (const key of this.cache.keys()) {
      if (pattern.test(key)) {
        this.cache.delete(key);
        count++;
      }
    }
    return count;
  }

  private isValidEntry(entry: CacheEntry): boolean {
    const now = Date.now();
    const age = now - entry.timestamp;
    
    // Check TTL
    if (age > entry.ttl) {
      return false;
    }
    
    // Additional staleness checks for revalidation strategies
    if (this.config.revalidationStrategy === 'stale-while-revalidate') {
      const staleThreshold = entry.ttl * 0.8; // 80% of TTL
      return age < staleThreshold;
    }
    
    return true;
  }

  private shouldRevalidate(entry: CacheEntry): boolean {
    if (this.config.revalidationStrategy !== 'background-refresh') {
      return false;
    }
    
    const age = Date.now() - entry.timestamp;
    const refreshThreshold = entry.ttl * 0.7; // 70% of TTL
    
    return age > refreshThreshold;
  }

  private async scheduleBackgroundRevalidation(key: string, metadata?: any): Promise<void> {
    // In a real implementation, this would trigger a background job
    console.log(`Scheduling background revalidation for key: ${key}`);
  }

  private shouldCompress(data: any): boolean {
    const size = this.calculateSize(data);
    const sizeThreshold = 1024; // 1KB
    
    if (size < sizeThreshold) return false;
    
    // Check content type
    if (typeof data === 'string') {
      return true; // Text content usually compresses well
    }
    
    return false;
  }

  private async compress(data: any): Promise<any> {
    // Simulate compression (in real implementation, use gzip/brotli)
    if (typeof data === 'string') {
      return `compressed:${data}`;
    }
    return data;
  }

  private async decompress(data: any): Promise<any> {
    // Simulate decompression
    if (typeof data === 'string' && data.startsWith('compressed:')) {
      return data.substring('compressed:'.length);
    }
    return data;
  }

  private calculateSize(data: any): number {
    if (typeof data === 'string') {
      return new Blob([data]).size;
    }
    return JSON.stringify(data).length * 2; // Rough estimate
  }

  private getCurrentCacheSize(): number {
    let totalSize = 0;
    for (const entry of this.cache.values()) {
      totalSize += entry.size;
    }
    return totalSize;
  }

  private async evictEntry(): Promise<void> {
    let keyToEvict: string | null = null;
    
    switch (this.config.purgeStrategy) {
      case 'lru':
        keyToEvict = this.findLRUKey();
        break;
      case 'lfu':
        keyToEvict = this.findLFUKey();
        break;
      case 'fifo':
        keyToEvict = this.findFIFOKey();
        break;
      case 'adaptive':
        keyToEvict = this.findAdaptiveKey();
        break;
    }

    if (keyToEvict) {
      this.cache.delete(keyToEvict);
      this.stats.purgeEvents++;
    }
  }

  private findLRUKey(): string | null {
    let oldestKey: string | null = null;
    let oldestTime = Infinity;

    for (const [key, entry] of this.cache.entries()) {
      if (entry.lastAccessed < oldestTime) {
        oldestTime = entry.lastAccessed;
        oldestKey = key;
      }
    }

    return oldestKey;
  }

  private findLFUKey(): string | null {
    let leastUsedKey: string | null = null;
    let leastUsedCount = Infinity;

    for (const [key, entry] of this.cache.entries()) {
      if (entry.accessCount < leastUsedCount) {
        leastUsedCount = entry.accessCount;
        leastUsedKey = key;
      }
    }

    return leastUsedKey;
  }

  private findFIFOKey(): string | null {
    let oldestKey: string | null = null;
    let oldestTimestamp = Infinity;

    for (const [key, entry] of this.cache.entries()) {
      if (entry.timestamp < oldestTimestamp) {
        oldestTimestamp = entry.timestamp;
        oldestKey = key;
      }
    }

    return oldestKey;
  }

  private findAdaptiveKey(): string | null {
    // Use ML model to predict which key is least likely to be accessed
    const predictions = new Map<string, number>();
    
    for (const key of this.cache.keys()) {
      const pattern = this.getAccessPattern(key);
      const probability = this.mlPredictor.predict(key, pattern);
      predictions.set(key, probability);
    }

    // Find key with lowest access probability
    let lowestKey: string | null = null;
    let lowestProbability = Infinity;

    for (const [key, probability] of predictions.entries()) {
      if (probability < lowestProbability) {
        lowestProbability = probability;
        lowestKey = key;
      }
    }

    return lowestKey;
  }

  private getAccessPattern(key: string): number[] {
    const recentAccesses = this.accessLog
      .filter(log => log.key === key)
      .slice(-24); // Last 24 accesses
    
    // Convert to time-based pattern
    const pattern: number[] = new Array(24).fill(0);
    const now = Date.now();
    const hourMs = 60 * 60 * 1000;

    for (const access of recentAccesses) {
      const hoursAgo = Math.floor((now - access.timestamp) / hourMs);
      if (hoursAgo < 24) {
        pattern[23 - hoursAgo]++;
      }
    }

    return pattern;
  }

  private logAccess(key: string, hit: boolean): void {
    this.accessLog.push({
      key,
      timestamp: Date.now(),
      hit
    });

    // Keep only recent access logs
    if (this.accessLog.length > 10000) {
      this.accessLog = this.accessLog.slice(-5000);
    }
  }

  private updateResponseTimeStats(responseTime: number): void {
    const alpha = 0.1; // Exponential moving average factor
    this.stats.averageResponseTime = 
      this.stats.averageResponseTime * (1 - alpha) + responseTime * alpha;
  }

  private performMaintenance(): void {
    this.updateStats();
    this.cleanupExpiredEntries();
    this.optimizeCache();
  }

  private updateStats(): void {
    const total = this.stats.totalRequests;
    if (total > 0) {
      this.stats.hitRate = this.stats.totalHits / total;
      this.stats.missRate = this.stats.totalMisses / total;
    }

    this.stats.cacheSizeUtilization = this.getCurrentCacheSize() / this.config.maxSize;
  }

  private cleanupExpiredEntries(): void {
    const now = Date.now();
    for (const [key, entry] of this.cache.entries()) {
      if (now - entry.timestamp > entry.ttl) {
        this.cache.delete(key);
      }
    }
  }

  private optimizeCache(): void {
    // Implement cache optimization strategies
    this.compressLowAccessEntries();
    this.adjustTTLBasedOnUsage();
  }

  private compressLowAccessEntries(): void {
    const now = Date.now();
    for (const [key, entry] of this.cache.entries()) {
      if (!entry.compressed && 
          now - entry.lastAccessed > 60 * 60 * 1000 && // 1 hour
          entry.accessCount < 5) {
        // Compress low-usage entries
        this.compressEntry(key, entry);
      }
    }
  }

  private async compressEntry(key: string, entry: CacheEntry): Promise<void> {
    if (entry.compressed) return;
    
    const compressedData = await this.compress(entry.data);
    entry.data = compressedData;
    entry.compressed = true;
    entry.size = this.calculateSize(compressedData);
  }

  private adjustTTLBasedOnUsage(): void {
    for (const [key, entry] of this.cache.entries()) {
      const accessRate = entry.accessCount / ((Date.now() - entry.timestamp) / 3600000); // per hour
      
      if (accessRate > 10) {
        // High access rate - extend TTL
        entry.ttl = Math.min(entry.ttl * 1.1, this.config.ttl * 2);
      } else if (accessRate < 1) {
        // Low access rate - reduce TTL
        entry.ttl = Math.max(entry.ttl * 0.9, this.config.ttl * 0.5);
      }
    }
  }

  public getStats(): CacheStats {
    return { ...this.stats };
  }

  public getCacheInfo(): any {
    return {
      totalEntries: this.cache.size,
      totalSize: this.getCurrentCacheSize(),
      sizeUtilization: this.getCurrentCacheSize() / this.config.maxSize,
      config: this.config,
      topKeys: this.getTopAccessedKeys(10)
    };
  }

  private getTopAccessedKeys(limit: number): Array<{ key: string; accessCount: number }> {
    return Array.from(this.cache.entries())
      .map(([key, entry]) => ({ key, accessCount: entry.accessCount }))
      .sort((a, b) => b.accessCount - a.accessCount)
      .slice(0, limit);
  }
}

// Machine Learning model for cache prediction
class CachePredictionModel {
  private model: Map<string, any> = new Map();

  public train(key: string, entry: CacheEntry, accessPattern: number[]): void {
    // Simple training data storage (in real implementation, use TensorFlow.js or similar)
    const features = {
      contentType: entry.metadata.contentType,
      size: entry.size,
      accessPattern: accessPattern,
      timestamp: entry.timestamp,
      compressed: entry.compressed
    };

    this.model.set(key, features);
  }

  public predict(key: string, accessPattern: number[]): number {
    const features = this.model.get(key);
    if (!features) return 0.5; // Default probability

    // Simple prediction based on access pattern similarity
    const similarity = this.calculateSimilarity(accessPattern, features.accessPattern);
    const recency = Math.exp(-(Date.now() - features.timestamp) / (24 * 60 * 60 * 1000)); // Decay over 24 hours
    
    return similarity * 0.7 + recency * 0.3;
  }

  private calculateSimilarity(pattern1: number[], pattern2: number[]): number {
    if (pattern1.length !== pattern2.length) return 0;

    let dotProduct = 0;
    let norm1 = 0;
    let norm2 = 0;

    for (let i = 0; i < pattern1.length; i++) {
      dotProduct += pattern1[i] * pattern2[i];
      norm1 += pattern1[i] * pattern1[i];
      norm2 += pattern2[i] * pattern2[i];
    }

    if (norm1 === 0 || norm2 === 0) return 0;
    
    return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
  }
}

export { IntelligentCacheManager, CacheConfig, CacheEntry, CacheStats };
```

## Edge Functions and Service Workers

### Advanced Service Worker Implementation

```typescript
// service-worker/advanced-sw.ts
/// <reference lib="webworker" />

interface CacheStrategy {
  name: string;
  urlPattern: RegExp;
  strategy: 'cache-first' | 'network-first' | 'stale-while-revalidate' | 'network-only' | 'cache-only';
  cacheName: string;
  maxEntries?: number;
  maxAgeSeconds?: number;
  networkTimeoutSeconds?: number;
}

interface SyncTask {
  id: string;
  tag: string;
  data: any;
  timestamp: number;
  retryCount: number;
  maxRetries: number;
}

class AdvancedServiceWorker {
  private static instance: AdvancedServiceWorker;
  private cacheStrategies: CacheStrategy[] = [];
  private syncTasks: Map<string, SyncTask> = new Map();
  private analytics: ServiceWorkerAnalytics;

  constructor() {
    this.analytics = new ServiceWorkerAnalytics();
    this.setupCacheStrategies();
    this.setupEventListeners();
  }

  public static getInstance(): AdvancedServiceWorker {
    if (!AdvancedServiceWorker.instance) {
      AdvancedServiceWorker.instance = new AdvancedServiceWorker();
    }
    return AdvancedServiceWorker.instance;
  }

  private setupCacheStrategies(): void {
    this.cacheStrategies = [
      {
        name: 'app-shell',
        urlPattern: /\/(index\.html|manifest\.json|sw\.js)$/,
        strategy: 'cache-first',
        cacheName: 'app-shell-v1',
        maxEntries: 10,
        maxAgeSeconds: 24 * 60 * 60 // 24 hours
      },
      {
        name: 'static-assets',
        urlPattern: /\.(?:js|css|png|jpg|jpeg|gif|svg|woff|woff2)$/,
        strategy: 'cache-first',
        cacheName: 'static-assets-v1',
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60 // 30 days
      },
      {
        name: 'api-cache',
        urlPattern: /\/api\/(?:products|categories|search)/,
        strategy: 'stale-while-revalidate',
        cacheName: 'api-cache-v1',
        maxEntries: 50,
        maxAgeSeconds: 5 * 60, // 5 minutes
        networkTimeoutSeconds: 3
      },
      {
        name: 'user-data',
        urlPattern: /\/api\/user/,
        strategy: 'network-first',
        cacheName: 'user-data-v1',
        maxEntries: 20,
        maxAgeSeconds: 60, // 1 minute
        networkTimeoutSeconds: 5
      },
      {
        name: 'images',
        urlPattern: /\/images\//,
        strategy: 'cache-first',
        cacheName: 'images-v1',
        maxEntries: 200,
        maxAgeSeconds: 7 * 24 * 60 * 60 // 7 days
      }
    ];
  }

  private setupEventListeners(): void {
    self.addEventListener('install', this.handleInstall.bind(this));
    self.addEventListener('activate', this.handleActivate.bind(this));
    self.addEventListener('fetch', this.handleFetch.bind(this));
    self.addEventListener('sync', this.handleBackgroundSync.bind(this));
    self.addEventListener('push', this.handlePush.bind(this));
    self.addEventListener('notificationclick', this.handleNotificationClick.bind(this));
    self.addEventListener('message', this.handleMessage.bind(this));
  }

  private async handleInstall(event: ExtendableEvent): Promise<void> {
    console.log('Service Worker installing...');
    
    event.waitUntil(
      Promise.all([
        this.precacheAppShell(),
        this.setupPeriodicSync(),
        self.skipWaiting()
      ])
    );
  }

  private async handleActivate(event: ExtendableEvent): Promise<void> {
    console.log('Service Worker activating...');
    
    event.waitUntil(
      Promise.all([
        this.cleanupOldCaches(),
        this.clearExpiredData(),
        self.clients.claim()
      ])
    );
  }

  private async handleFetch(event: FetchEvent): Promise<void> {
    const { request } = event;
    
    // Skip non-GET requests and chrome-extension requests
    if (request.method !== 'GET' || request.url.startsWith('chrome-extension://')) {
      return;
    }

    const strategy = this.findMatchingStrategy(request.url);
    
    if (strategy) {
      event.respondWith(this.executeStrategy(request, strategy));
    }
  }

  private findMatchingStrategy(url: string): CacheStrategy | null {
    return this.cacheStrategies.find(strategy => strategy.urlPattern.test(url)) || null;
  }

  private async executeStrategy(request: Request, strategy: CacheStrategy): Promise<Response> {
    const startTime = performance.now();
    
    try {
      let response: Response;
      
      switch (strategy.strategy) {
        case 'cache-first':
          response = await this.cacheFirst(request, strategy);
          break;
        case 'network-first':
          response = await this.networkFirst(request, strategy);
          break;
        case 'stale-while-revalidate':
          response = await this.staleWhileRevalidate(request, strategy);
          break;
        case 'network-only':
          response = await this.networkOnly(request);
          break;
        case 'cache-only':
          response = await this.cacheOnly(request, strategy);
          break;
        default:
          response = await fetch(request);
      }

      // Track analytics
      this.analytics.trackRequest(request, response, strategy.strategy, performance.now() - startTime);
      
      return response;
    } catch (error) {
      console.error('Fetch strategy failed:', error);
      return await this.handleFetchError(request, strategy);
    }
  }

  private async cacheFirst(request: Request, strategy: CacheStrategy): Promise<Response> {
    const cache = await caches.open(strategy.cacheName);
    const cachedResponse = await cache.match(request);
    
    if (cachedResponse && !this.isCacheExpired(cachedResponse, strategy)) {
      return cachedResponse;
    }
    
    try {
      const networkResponse = await fetch(request);
      if (networkResponse.ok) {
        await this.putInCache(cache, request, networkResponse.clone(), strategy);
      }
      return networkResponse;
    } catch (error) {
      if (cachedResponse) {
        return cachedResponse; // Return stale cache as fallback
      }
      throw error;
    }
  }

  private async networkFirst(request: Request, strategy: CacheStrategy): Promise<Response> {
    const cache = await caches.open(strategy.cacheName);
    
    try {
      const networkResponse = await this.fetchWithTimeout(request, strategy.networkTimeoutSeconds);
      if (networkResponse.ok) {
        await this.putInCache(cache, request, networkResponse.clone(), strategy);
      }
      return networkResponse;
    } catch (error) {
      const cachedResponse = await cache.match(request);
      if (cachedResponse) {
        return cachedResponse;
      }
      throw error;
    }
  }

  private async staleWhileRevalidate(request: Request, strategy: CacheStrategy): Promise<Response> {
    const cache = await caches.open(strategy.cacheName);
    const cachedResponse = await cache.match(request);
    
    // Always try to fetch from network in background
    const networkResponsePromise = fetch(request).then(response => {
      if (response.ok) {
        this.putInCache(cache, request, response.clone(), strategy);
      }
      return response;
    });
    
    // Return cached response immediately if available
    if (cachedResponse && !this.isCacheExpired(cachedResponse, strategy)) {
      return cachedResponse;
    }
    
    // If no cache or expired, wait for network
    return await networkResponsePromise;
  }

  private async networkOnly(request: Request): Promise<Response> {
    return await fetch(request);
  }

  private async cacheOnly(request: Request, strategy: CacheStrategy): Promise<Response> {
    const cache = await caches.open(strategy.cacheName);
    const cachedResponse = await cache.match(request);
    
    if (cachedResponse) {
      return cachedResponse;
    }
    
    throw new Error('No cached response available');
  }

  private async fetchWithTimeout(request: Request, timeoutSeconds?: number): Promise<Response> {
    if (!timeoutSeconds) {
      return await fetch(request);
    }
    
    const timeoutPromise = new Promise<Response>((_, reject) => {
      setTimeout(() => reject(new Error('Network timeout')), timeoutSeconds * 1000);
    });
    
    return await Promise.race([fetch(request), timeoutPromise]);
  }

  private async putInCache(
    cache: Cache, 
    request: Request, 
    response: Response, 
    strategy: CacheStrategy
  ): Promise<void> {
    if (!response.ok) return;
    
    // Add timestamp header for expiration checking
    const responseWithTimestamp = new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: {
        ...Object.fromEntries(response.headers),
        'sw-cached-at': Date.now().toString()
      }
    });
    
    await cache.put(request, responseWithTimestamp);
    
    // Cleanup old entries if necessary
    await this.cleanupCache(cache, strategy);
  }

  private isCacheExpired(response: Response, strategy: CacheStrategy): boolean {
    if (!strategy.maxAgeSeconds) return false;
    
    const cachedAt = response.headers.get('sw-cached-at');
    if (!cachedAt) return false;
    
    const age = (Date.now() - parseInt(cachedAt)) / 1000;
    return age > strategy.maxAgeSeconds;
  }

  private async cleanupCache(cache: Cache, strategy: CacheStrategy): Promise<void> {
    if (!strategy.maxEntries) return;
    
    const requests = await cache.keys();
    
    if (requests.length > strategy.maxEntries) {
      // Remove oldest entries
      const sortedRequests = await Promise.all(
        requests.map(async (request) => {
          const response = await cache.match(request);
          const cachedAt = response?.headers.get('sw-cached-at') || '0';
          return { request, cachedAt: parseInt(cachedAt) };
        })
      );
      
      sortedRequests.sort((a, b) => a.cachedAt - b.cachedAt);
      
      const entriesToDelete = sortedRequests.slice(0, requests.length - strategy.maxEntries);
      await Promise.all(
        entriesToDelete.map(entry => cache.delete(entry.request))
      );
    }
  }

  private async handleFetchError(request: Request, strategy: CacheStrategy): Promise<Response> {
    // Try to serve from cache as fallback
    const cache = await caches.open(strategy.cacheName);
    const cachedResponse = await cache.match(request);
    
    if (cachedResponse) {
      return cachedResponse;
    }
    
    // Return offline page for navigation requests
    if (request.mode === 'navigate') {
      const offlineResponse = await cache.match('/offline.html');
      if (offlineResponse) {
        return offlineResponse;
      }
    }
    
    // Return generic error response
    return new Response('Network error occurred', {
      status: 503,
      statusText: 'Service Unavailable'
    });
  }

  private async precacheAppShell(): Promise<void> {
    const cache = await caches.open('app-shell-v1');
    const urlsToCache = [
      '/',
      '/index.html',
      '/manifest.json',
      '/offline.html',
      '/static/css/main.css',
      '/static/js/main.js',
      '/icons/icon-192x192.png',
      '/icons/icon-512x512.png'
    ];
    
    await cache.addAll(urlsToCache);
  }

  private async cleanupOldCaches(): Promise<void> {
    const cacheNames = await caches.keys();
    const currentCacheNames = this.cacheStrategies.map(s => s.cacheName);
    
    await Promise.all(
      cacheNames
        .filter(name => !currentCacheNames.includes(name))
        .map(name => caches.delete(name))
    );
  }

  private async clearExpiredData(): Promise<void> {
    // Clear expired sync tasks
    const now = Date.now();
    for (const [id, task] of this.syncTasks.entries()) {
      if (now - task.timestamp > 24 * 60 * 60 * 1000) { // 24 hours
        this.syncTasks.delete(id);
      }
    }
  }

  private async setupPeriodicSync(): Promise<void> {
    // Setup periodic background sync for data updates
    if ('serviceWorker' in navigator && 'sync' in window.ServiceWorkerRegistration.prototype) {
      const registration = await navigator.serviceWorker.ready;
      await registration.sync.register('background-data-sync');
    }
  }

  private async handleBackgroundSync(event: any): Promise<void> {
    if (event.tag === 'background-data-sync') {
      event.waitUntil(this.doBackgroundSync());
    }
  }

  private async doBackgroundSync(): Promise<void> {
    console.log('Performing background sync...');
    
    // Process queued sync tasks
    for (const [id, task] of this.syncTasks.entries()) {
      try {
        await this.processSyncTask(task);
        this.syncTasks.delete(id);
      } catch (error) {
        task.retryCount++;
        if (task.retryCount >= task.maxRetries) {
          this.syncTasks.delete(id);
        }
      }
    }
  }

  private async processSyncTask(task: SyncTask): Promise<void> {
    // Process different types of sync tasks
    switch (task.tag) {
      case 'analytics':
        await this.syncAnalyticsData(task.data);
        break;
      case 'user-data':
        await this.syncUserData(task.data);
        break;
      default:
        console.warn('Unknown sync task:', task.tag);
    }
  }

  private async syncAnalyticsData(data: any): Promise<void> {
    await fetch('/api/analytics', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
  }

  private async syncUserData(data: any): Promise<void> {
    await fetch('/api/user/sync', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
  }

  private async handlePush(event: PushEvent): Promise<void> {
    const options = {
      body: 'You have a new message!',
      icon: '/icons/icon-192x192.png',
      badge: '/icons/badge-72x72.png',
      vibrate: [100, 50, 100],
      data: {
        dateOfArrival: Date.now(),
        primaryKey: 1
      },
      actions: [
        {
          action: 'explore',
          title: 'Explore',
          icon: '/icons/checkmark.png'
        },
        {
          action: 'close',
          title: 'Close',
          icon: '/icons/xmark.png'
        }
      ]
    };

    if (event.data) {
      const payload = event.data.json();
      options.body = payload.body || options.body;
      options.data = { ...options.data, ...payload.data };
    }

    event.waitUntil(
      self.registration.showNotification('PWA Notification', options)
    );
  }

  private async handleNotificationClick(event: NotificationEvent): Promise<void> {
    event.notification.close();

    if (event.action === 'explore') {
      event.waitUntil(
        self.clients.openWindow('/explore')
      );
    } else if (event.action === 'close') {
      // Just close the notification
    } else {
      // Default action - open the app
      event.waitUntil(
        self.clients.openWindow('/')
      );
    }
  }

  private async handleMessage(event: ExtendableMessageEvent): Promise<void> {
    const { data } = event;
    
    switch (data.type) {
      case 'SKIP_WAITING':
        self.skipWaiting();
        break;
      case 'GET_CACHE_INFO':
        const cacheInfo = await this.getCacheInfo();
        event.ports[0].postMessage(cacheInfo);
        break;
      case 'CLEAR_CACHE':
        await this.clearAllCaches();
        event.ports[0].postMessage({ success: true });
        break;
      case 'QUEUE_SYNC_TASK':
        this.queueSyncTask(data.task);
        break;
    }
  }

  private async getCacheInfo(): Promise<any> {
    const cacheInfo = {};
    
    for (const strategy of this.cacheStrategies) {
      const cache = await caches.open(strategy.cacheName);
      const requests = await cache.keys();
      cacheInfo[strategy.cacheName] = {
        entryCount: requests.length,
        maxEntries: strategy.maxEntries,
        strategy: strategy.strategy
      };
    }
    
    return cacheInfo;
  }

  private async clearAllCaches(): Promise<void> {
    const cacheNames = await caches.keys();
    await Promise.all(cacheNames.map(name => caches.delete(name)));
  }

  private queueSyncTask(task: Partial<SyncTask>): void {
    const syncTask: SyncTask = {
      id: task.id || this.generateId(),
      tag: task.tag!,
      data: task.data,
      timestamp: Date.now(),
      retryCount: 0,
      maxRetries: task.maxRetries || 3
    };
    
    this.syncTasks.set(syncTask.id, syncTask);
  }

  private generateId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

class ServiceWorkerAnalytics {
  private requests: any[] = [];
  
  public trackRequest(
    request: Request, 
    response: Response, 
    strategy: string, 
    duration: number
  ): void {
    this.requests.push({
      url: request.url,
      method: request.method,
      status: response.status,
      strategy,
      duration,
      timestamp: Date.now(),
      fromCache: response.headers.has('sw-cached-at')
    });
    
    // Keep only recent requests
    if (this.requests.length > 1000) {
      this.requests = this.requests.slice(-500);
    }
  }
  
  public getAnalytics(): any {
    const totalRequests = this.requests.length;
    const cacheHits = this.requests.filter(r => r.fromCache).length;
    const averageDuration = this.requests.reduce((sum, r) => sum + r.duration, 0) / totalRequests;
    
    return {
      totalRequests,
      cacheHitRate: cacheHits / totalRequests,
      averageDuration,
      strategyCounts: this.groupBy(this.requests, 'strategy'),
      statusCounts: this.groupBy(this.requests, 'status')
    };
  }
  
  private groupBy(array: any[], key: string): Record<string, number> {
    return array.reduce((result, item) => {
      const group = item[key];
      result[group] = (result[group] || 0) + 1;
      return result;
    }, {});
  }
}

// Initialize the service worker
const sw = AdvancedServiceWorker.getInstance();

export { AdvancedServiceWorker };
```

## Interview-Ready Edge Computing & CDN Summary

**Edge Computing Architecture:**
1. **Multi-Tier Edge Strategy** - Global node distribution with intelligent request routing and load balancing
2. **Edge Processing** - Compute-at-edge for search, recommendations, and personalization
3. **Adaptive Caching** - ML-driven cache management with predictive eviction policies
4. **Analytics & Monitoring** - Real-time performance tracking and optimization

**Advanced CDN Optimization:**
- Intelligent cache management with compression and background revalidation
- Content-aware caching strategies with TTL optimization
- Geographic distribution with latency-based routing
- Cache warming and predictive pre-loading

**Service Worker Excellence:**
- Multi-strategy caching (cache-first, network-first, stale-while-revalidate)
- Background sync with retry mechanisms and task queuing
- Offline-first architecture with graceful degradation
- Push notifications and progressive enhancement

**Key Interview Topics:** Edge computing vs traditional CDN, cache invalidation strategies, service worker lifecycle management, offline-first design patterns, performance optimization techniques, global content distribution, edge security considerations, and progressive web app capabilities.