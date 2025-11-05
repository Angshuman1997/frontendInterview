# Enterprise API Design and Microservices Communication

Modern enterprise applications require sophisticated API design patterns and robust microservices communication strategies. This guide covers advanced API architecture, service mesh integration, and enterprise-grade communication patterns.

## Advanced API Design Patterns

### 1. Enterprise API Gateway Implementation
```typescript
// Enterprise API Gateway with Kong/Express Gateway patterns
import express, { Request, Response, NextFunction } from 'express';
import rateLimit from 'express-rate-limit';
import jwt from 'jsonwebtoken';
import redis from 'redis';
import { createProxyMiddleware, Options } from 'http-proxy-middleware';
import CircuitBreaker from 'opossum';
import { Tracer } from 'jaeger-client';

// API Gateway Configuration
interface GatewayConfig {
  services: ServiceConfig[];
  rateLimiting: RateLimitConfig;
  authentication: AuthConfig;
  monitoring: MonitoringConfig;
  circuitBreaker: CircuitBreakerConfig;
}

interface ServiceConfig {
  name: string;
  path: string;
  target: string;
  healthCheck: string;
  timeout: number;
  retries: number;
  rateLimit?: RateLimitConfig;
  authentication?: boolean;
  circuitBreaker?: CircuitBreakerConfig;
  transformations?: {
    request?: RequestTransformation[];
    response?: ResponseTransformation[];
  };
}

// Advanced API Gateway Class
class EnterpriseAPIGateway {
  private app: express.Application;
  private redisClient: any;
  private tracer: Tracer;
  private circuitBreakers: Map<string, CircuitBreaker>;
  private serviceRegistry: ServiceRegistry;

  constructor(private config: GatewayConfig) {
    this.app = express();
    this.circuitBreakers = new Map();
    this.serviceRegistry = new ServiceRegistry();
    this.initializeRedis();
    this.initializeTracing();
    this.setupMiddleware();
    this.setupRoutes();
  }

  private initializeRedis(): void {
    this.redisClient = redis.createClient({
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379'),
      password: process.env.REDIS_PASSWORD,
      retry_strategy: (options) => {
        if (options.error && options.error.code === 'ECONNREFUSED') {
          return new Error('Redis server connection refused');
        }
        if (options.total_retry_time > 1000 * 60 * 60) {
          return new Error('Redis retry time exhausted');
        }
        return Math.min(options.attempt * 100, 3000);
      }
    });
  }

  private initializeTracing(): void {
    const jaegerConfig = {
      serviceName: 'api-gateway',
      sampler: {
        type: 'const',
        param: 1
      },
      reporter: {
        logSpans: true,
        agentHost: process.env.JAEGER_AGENT_HOST || 'localhost',
        agentPort: parseInt(process.env.JAEGER_AGENT_PORT || '6832')
      }
    };

    this.tracer = new Tracer(jaegerConfig);
  }

  private setupMiddleware(): void {
    // Security headers
    this.app.use((req: Request, res: Response, next: NextFunction) => {
      res.setHeader('X-Content-Type-Options', 'nosniff');
      res.setHeader('X-Frame-Options', 'DENY');
      res.setHeader('X-XSS-Protection', '1; mode=block');
      res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
      next();
    });

    // Request correlation ID
    this.app.use((req: Request, res: Response, next: NextFunction) => {
      const correlationId = req.headers['x-correlation-id'] || this.generateCorrelationId();
      req.headers['x-correlation-id'] = correlationId;
      res.setHeader('x-correlation-id', correlationId);
      next();
    });

    // Distributed tracing
    this.app.use((req: Request, res: Response, next: NextFunction) => {
      const span = this.tracer.startSpan(`${req.method} ${req.path}`);
      span.setTag('http.method', req.method);
      span.setTag('http.url', req.url);
      span.setTag('correlation.id', req.headers['x-correlation-id']);
      
      req.span = span;
      
      res.on('finish', () => {
        span.setTag('http.status_code', res.statusCode);
        span.finish();
      });
      
      next();
    });

    // Global rate limiting
    this.app.use(rateLimit({
      windowMs: this.config.rateLimiting.windowMs,
      max: this.config.rateLimiting.max,
      keyGenerator: (req) => {
        return req.ip + ':' + (req.headers['x-api-key'] || 'anonymous');
      },
      store: new RedisStore({
        client: this.redisClient,
        prefix: 'rate_limit:'
      }),
      handler: (req, res) => {
        res.status(429).json({
          error: 'Too Many Requests',
          message: 'Rate limit exceeded',
          retryAfter: Math.ceil(this.config.rateLimiting.windowMs / 1000)
        });
      }
    }));

    // Request logging
    this.app.use((req: Request, res: Response, next: NextFunction) => {
      const start = Date.now();
      
      res.on('finish', () => {
        const duration = Date.now() - start;
        console.log({
          timestamp: new Date().toISOString(),
          method: req.method,
          path: req.path,
          statusCode: res.statusCode,
          duration,
          correlationId: req.headers['x-correlation-id'],
          userAgent: req.headers['user-agent'],
          ip: req.ip
        });
      });
      
      next();
    });
  }

  private setupRoutes(): void {
    // Health check endpoint
    this.app.get('/health', async (req: Request, res: Response) => {
      const healthChecks = await Promise.allSettled(
        this.config.services.map(service => this.checkServiceHealth(service))
      );

      const services = this.config.services.map((service, index) => ({
        name: service.name,
        status: healthChecks[index].status === 'fulfilled' ? 'healthy' : 'unhealthy',
        ...(healthChecks[index].status === 'rejected' && {
          error: (healthChecks[index] as PromiseRejectedResult).reason.message
        })
      }));

      const overallHealth = services.every(service => service.status === 'healthy');

      res.status(overallHealth ? 200 : 503).json({
        status: overallHealth ? 'healthy' : 'unhealthy',
        services,
        timestamp: new Date().toISOString()
      });
    });

    // Service discovery endpoint
    this.app.get('/services', async (req: Request, res: Response) => {
      const services = await this.serviceRegistry.getAllServices();
      res.json({
        services: services.map(service => ({
          name: service.name,
          version: service.version,
          endpoints: service.endpoints,
          health: service.health
        }))
      });
    });

    // Setup service proxies
    this.config.services.forEach(service => {
      this.setupServiceProxy(service);
    });

    // 404 handler
    this.app.use((req: Request, res: Response) => {
      res.status(404).json({
        error: 'Not Found',
        message: `Endpoint ${req.method} ${req.path} not found`,
        availableServices: this.config.services.map(s => s.path)
      });
    });
  }

  private setupServiceProxy(service: ServiceConfig): void {
    // Create circuit breaker for service
    const circuitBreaker = new CircuitBreaker(
      async (req: Request) => this.proxyRequest(req, service),
      {
        timeout: service.timeout || 10000,
        errorThresholdPercentage: service.circuitBreaker?.errorThreshold || 50,
        resetTimeout: service.circuitBreaker?.resetTimeout || 30000
      }
    );

    circuitBreaker.on('open', () => {
      console.warn(`Circuit breaker opened for service: ${service.name}`);
    });

    circuitBreaker.on('halfOpen', () => {
      console.info(`Circuit breaker half-open for service: ${service.name}`);
    });

    this.circuitBreakers.set(service.name, circuitBreaker);

    // Service-specific rate limiting
    if (service.rateLimit) {
      this.app.use(service.path, rateLimit({
        windowMs: service.rateLimit.windowMs,
        max: service.rateLimit.max,
        keyGenerator: (req) => `${service.name}:${req.ip}:${req.headers['x-api-key'] || 'anonymous'}`
      }));
    }

    // Authentication middleware
    if (service.authentication) {
      this.app.use(service.path, this.authenticateRequest.bind(this));
    }

    // Request transformation
    if (service.transformations?.request) {
      this.app.use(service.path, (req: Request, res: Response, next: NextFunction) => {
        service.transformations!.request!.forEach(transformation => {
          req = this.applyRequestTransformation(req, transformation);
        });
        next();
      });
    }

    // Main proxy middleware
    this.app.use(service.path, async (req: Request, res: Response, next: NextFunction) => {
      try {
        const result = await circuitBreaker.fire(req);
        
        // Apply response transformations
        if (service.transformations?.response) {
          service.transformations.response.forEach(transformation => {
            result.data = this.applyResponseTransformation(result.data, transformation);
          });
        }

        res.status(result.status).json(result.data);
      } catch (error) {
        if (error.message === 'Circuit breaker is open') {
          res.status(503).json({
            error: 'Service Unavailable',
            message: `Service ${service.name} is currently unavailable`,
            service: service.name
          });
        } else {
          next(error);
        }
      }
    });
  }

  private async proxyRequest(req: Request, service: ServiceConfig): Promise<any> {
    const target = await this.serviceRegistry.getServiceEndpoint(service.name);
    const url = `${target}${req.path.replace(service.path, '')}`;
    
    const requestOptions = {
      method: req.method,
      headers: {
        ...req.headers,
        'x-forwarded-for': req.ip,
        'x-service-name': service.name
      },
      ...(req.body && { body: JSON.stringify(req.body) })
    };

    const response = await fetch(url, requestOptions);
    const data = await response.json();

    return {
      status: response.status,
      data
    };
  }

  private async authenticateRequest(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const token = req.headers.authorization?.replace('Bearer ', '');
      
      if (!token) {
        res.status(401).json({ error: 'Authentication required' });
        return;
      }

      // Verify JWT token
      const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any;
      
      // Check token in Redis cache for revocation
      const isRevoked = await this.redisClient.get(`revoked_token:${token}`);
      if (isRevoked) {
        res.status(401).json({ error: 'Token revoked' });
        return;
      }

      req.user = decoded;
      req.headers['x-user-id'] = decoded.sub;
      req.headers['x-user-roles'] = decoded.roles?.join(',');
      
      next();
    } catch (error) {
      res.status(401).json({ error: 'Invalid authentication token' });
    }
  }

  private applyRequestTransformation(req: Request, transformation: RequestTransformation): Request {
    switch (transformation.type) {
      case 'ADD_HEADER':
        req.headers[transformation.header!] = transformation.value!;
        break;
      case 'REMOVE_HEADER':
        delete req.headers[transformation.header!];
        break;
      case 'MODIFY_PATH':
        req.path = req.path.replace(transformation.pattern!, transformation.replacement!);
        break;
      case 'ADD_QUERY_PARAM':
        req.query[transformation.param!] = transformation.value!;
        break;
    }
    return req;
  }

  private applyResponseTransformation(data: any, transformation: ResponseTransformation): any {
    switch (transformation.type) {
      case 'REMOVE_FIELD':
        delete data[transformation.field!];
        break;
      case 'RENAME_FIELD':
        if (data[transformation.from!]) {
          data[transformation.to!] = data[transformation.from!];
          delete data[transformation.from!];
        }
        break;
      case 'ADD_FIELD':
        data[transformation.field!] = transformation.value!;
        break;
      case 'TRANSFORM_FIELD':
        if (data[transformation.field!] && transformation.transformer) {
          data[transformation.field!] = transformation.transformer(data[transformation.field!]);
        }
        break;
    }
    return data;
  }

  private async checkServiceHealth(service: ServiceConfig): Promise<boolean> {
    try {
      const endpoint = await this.serviceRegistry.getServiceEndpoint(service.name);
      const response = await fetch(`${endpoint}${service.healthCheck}`);
      return response.ok;
    } catch (error) {
      throw new Error(`Health check failed for ${service.name}: ${error.message}`);
    }
  }

  private generateCorrelationId(): string {
    return Math.random().toString(36).substring(2, 15) + 
           Math.random().toString(36).substring(2, 15);
  }

  public start(port: number = 3000): void {
    this.app.listen(port, () => {
      console.log(`API Gateway running on port ${port}`);
      console.log(`Services: ${this.config.services.map(s => s.name).join(', ')}`);
    });
  }
}

// Service Registry for Dynamic Service Discovery
class ServiceRegistry {
  private services: Map<string, ServiceInstance> = new Map();
  private redisClient: any;

  constructor() {
    this.redisClient = redis.createClient();
    this.startHealthChecking();
  }

  async registerService(service: ServiceInstance): Promise<void> {
    this.services.set(service.name, service);
    
    // Store in Redis for persistence
    await this.redisClient.hset(
      'service_registry',
      service.name,
      JSON.stringify(service)
    );

    console.log(`Service registered: ${service.name} at ${service.endpoint}`);
  }

  async unregisterService(serviceName: string): Promise<void> {
    this.services.delete(serviceName);
    await this.redisClient.hdel('service_registry', serviceName);
    console.log(`Service unregistered: ${serviceName}`);
  }

  async getServiceEndpoint(serviceName: string): Promise<string> {
    const service = this.services.get(serviceName);
    
    if (!service) {
      throw new Error(`Service not found: ${serviceName}`);
    }

    if (!service.healthy) {
      throw new Error(`Service unhealthy: ${serviceName}`);
    }

    return service.endpoint;
  }

  async getAllServices(): Promise<ServiceInstance[]> {
    return Array.from(this.services.values());
  }

  private async startHealthChecking(): void {
    setInterval(async () => {
      const healthChecks = Array.from(this.services.values()).map(async (service) => {
        try {
          const response = await fetch(`${service.endpoint}/health`, { timeout: 5000 });
          service.healthy = response.ok;
          service.lastHealthCheck = new Date();
        } catch (error) {
          service.healthy = false;
          service.lastHealthCheck = new Date();
          console.warn(`Health check failed for ${service.name}: ${error.message}`);
        }
      });

      await Promise.allSettled(healthChecks);
    }, 30000); // Check every 30 seconds
  }
}
```

