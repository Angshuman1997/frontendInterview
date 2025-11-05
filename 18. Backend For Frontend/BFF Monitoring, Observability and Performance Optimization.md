# BFF Monitoring, Observability and Performance Optimization

Comprehensive monitoring and observability strategies for Backend For Frontend (BFF) services are crucial for maintaining enterprise-grade performance and reliability. This guide covers advanced monitoring patterns, performance optimization techniques, and production observability best practices.

## BFF Monitoring Architecture

### 1. Comprehensive Observability Stack
```typescript
// BFF Observability Integration
import express, { Request, Response, NextFunction } from 'express';
import prometheus from 'prom-client';
import opentelemetry from '@opentelemetry/api';
import { NodeSDK } from '@opentelemetry/auto-instrumentations-node';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';

// Metrics Collection
class BFFMetrics {
  private static instance: BFFMetrics;
  private registry: prometheus.Registry;
  private httpRequestDuration: prometheus.Histogram<string>;
  private httpRequestTotal: prometheus.Counter<string>;
  private activeConnections: prometheus.Gauge<string>;
  private bffRequestDuration: prometheus.Histogram<string>;
  private serviceCallDuration: prometheus.Histogram<string>;
  private cacheHitRate: prometheus.Gauge<string>;
  private errorRate: prometheus.Counter<string>;

  private constructor() {
    this.registry = new prometheus.Registry();
    this.initializeMetrics();
    prometheus.collectDefaultMetrics({ register: this.registry });
  }

  static getInstance(): BFFMetrics {
    if (!BFFMetrics.instance) {
      BFFMetrics.instance = new BFFMetrics();
    }
    return BFFMetrics.instance;
  }

  private initializeMetrics(): void {
    // HTTP Request Duration
    this.httpRequestDuration = new prometheus.Histogram({
      name: 'bff_http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status_code', 'platform'],
      buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
    });

    // HTTP Request Total
    this.httpRequestTotal = new prometheus.Counter({
      name: 'bff_http_requests_total',
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'route', 'status_code', 'platform']
    });

    // Active Connections
    this.activeConnections = new prometheus.Gauge({
      name: 'bff_active_connections',
      help: 'Number of active connections',
      labelNames: ['service']
    });

    // BFF-specific Request Duration
    this.bffRequestDuration = new prometheus.Histogram({
      name: 'bff_request_processing_duration_seconds',
      help: 'Time spent processing BFF requests',
      labelNames: ['operation', 'platform', 'cache_status'],
      buckets: [0.01, 0.05, 0.1, 0.2, 0.5, 1, 2, 5]
    });

    // Service Call Duration
    this.serviceCallDuration = new prometheus.Histogram({
      name: 'bff_service_call_duration_seconds',
      help: 'Duration of downstream service calls',
      labelNames: ['service', 'method', 'status'],
      buckets: [0.01, 0.05, 0.1, 0.2, 0.5, 1, 2, 5, 10]
    });

    // Cache Hit Rate
    this.cacheHitRate = new prometheus.Gauge({
      name: 'bff_cache_hit_rate',
      help: 'Cache hit rate percentage',
      labelNames: ['cache_type', 'operation']
    });

    // Error Rate
    this.errorRate = new prometheus.Counter({
      name: 'bff_errors_total',
      help: 'Total number of errors',
      labelNames: ['type', 'service', 'operation']
    });

    // Register all metrics
    this.registry.registerMetric(this.httpRequestDuration);
    this.registry.registerMetric(this.httpRequestTotal);
    this.registry.registerMetric(this.activeConnections);
    this.registry.registerMetric(this.bffRequestDuration);
    this.registry.registerMetric(this.serviceCallDuration);
    this.registry.registerMetric(this.cacheHitRate);
    this.registry.registerMetric(this.errorRate);
  }

  // Record HTTP request metrics
  recordHttpRequest(method: string, route: string, statusCode: number, duration: number, platform: string): void {
    this.httpRequestDuration
      .labels(method, route, statusCode.toString(), platform)
      .observe(duration);
    
    this.httpRequestTotal
      .labels(method, route, statusCode.toString(), platform)
      .inc();
  }

  // Record BFF processing metrics
  recordBFFProcessing(operation: string, platform: string, duration: number, cacheStatus: string): void {
    this.bffRequestDuration
      .labels(operation, platform, cacheStatus)
      .observe(duration);
  }

  // Record service call metrics
  recordServiceCall(service: string, method: string, status: string, duration: number): void {
    this.serviceCallDuration
      .labels(service, method, status)
      .observe(duration);
  }

  // Update cache metrics
  updateCacheHitRate(cacheType: string, operation: string, hitRate: number): void {
    this.cacheHitRate
      .labels(cacheType, operation)
      .set(hitRate);
  }

  // Record error
  recordError(type: string, service: string, operation: string): void {
    this.errorRate
      .labels(type, service, operation)
      .inc();
  }

  // Get metrics registry
  getRegistry(): prometheus.Registry {
    return this.registry;
  }

  // Generate metrics report
  async getMetricsReport(): Promise<string> {
    return this.registry.metrics();
  }
}

// Tracing Integration
class BFFTracing {
  private tracer: opentelemetry.Tracer;
  private sdk: NodeSDK;

  constructor(serviceName: string, version: string) {
    this.initializeTracing(serviceName, version);
    this.tracer = opentelemetry.trace.getTracer(serviceName, version);
  }

  private initializeTracing(serviceName: string, version: string): void {
    const jaegerExporter = new JaegerExporter({
      endpoint: process.env.JAEGER_ENDPOINT || 'http://localhost:14268/api/traces',
    });

    this.sdk = new NodeSDK({
      resource: new Resource({
        [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
        [SemanticResourceAttributes.SERVICE_VERSION]: version,
        [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
      }),
      spanProcessor: new BatchSpanProcessor(jaegerExporter),
    });

    this.sdk.start();
  }

  // Create span for BFF operation
  createBFFSpan(operationName: string, attributes: Record<string, any> = {}): opentelemetry.Span {
    const span = this.tracer.startSpan(operationName, {
      attributes: {
        'service.type': 'bff',
        'operation.type': 'aggregation',
        ...attributes
      }
    });

    return span;
  }

  // Create span for service call
  createServiceCallSpan(serviceName: string, operation: string, attributes: Record<string, any> = {}): opentelemetry.Span {
    const span = this.tracer.startSpan(`${serviceName}.${operation}`, {
      attributes: {
        'service.name': serviceName,
        'service.operation': operation,
        'call.type': 'downstream',
        ...attributes
      }
    });

    return span;
  }

  // Add span event
  addSpanEvent(span: opentelemetry.Span, name: string, attributes: Record<string, any> = {}): void {
    span.addEvent(name, attributes);
  }

  // Set span error
  setSpanError(span: opentelemetry.Span, error: Error): void {
    span.recordException(error);
    span.setStatus({ code: opentelemetry.SpanStatusCode.ERROR, message: error.message });
  }

  // End span
  endSpan(span: opentelemetry.Span): void {
    span.end();
  }
}

// Logging Integration
import winston from 'winston';
import { ElasticsearchTransport } from 'winston-elasticsearch';

class BFFLogger {
  private logger: winston.Logger;

  constructor(serviceName: string) {
    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json(),
        winston.format.printf(({ timestamp, level, message, service, operation, traceId, spanId, ...meta }) => {
          return JSON.stringify({
            timestamp,
            level,
            message,
            service: serviceName,
            operation,
            traceId,
            spanId,
            ...meta
          });
        })
      ),
      defaultMeta: { service: serviceName },
      transports: [
        new winston.transports.Console({
          format: winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
          )
        }),
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' }),
      ],
    });

    // Add Elasticsearch transport for production
    if (process.env.NODE_ENV === 'production' && process.env.ELASTICSEARCH_URL) {
      this.logger.add(new ElasticsearchTransport({
        level: 'info',
        clientOpts: { node: process.env.ELASTICSEARCH_URL },
        index: `bff-logs-${serviceName}`,
      }));
    }
  }

  info(message: string, meta: any = {}): void {
    this.logger.info(message, this.addTraceContext(meta));
  }

  warn(message: string, meta: any = {}): void {
    this.logger.warn(message, this.addTraceContext(meta));
  }

  error(message: string, error?: Error, meta: any = {}): void {
    this.logger.error(message, {
      ...this.addTraceContext(meta),
      error: error ? {
        message: error.message,
        stack: error.stack,
        name: error.name
      } : undefined
    });
  }

  debug(message: string, meta: any = {}): void {
    this.logger.debug(message, this.addTraceContext(meta));
  }

  private addTraceContext(meta: any): any {
    const activeSpan = opentelemetry.trace.getActiveSpan();
    if (activeSpan) {
      const spanContext = activeSpan.spanContext();
      return {
        ...meta,
        traceId: spanContext.traceId,
        spanId: spanContext.spanId
      };
    }
    return meta;
  }
}

// Middleware for metrics and tracing
function createObservabilityMiddleware(metrics: BFFMetrics, tracer: BFFTracing, logger: BFFLogger) {
  return (req: Request, res: Response, next: NextFunction) => {
    const startTime = process.hrtime.bigint();
    const platform = req.headers['x-platform'] || 'web';
    const operation = `${req.method} ${req.route?.path || req.path}`;

    // Create tracing span
    const span = tracer.createBFFSpan(operation, {
      'http.method': req.method,
      'http.url': req.originalUrl,
      'http.user_agent': req.headers['user-agent'],
      'http.platform': platform,
      'user.id': req.user?.id
    });

    // Add trace context to request
    req.traceContext = {
      span,
      traceId: span.spanContext().traceId,
      spanId: span.spanContext().spanId
    };

    // Log request start
    logger.info('BFF request started', {
      operation,
      method: req.method,
      url: req.originalUrl,
      platform,
      userId: req.user?.id
    });

    // Override res.end to capture metrics
    const originalEnd = res.end;
    res.end = function(chunk?: any, encoding?: any) {
      const endTime = process.hrtime.bigint();
      const duration = Number(endTime - startTime) / 1_000_000_000; // Convert to seconds

      // Record metrics
      metrics.recordHttpRequest(
        req.method,
        req.route?.path || req.path,
        res.statusCode,
        duration,
        platform as string
      );

      // Add span attributes
      span.setAttributes({
        'http.status_code': res.statusCode,
        'http.response_size': res.get('content-length') || 0
      });

      // Log request completion
      if (res.statusCode >= 400) {
        logger.error('BFF request failed', undefined, {
          operation,
          statusCode: res.statusCode,
          duration,
          platform
        });

        tracer.setSpanError(span, new Error(`HTTP ${res.statusCode}`));
        metrics.recordError('http_error', 'bff', operation);
      } else {
        logger.info('BFF request completed', {
          operation,
          statusCode: res.statusCode,
          duration,
          platform
        });
      }

      tracer.endSpan(span);
      originalEnd.call(this, chunk, encoding);
    };

    next();
  };
}
```

### 2. Performance Optimization Strategies
```typescript
// BFF Performance Optimization
import NodeCache from 'node-cache';
import Redis from 'ioredis';
import { promisify } from 'util';

// Multi-level Caching Strategy
class BFFCacheManager {
  private l1Cache: NodeCache; // In-memory cache
  private l2Cache: Redis; // Redis cache
  private metrics: BFFMetrics;
  private logger: BFFLogger;

  constructor(config: CacheConfig, metrics: BFFMetrics, logger: BFFLogger) {
    this.l1Cache = new NodeCache({
      stdTTL: config.l1TTL || 60,
      maxKeys: config.l1MaxKeys || 1000,
      useClones: false
    });

    this.l2Cache = new Redis({
      host: config.redisHost || 'localhost',
      port: config.redisPort || 6379,
      retryDelayOnFailover: 100,
      maxRetriesPerRequest: 3,
      lazyConnect: true
    });

    this.metrics = metrics;
    this.logger = logger;
    this.setupCacheEventHandlers();
  }

  async get<T>(key: string): Promise<T | null> {
    const startTime = process.hrtime.bigint();
    
    try {
      // Try L1 cache first
      const l1Result = this.l1Cache.get<T>(key);
      if (l1Result !== undefined) {
        this.recordCacheHit('l1', key, startTime);
        return l1Result;
      }

      // Try L2 cache
      const l2Result = await this.l2Cache.get(key);
      if (l2Result) {
        const parsed = JSON.parse(l2Result) as T;
        
        // Populate L1 cache
        this.l1Cache.set(key, parsed);
        
        this.recordCacheHit('l2', key, startTime);
        return parsed;
      }

      this.recordCacheMiss(key, startTime);
      return null;
    } catch (error) {
      this.logger.error('Cache get error', error as Error, { key });
      this.recordCacheMiss(key, startTime);
      return null;
    }
  }

  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    try {
      // Set in L1 cache
      this.l1Cache.set(key, value, ttl || 60);

      // Set in L2 cache
      await this.l2Cache.setex(key, ttl || 300, JSON.stringify(value));

      this.logger.debug('Cache set', { key, ttl });
    } catch (error) {
      this.logger.error('Cache set error', error as Error, { key });
    }
  }

  async delete(key: string): Promise<void> {
    try {
      this.l1Cache.del(key);
      await this.l2Cache.del(key);
      this.logger.debug('Cache delete', { key });
    } catch (error) {
      this.logger.error('Cache delete error', error as Error, { key });
    }
  }

  async clear(): Promise<void> {
    try {
      this.l1Cache.flushAll();
      await this.l2Cache.flushall();
      this.logger.info('Cache cleared');
    } catch (error) {
      this.logger.error('Cache clear error', error as Error);
    }
  }

  private setupCacheEventHandlers(): void {
    this.l1Cache.on('expired', (key, value) => {
      this.logger.debug('L1 cache key expired', { key });
    });

    this.l1Cache.on('del', (key, value) => {
      this.logger.debug('L1 cache key deleted', { key });
    });

    this.l2Cache.on('error', (error) => {
      this.logger.error('L2 cache error', error);
    });
  }

  private recordCacheHit(level: string, key: string, startTime: bigint): void {
    const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000; // ms
    this.logger.debug('Cache hit', { level, key, duration });
    this.metrics.updateCacheHitRate(level, 'get', 1);
  }

  private recordCacheMiss(key: string, startTime: bigint): void {
    const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000; // ms
    this.logger.debug('Cache miss', { key, duration });
    this.metrics.updateCacheHitRate('combined', 'get', 0);
  }
}

// Request Batching and Deduplication
class BFFRequestOptimizer {
  private pendingRequests: Map<string, Promise<any>>;
  private batchQueues: Map<string, BatchQueue>;
  private metrics: BFFMetrics;
  private logger: BFFLogger;

  constructor(metrics: BFFMetrics, logger: BFFLogger) {
    this.pendingRequests = new Map();
    this.batchQueues = new Map();
    this.metrics = metrics;
    this.logger = logger;
  }

  // Deduplicate identical requests
  async deduplicate<T>(key: string, fn: () => Promise<T>): Promise<T> {
    if (this.pendingRequests.has(key)) {
      this.logger.debug('Request deduplicated', { key });
      return this.pendingRequests.get(key) as Promise<T>;
    }

    const promise = fn().finally(() => {
      this.pendingRequests.delete(key);
    });

    this.pendingRequests.set(key, promise);
    return promise;
  }

  // Batch similar requests
  async batch<T>(
    queueName: string,
    item: T,
    batchProcessor: (items: T[]) => Promise<any[]>,
    options: BatchOptions = {}
  ): Promise<any> {
    if (!this.batchQueues.has(queueName)) {
      this.batchQueues.set(queueName, new BatchQueue(
        batchProcessor,
        options,
        this.metrics,
        this.logger
      ));
    }

    const queue = this.batchQueues.get(queueName)!;
    return queue.add(item);
  }
}

class BatchQueue<T> {
  private items: Array<{ item: T; resolve: (value: any) => void; reject: (error: any) => void }> = [];
  private timer: NodeJS.Timeout | null = null;
  private processing = false;

  constructor(
    private processor: (items: T[]) => Promise<any[]>,
    private options: BatchOptions,
    private metrics: BFFMetrics,
    private logger: BFFLogger
  ) {}

  add(item: T): Promise<any> {
    return new Promise((resolve, reject) => {
      this.items.push({ item, resolve, reject });

      if (this.items.length >= (this.options.maxBatchSize || 10)) {
        this.processBatch();
      } else if (!this.timer) {
        this.timer = setTimeout(() => {
          this.processBatch();
        }, this.options.maxWaitTime || 50);
      }
    });
  }

  private async processBatch(): Promise<void> {
    if (this.processing || this.items.length === 0) return;

    this.processing = true;
    
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    const batch = this.items.splice(0);
    const startTime = process.hrtime.bigint();

    try {
      const results = await this.processor(batch.map(b => b.item));
      
      batch.forEach((item, index) => {
        item.resolve(results[index]);
      });

      const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000_000;
      this.metrics.recordBFFProcessing('batch_process', 'all', duration, 'none');
      this.logger.debug('Batch processed successfully', { 
        batchSize: batch.length, 
        duration 
      });
    } catch (error) {
      batch.forEach(item => {
        item.reject(error);
      });

      this.logger.error('Batch processing failed', error as Error, { 
        batchSize: batch.length 
      });
      this.metrics.recordError('batch_error', 'bff', 'batch_process');
    } finally {
      this.processing = false;
    }
  }
}

// Connection Pooling and Circuit Breaker
import CircuitBreaker from 'opossum';
import { Pool } from 'generic-pool';

class BFFServiceClient {
  private circuitBreaker: CircuitBreaker<any[], any>;
  private connectionPool: Pool<any>;
  private metrics: BFFMetrics;
  private logger: BFFLogger;

  constructor(
    serviceName: string,
    config: ServiceClientConfig,
    metrics: BFFMetrics,
    logger: BFFLogger
  ) {
    this.metrics = metrics;
    this.logger = logger;
    
    this.setupConnectionPool(config);
    this.setupCircuitBreaker(serviceName, config);
  }

  private setupConnectionPool(config: ServiceClientConfig): void {
    const factory = {
      create: async () => {
        // Create HTTP agent or connection
        return new (require('http').Agent)({
          keepAlive: true,
          maxSockets: config.maxConnections || 50,
          timeout: config.connectionTimeout || 5000
        });
      },
      destroy: async (connection: any) => {
        connection.destroy();
      },
      validate: async (connection: any) => {
        return connection && !connection.destroyed;
      }
    };

    this.connectionPool = require('generic-pool').createPool(factory, {
      max: config.maxConnections || 50,
      min: config.minConnections || 5,
      acquireTimeoutMillis: config.acquireTimeout || 3000,
      idleTimeoutMillis: config.idleTimeout || 30000
    });
  }

  private setupCircuitBreaker(serviceName: string, config: ServiceClientConfig): void {
    const options = {
      timeout: config.requestTimeout || 5000,
      errorThresholdPercentage: config.errorThreshold || 50,
      resetTimeout: config.resetTimeout || 30000,
      rollingCountTimeout: config.rollingWindow || 10000,
      rollingCountBuckets: 10,
      name: serviceName,
      group: 'bff-services'
    };

    this.circuitBreaker = new CircuitBreaker(
      this.makeRequest.bind(this),
      options
    );

    // Circuit breaker event handlers
    this.circuitBreaker.on('open', () => {
      this.logger.warn('Circuit breaker opened', { service: serviceName });
      this.metrics.recordError('circuit_open', serviceName, 'request');
    });

    this.circuitBreaker.on('halfOpen', () => {
      this.logger.info('Circuit breaker half-open', { service: serviceName });
    });

    this.circuitBreaker.on('close', () => {
      this.logger.info('Circuit breaker closed', { service: serviceName });
    });

    this.circuitBreaker.on('reject', () => {
      this.metrics.recordError('circuit_reject', serviceName, 'request');
    });
  }

  async request<T>(options: RequestOptions): Promise<T> {
    const startTime = process.hrtime.bigint();
    
    try {
      const result = await this.circuitBreaker.fire(options);
      
      const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000_000;
      this.metrics.recordServiceCall(
        options.service || 'unknown',
        options.method || 'GET',
        'success',
        duration
      );

      return result;
    } catch (error) {
      const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000_000;
      this.metrics.recordServiceCall(
        options.service || 'unknown',
        options.method || 'GET',
        'error',
        duration
      );

      throw error;
    }
  }

  private async makeRequest(options: RequestOptions): Promise<any> {
    const connection = await this.connectionPool.acquire();
    
    try {
      // Make actual HTTP request using connection
      const response = await fetch(options.url, {
        method: options.method,
        headers: options.headers,
        body: options.body,
        agent: connection
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return await response.json();
    } finally {
      await this.connectionPool.release(connection);
    }
  }
}
```