### 2. Advanced Microservices Communication Patterns
```typescript
// Event-Driven Microservices Communication
import { EventEmitter } from 'events';
import amqp from 'amqplib';
import { Kafka } from 'kafkajs';

// Message Bus Interface
interface MessageBus {
  publish(topic: string, message: any, options?: PublishOptions): Promise<void>;
  subscribe(topic: string, handler: MessageHandler): Promise<void>;
  request(service: string, method: string, data: any): Promise<any>;
  disconnect(): Promise<void>;
}

interface PublishOptions {
  partition?: string;
  key?: string;
  headers?: Record<string, string>;
  timeout?: number;
}

type MessageHandler = (message: any, metadata: MessageMetadata) => Promise<void>;

interface MessageMetadata {
  topic: string;
  partition?: number;
  offset?: number;
  timestamp: Date;
  correlationId: string;
  headers: Record<string, string>;
}

// Kafka Implementation
class KafkaMessageBus implements MessageBus {
  private kafka: Kafka;
  private producer: any;
  private consumer: any;
  private admin: any;
  private subscriptions: Map<string, MessageHandler> = new Map();

  constructor(private config: KafkaConfig) {
    this.kafka = new Kafka({
      clientId: config.clientId,
      brokers: config.brokers,
      retry: {
        initialRetryTime: 100,
        retries: 8
      }
    });

    this.producer = this.kafka.producer({
      maxInFlightRequests: 1,
      idempotent: true,
      transactionTimeout: 30000
    });

    this.consumer = this.kafka.consumer({
      groupId: config.groupId,
      sessionTimeout: 30000,
      rebalanceTimeout: 60000,
      heartbeatInterval: 3000
    });

    this.admin = this.kafka.admin();
  }

  async initialize(): Promise<void> {
    await this.producer.connect();
    await this.consumer.connect();
    await this.admin.connect();

    // Create topics if they don't exist
    await this.createTopics();

    // Set up consumer message handling
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const handler = this.subscriptions.get(topic);
        if (handler) {
          const metadata: MessageMetadata = {
            topic,
            partition,
            offset: parseInt(message.offset),
            timestamp: new Date(parseInt(message.timestamp)),
            correlationId: message.headers?.correlationId?.toString() || '',
            headers: this.parseHeaders(message.headers)
          };

          try {
            const messageData = JSON.parse(message.value?.toString() || '{}');
            await handler(messageData, metadata);
          } catch (error) {
            console.error(`Error processing message from topic ${topic}:`, error);
            // Implement dead letter queue logic here
            await this.handleFailedMessage(topic, message, error);
          }
        }
      }
    });
  }

  async publish(topic: string, message: any, options: PublishOptions = {}): Promise<void> {
    const messageToSend = {
      topic,
      key: options.key,
      value: JSON.stringify(message),
      partition: options.partition ? parseInt(options.partition) : undefined,
      headers: {
        ...options.headers,
        correlationId: this.generateCorrelationId(),
        timestamp: new Date().toISOString()
      }
    };

    await this.producer.send({
      topic,
      messages: [messageToSend]
    });
  }

  async subscribe(topic: string, handler: MessageHandler): Promise<void> {
    this.subscriptions.set(topic, handler);
    await this.consumer.subscribe({ topic, fromBeginning: false });
  }

  async request(service: string, method: string, data: any): Promise<any> {
    const correlationId = this.generateCorrelationId();
    const responseTopicName = `${service}_response_${correlationId}`;
    
    // Create temporary topic for response
    await this.admin.createTopics({
      topics: [{
        topic: responseTopicName,
        numPartitions: 1,
        replicationFactor: 1,
        configEntries: [
          { name: 'cleanup.policy', value: 'delete' },
          { name: 'retention.ms', value: '60000' } // 1 minute retention
        ]
      }]
    });

    // Set up response consumer
    const responseConsumer = this.kafka.consumer({
      groupId: `response_${correlationId}`
    });
    
    await responseConsumer.connect();
    await responseConsumer.subscribe({ topic: responseTopicName });

    // Send request
    await this.publish(`${service}_requests`, {
      method,
      data,
      responseTopicName,
      correlationId
    });

    // Wait for response
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(new Error('Request timeout'));
      }, 30000);

      responseConsumer.run({
        eachMessage: async ({ message }) => {
          clearTimeout(timeout);
          await responseConsumer.disconnect();
          
          try {
            const response = JSON.parse(message.value?.toString() || '{}');
            if (response.error) {
              reject(new Error(response.error));
            } else {
              resolve(response.data);
            }
          } catch (error) {
            reject(error);
          }
        }
      });
    });
  }

  private async createTopics(): Promise<void> {
    const topics = [
      'user_events',
      'order_events',
      'notification_events',
      'analytics_events',
      'audit_events'
    ];

    await this.admin.createTopics({
      topics: topics.map(topic => ({
        topic,
        numPartitions: 3,
        replicationFactor: 1
      }))
    });
  }

  private parseHeaders(headers: any): Record<string, string> {
    const parsed: Record<string, string> = {};
    if (headers) {
      Object.keys(headers).forEach(key => {
        parsed[key] = headers[key].toString();
      });
    }
    return parsed;
  }

  private async handleFailedMessage(topic: string, message: any, error: Error): Promise<void> {
    const deadLetterTopic = `${topic}_dead_letter`;
    
    await this.publish(deadLetterTopic, {
      originalMessage: message.value?.toString(),
      error: error.message,
      timestamp: new Date().toISOString(),
      originalTopic: topic
    });
  }

  private generateCorrelationId(): string {
    return Math.random().toString(36).substring(2, 15) + Date.now().toString(36);
  }

  async disconnect(): Promise<void> {
    await this.producer.disconnect();
    await this.consumer.disconnect();
    await this.admin.disconnect();
  }
}

// RabbitMQ Implementation
class RabbitMQMessageBus implements MessageBus {
  private connection: amqp.Connection | null = null;
  private channel: amqp.Channel | null = null;
  private subscriptions: Map<string, MessageHandler> = new Map();

  constructor(private config: RabbitMQConfig) {}

  async initialize(): Promise<void> {
    this.connection = await amqp.connect({
      protocol: 'amqp',
      hostname: this.config.host,
      port: this.config.port,
      username: this.config.username,
      password: this.config.password,
      vhost: this.config.vhost || '/',
      heartbeat: 60
    });

    this.channel = await this.connection.createChannel();
    
    // Set up exchanges and queues
    await this.setupExchangesAndQueues();

    this.connection.on('error', (error) => {
      console.error('RabbitMQ connection error:', error);
      this.reconnect();
    });

    this.connection.on('close', () => {
      console.warn('RabbitMQ connection closed, attempting to reconnect...');
      this.reconnect();
    });
  }

  async publish(topic: string, message: any, options: PublishOptions = {}): Promise<void> {
    if (!this.channel) throw new Error('RabbitMQ not initialized');

    const messageBuffer = Buffer.from(JSON.stringify(message));
    const correlationId = this.generateCorrelationId();

    await this.channel.publish(
      'events_exchange',
      topic,
      messageBuffer,
      {
        correlationId,
        timestamp: Date.now(),
        headers: {
          ...options.headers,
          correlationId
        },
        persistent: true
      }
    );
  }

  async subscribe(topic: string, handler: MessageHandler): Promise<void> {
    if (!this.channel) throw new Error('RabbitMQ not initialized');

    this.subscriptions.set(topic, handler);
    
    const queueName = `${topic}_queue`;
    await this.channel.assertQueue(queueName, { durable: true });
    await this.channel.bindQueue(queueName, 'events_exchange', topic);

    await this.channel.consume(queueName, async (message) => {
      if (!message) return;

      const metadata: MessageMetadata = {
        topic,
        timestamp: new Date(message.properties.timestamp),
        correlationId: message.properties.correlationId || '',
        headers: message.properties.headers || {}
      };

      try {
        const messageData = JSON.parse(message.content.toString());
        await handler(messageData, metadata);
        this.channel!.ack(message);
      } catch (error) {
        console.error(`Error processing message from topic ${topic}:`, error);
        
        // Retry logic
        const retryCount = (message.properties.headers?.retryCount || 0) + 1;
        if (retryCount <= 3) {
          // Requeue with delay
          setTimeout(() => {
            this.channel!.publish(
              'events_exchange',
              topic,
              message.content,
              {
                ...message.properties,
                headers: {
                  ...message.properties.headers,
                  retryCount
                }
              }
            );
          }, 1000 * retryCount);
        } else {
          // Send to dead letter queue
          await this.sendToDeadLetterQueue(topic, message, error);
        }
        
        this.channel!.ack(message);
      }
    });
  }

  async request(service: string, method: string, data: any): Promise<any> {
    if (!this.channel) throw new Error('RabbitMQ not initialized');

    const correlationId = this.generateCorrelationId();
    const replyQueue = `rpc_reply_${correlationId}`;
    
    // Create temporary reply queue
    await this.channel.assertQueue(replyQueue, { exclusive: true, autoDelete: true });

    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(new Error('RPC request timeout'));
      }, 30000);

      // Set up reply consumer
      this.channel!.consume(replyQueue, (message) => {
        if (message && message.properties.correlationId === correlationId) {
          clearTimeout(timeout);
          
          try {
            const response = JSON.parse(message.content.toString());
            if (response.error) {
              reject(new Error(response.error));
            } else {
              resolve(response.data);
            }
          } catch (error) {
            reject(error);
          }
          
          this.channel!.ack(message);
        }
      }, { noAck: false });

      // Send request
      this.channel!.sendToQueue(
        `${service}_rpc_queue`,
        Buffer.from(JSON.stringify({ method, data })),
        {
          correlationId,
          replyTo: replyQueue,
          timestamp: Date.now()
        }
      );
    });
  }

  private async setupExchangesAndQueues(): Promise<void> {
    if (!this.channel) return;

    // Create main events exchange
    await this.channel.assertExchange('events_exchange', 'topic', { durable: true });
    
    // Create dead letter exchange
    await this.channel.assertExchange('dead_letter_exchange', 'direct', { durable: true });
    await this.channel.assertQueue('dead_letter_queue', { durable: true });
    await this.channel.bindQueue('dead_letter_queue', 'dead_letter_exchange', 'dead_letter');
  }

  private async sendToDeadLetterQueue(topic: string, message: amqp.Message, error: Error): Promise<void> {
    if (!this.channel) return;

    await this.channel.publish(
      'dead_letter_exchange',
      'dead_letter',
      message.content,
      {
        headers: {
          originalTopic: topic,
          error: error.message,
          timestamp: new Date().toISOString()
        }
      }
    );
  }

  private async reconnect(): Promise<void> {
    setTimeout(async () => {
      try {
        await this.initialize();
        console.log('RabbitMQ reconnected successfully');
      } catch (error) {
        console.error('RabbitMQ reconnection failed:', error);
        this.reconnect();
      }
    }, 5000);
  }

  private generateCorrelationId(): string {
    return Math.random().toString(36).substring(2, 15) + Date.now().toString(36);
  }

  async disconnect(): Promise<void> {
    if (this.channel) {
      await this.channel.close();
    }
    if (this.connection) {
      await this.connection.close();
    }
  }
}

// Service Communication Manager
class ServiceCommunicationManager {
  private messageBus: MessageBus;
  private serviceRegistry: ServiceRegistry;
  private circuitBreakers: Map<string, CircuitBreaker> = new Map();

  constructor(messageBus: MessageBus, serviceRegistry: ServiceRegistry) {
    this.messageBus = messageBus;
    this.serviceRegistry = serviceRegistry;
  }

  // Saga Pattern Implementation
  async executeSaga(sagaDefinition: SagaDefinition, data: any): Promise<SagaResult> {
    const sagaId = this.generateSagaId();
    const sagaContext = {
      id: sagaId,
      steps: sagaDefinition.steps,
      currentStep: 0,
      data,
      completedSteps: [],
      compensationStack: []
    };

    try {
      for (const step of sagaDefinition.steps) {
        await this.executeStep(step, sagaContext);
        sagaContext.currentStep++;
        sagaContext.completedSteps.push(step);
      }

      return { success: true, sagaId, data: sagaContext.data };
    } catch (error) {
      // Execute compensation in reverse order
      await this.compensateSaga(sagaContext);
      return { success: false, sagaId, error: error.message };
    }
  }

  private async executeStep(step: SagaStep, context: SagaContext): Promise<void> {
    const circuitBreaker = this.getCircuitBreaker(step.service);
    
    try {
      const result = await circuitBreaker.fire(async () => {
        return this.messageBus.request(step.service, step.action, {
          ...step.data,
          ...context.data,
          sagaId: context.id
        });
      });

      // Update context with result
      context.data = { ...context.data, ...result };
      
      // Add compensation action to stack
      if (step.compensation) {
        context.compensationStack.push({
          service: step.service,
          action: step.compensation,
          data: result
        });
      }
    } catch (error) {
      throw new Error(`Step ${step.action} failed: ${error.message}`);
    }
  }

  private async compensateSaga(context: SagaContext): Promise<void> {
    for (const compensation of context.compensationStack.reverse()) {
      try {
        await this.messageBus.request(
          compensation.service,
          compensation.action,
          compensation.data
        );
      } catch (error) {
        console.error(`Compensation failed for ${compensation.action}:`, error);
      }
    }
  }

  private getCircuitBreaker(serviceName: string): CircuitBreaker {
    if (!this.circuitBreakers.has(serviceName)) {
      const breaker = new CircuitBreaker(async (fn: () => Promise<any>) => fn(), {
        timeout: 10000,
        errorThresholdPercentage: 50,
        resetTimeout: 30000
      });

      this.circuitBreakers.set(serviceName, breaker);
    }

    return this.circuitBreakers.get(serviceName)!;
  }

  private generateSagaId(): string {
    return `saga_${Date.now()}_${Math.random().toString(36).substring(2, 15)}`;
  }
}

// Type definitions
interface KafkaConfig {
  clientId: string;
  brokers: string[];
  groupId: string;
}

interface RabbitMQConfig {
  host: string;
  port: number;
  username: string;
  password: string;
  vhost?: string;
}

interface ServiceInstance {
  name: string;
  version: string;
  endpoint: string;
  healthy: boolean;
  lastHealthCheck?: Date;
  endpoints: string[];
  health: string;
}

interface SagaDefinition {
  name: string;
  steps: SagaStep[];
}

interface SagaStep {
  service: string;
  action: string;
  data?: any;
  compensation?: string;
}

interface SagaContext {
  id: string;
  steps: SagaStep[];
  currentStep: number;
  data: any;
  completedSteps: SagaStep[];
  compensationStack: Array<{
    service: string;
    action: string;
    data: any;
  }>;
}

interface SagaResult {
  success: boolean;
  sagaId: string;
  data?: any;
  error?: string;
}

interface RequestTransformation {
  type: 'ADD_HEADER' | 'REMOVE_HEADER' | 'MODIFY_PATH' | 'ADD_QUERY_PARAM';
  header?: string;
  value?: string;
  pattern?: string;
  replacement?: string;
  param?: string;
}

interface ResponseTransformation {
  type: 'REMOVE_FIELD' | 'RENAME_FIELD' | 'ADD_FIELD' | 'TRANSFORM_FIELD';
  field?: string;
  from?: string;
  to?: string;
  value?: any;
  transformer?: (value: any) => any;
}

interface RateLimitConfig {
  windowMs: number;
  max: number;
}

interface AuthConfig {
  jwtSecret: string;
  issuer: string;
  audience: string;
}

interface MonitoringConfig {
  jaegerEndpoint: string;
  prometheusEndpoint: string;
}

interface CircuitBreakerConfig {
  errorThreshold: number;
  resetTimeout: number;
}

export {
  EnterpriseAPIGateway,
  ServiceRegistry,
  KafkaMessageBus,
  RabbitMQMessageBus,
  ServiceCommunicationManager
};
```