### 3. Health Checks and Alerting
```typescript
// Health Check System
class BFFHealthChecker {
  private checks: Map<string, HealthCheck>;
  private metrics: BFFMetrics;
  private logger: BFFLogger;

  constructor(metrics: BFFMetrics, logger: BFFLogger) {
    this.checks = new Map();
    this.metrics = metrics;
    this.logger = logger;
  }

  addCheck(name: string, check: HealthCheck): void {
    this.checks.set(name, check);
  }

  async runChecks(): Promise<HealthStatus> {
    const results: HealthCheckResult[] = [];
    let overallStatus: 'healthy' | 'degraded' | 'unhealthy' = 'healthy';

    for (const [name, check] of this.checks) {
      try {
        const startTime = process.hrtime.bigint();
        const result = await check.execute();
        const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000;

        results.push({
          name,
          status: result.healthy ? 'healthy' : 'unhealthy',
          message: result.message,
          duration,
          details: result.details
        });

        if (!result.healthy) {
          if (check.critical) {
            overallStatus = 'unhealthy';
          } else if (overallStatus === 'healthy') {
            overallStatus = 'degraded';
          }
        }
      } catch (error) {
        results.push({
          name,
          status: 'unhealthy',
          message: (error as Error).message,
          duration: 0,
          details: { error: (error as Error).stack }
        });

        if (check.critical) {
          overallStatus = 'unhealthy';
        } else if (overallStatus === 'healthy') {
          overallStatus = 'degraded';
        }
      }
    }

    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      checks: results,
      version: process.env.APP_VERSION || 'unknown'
    };
  }

  // Express middleware for health endpoint
  getHealthMiddleware() {
    return async (req: Request, res: Response) => {
      const health = await this.runChecks();
      const statusCode = health.status === 'healthy' ? 200 : 
                        health.status === 'degraded' ? 200 : 503;
      
      res.status(statusCode).json(health);
    };
  }
}

// Built-in Health Checks
class DatabaseHealthCheck implements HealthCheck {
  critical = true;

  constructor(private database: any) {}

  async execute(): Promise<{ healthy: boolean; message: string; details?: any }> {
    try {
      await this.database.query('SELECT 1');
      return { healthy: true, message: 'Database connection healthy' };
    } catch (error) {
      return { 
        healthy: false, 
        message: 'Database connection failed',
        details: { error: (error as Error).message }
      };
    }
  }
}

class RedisHealthCheck implements HealthCheck {
  critical = false;

  constructor(private redis: Redis) {}

  async execute(): Promise<{ healthy: boolean; message: string; details?: any }> {
    try {
      const result = await this.redis.ping();
      return { 
        healthy: result === 'PONG', 
        message: result === 'PONG' ? 'Redis healthy' : 'Redis ping failed' 
      };
    } catch (error) {
      return { 
        healthy: false, 
        message: 'Redis connection failed',
        details: { error: (error as Error).message }
      };
    }
  }
}

class ServiceHealthCheck implements HealthCheck {
  critical = false;

  constructor(private serviceName: string, private serviceUrl: string) {}

  async execute(): Promise<{ healthy: boolean; message: string; details?: any }> {
    try {
      const response = await fetch(`${this.serviceUrl}/health`, {
        timeout: 5000
      });

      if (response.ok) {
        return { healthy: true, message: `${this.serviceName} healthy` };
      } else {
        return { 
          healthy: false, 
          message: `${this.serviceName} unhealthy`,
          details: { status: response.status, statusText: response.statusText }
        };
      }
    } catch (error) {
      return { 
        healthy: false, 
        message: `${this.serviceName} unreachable`,
        details: { error: (error as Error).message }
      };
    }
  }
}

// Type Definitions
interface CacheConfig {
  l1TTL?: number;
  l1MaxKeys?: number;
  redisHost?: string;
  redisPort?: number;
}

interface BatchOptions {
  maxBatchSize?: number;
  maxWaitTime?: number;
}

interface ServiceClientConfig {
  maxConnections?: number;
  minConnections?: number;
  connectionTimeout?: number;
  acquireTimeout?: number;
  idleTimeout?: number;
  requestTimeout?: number;
  errorThreshold?: number;
  resetTimeout?: number;
  rollingWindow?: number;
}

interface RequestOptions {
  url: string;
  method?: string;
  headers?: Record<string, string>;
  body?: any;
  service?: string;
}

interface HealthCheck {
  critical: boolean;
  execute(): Promise<{ healthy: boolean; message: string; details?: any }>;
}

interface HealthCheckResult {
  name: string;
  status: 'healthy' | 'unhealthy';
  message: string;
  duration: number;
  details?: any;
}

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  checks: HealthCheckResult[];
  version: string;
}

export {
  BFFMetrics,
  BFFTracing,
  BFFLogger,
  BFFCacheManager,
  BFFRequestOptimizer,
  BFFServiceClient,
  BFFHealthChecker,
  createObservabilityMiddleware
};
```

## Interview-Ready Monitoring Summary

**BFF Monitoring & Observability covers:**

1. **Metrics Collection** - Prometheus metrics for HTTP requests, service calls, cache performance
2. **Distributed Tracing** - OpenTelemetry integration with Jaeger for request flow tracking
3. **Structured Logging** - Winston with Elasticsearch integration for log aggregation
4. **Performance Optimization** - Multi-level caching, request batching, connection pooling
5. **Health Monitoring** - Comprehensive health checks with circuit breaker patterns

**Key Monitoring Patterns:**
- **Three Pillars** - Metrics, traces, and logs integrated across all BFF operations
- **Performance Optimization** - Caching strategies, request deduplication, batching
- **Reliability Patterns** - Circuit breakers, health checks, graceful degradation
- **Observability Context** - Trace correlation across all logs and metrics

**Production Benefits:**
- **Proactive Monitoring** - Early detection of performance issues and failures
- **Root Cause Analysis** - Distributed tracing for complex request flows
- **Performance Optimization** - Data-driven optimization with comprehensive metrics
- **Operational Excellence** - Health checks, alerting, and automated recovery

**Enterprise Features:** Elasticsearch log aggregation, Prometheus metrics, Jaeger tracing, Redis caching, circuit breaker resilience, comprehensive health monitoring.