## Interview-Ready Summary

**Enterprise API Design Principles:**

1. **API Gateway Pattern** - Centralized entry point, request routing, rate limiting, authentication
2. **Service Discovery** - Dynamic service registration, health checking, load balancing
3. **Circuit Breaker** - Fault tolerance, cascade failure prevention, graceful degradation
4. **Request/Response Transformation** - Data format adaptation, legacy system integration
5. **Distributed Tracing** - Request correlation, performance monitoring, debugging

**Microservices Communication Strategies:**
- **Synchronous** - HTTP REST APIs, GraphQL, gRPC for real-time operations
- **Asynchronous** - Event-driven messaging, pub/sub patterns, message queues
- **Hybrid** - Request-response with async callbacks, event sourcing with CQRS

**Advanced Patterns:**
- **Saga Pattern** - Distributed transaction management across services
- **Event Sourcing** - Immutable event log for state management
- **CQRS** - Command Query Responsibility Segregation for scalability
- **Outbox Pattern** - Reliable message publishing with database transactions

**Production Considerations:**
- **Security** - Service-to-service authentication, encryption, network policies
- **Observability** - Logging, metrics, tracing, alerting
- **Scalability** - Horizontal scaling, load balancing, caching strategies
- **Resilience** - Retry policies, timeouts, bulkhead pattern, chaos engineering

**Key Interview Topics:** API Gateway vs Service Mesh, event-driven architecture benefits, distributed transaction patterns, service discovery mechanisms, circuit breaker implementation, message queue comparisons, microservice decomposition strategies.