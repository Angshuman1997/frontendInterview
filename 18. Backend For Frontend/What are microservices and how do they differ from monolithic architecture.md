# What are microservices and how do they differ from monolithic architecture

Microservices architecture is a design approach where applications are built as a collection of loosely coupled, independently deployable services. Each service is focused on a specific business capability and communicates with other services through well-defined APIs. This differs significantly from monolithic architecture where all components are interconnected and deployed as a single unit.

## Core Concepts

### Monolithic Architecture

**Characteristics:**
- Single deployable unit
- Shared database
- Internal function calls
- Centralized business logic
- Single technology stack

```typescript
// Monolithic architecture example
// Everything in one application

// User service, order service, payment service all in one codebase
class ECommerceApp {
  private userService: UserService;
  private orderService: OrderService;
  private paymentService: PaymentService;
  private inventoryService: InventoryService;

  constructor() {
    // All services initialized together
    this.userService = new UserService();
    this.orderService = new OrderService();
    this.paymentService = new PaymentService();
    this.inventoryService = new InventoryService();
  }

  // All business logic in one application
  async processOrder(orderData: OrderData) {
    // Direct function calls
    const user = await this.userService.validateUser(orderData.userId);
    const inventory = await this.inventoryService.checkStock(orderData.items);
    const payment = await this.paymentService.processPayment(orderData.payment);
    const order = await this.orderService.createOrder(orderData);
    
    return order;
  }
}

// Single database, single deployment
const app = new ECommerceApp();
app.listen(3000);
```

### Microservices Architecture

**Characteristics:**
- Multiple independent services
- Service-specific databases
- Network-based communication
- Distributed business logic
- Technology diversity allowed

```typescript
// Microservices architecture example
// Each service is separate and independent

// User Service (separate application)
class UserService {
  private database: UserDatabase;

  async validateUser(userId: string): Promise<User> {
    return this.database.findById(userId);
  }

  async createUser(userData: UserData): Promise<User> {
    return this.database.create(userData);
  }
}

// Order Service (separate application)
class OrderService {
  private database: OrderDatabase;
  private userServiceClient: UserServiceClient;
  private inventoryServiceClient: InventoryServiceClient;
  private paymentServiceClient: PaymentServiceClient;

  async createOrder(orderData: OrderData): Promise<Order> {
    // Network calls to other services
    const user = await this.userServiceClient.validateUser(orderData.userId);
    const stockCheck = await this.inventoryServiceClient.checkStock(orderData.items);
    
    if (!stockCheck.available) {
      throw new Error('Insufficient stock');
    }

    const payment = await this.paymentServiceClient.processPayment(orderData.payment);
    
    const order = await this.database.create({
      ...orderData,
      status: 'confirmed',
      paymentId: payment.id
    });

    // Publish event for other services
    await this.eventBus.publish('order.created', order);
    
    return order;
  }
}

// Payment Service (separate application)
class PaymentService {
  private database: PaymentDatabase;
  private stripeClient: StripeClient;

  async processPayment(paymentData: PaymentData): Promise<Payment> {
    const stripeCharge = await this.stripeClient.createCharge(paymentData);
    
    return this.database.create({
      ...paymentData,
      stripeChargeId: stripeCharge.id,
      status: 'completed'
    });
  }
}

// Each service runs independently
// User Service: localhost:3001
// Order Service: localhost:3002
// Payment Service: localhost:3003
// Inventory Service: localhost:3004
```

## Architecture Comparison

### 1. Deployment Patterns

**Monolithic Deployment:**
```yaml
# docker-compose.yml for monolith
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://localhost:5432/ecommerce
    depends_on:
      - database
  
  database:
    image: postgres:13
    environment:
      - POSTGRES_DB=ecommerce
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

**Microservices Deployment:**
```yaml
# docker-compose.yml for microservices
version: '3.8'
services:
  user-service:
    build: ./services/user-service
    ports:
      - "3001:3000"
    environment:
      - DATABASE_URL=postgresql://user-db:5432/users
    depends_on:
      - user-db
  
  order-service:
    build: ./services/order-service
    ports:
      - "3002:3000"
    environment:
      - DATABASE_URL=postgresql://order-db:5432/orders
      - USER_SERVICE_URL=http://user-service:3000
      - PAYMENT_SERVICE_URL=http://payment-service:3000
    depends_on:
      - order-db
  
  payment-service:
    build: ./services/payment-service
    ports:
      - "3003:3000"
    environment:
      - DATABASE_URL=postgresql://payment-db:5432/payments
    depends_on:
      - payment-db
  
  # Separate databases for each service
  user-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=users
  
  order-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=orders
      
  payment-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=payments
  
  # API Gateway
  api-gateway:
    build: ./gateway
    ports:
      - "3000:3000"
    environment:
      - USER_SERVICE_URL=http://user-service:3000
      - ORDER_SERVICE_URL=http://order-service:3000
      - PAYMENT_SERVICE_URL=http://payment-service:3000
```

### 2. Communication Patterns

**Synchronous Communication (REST/GraphQL):**
```typescript
// API Gateway routing requests to microservices
class APIGateway {
  private userServiceClient: UserServiceClient;
  private orderServiceClient: OrderServiceClient;
  private paymentServiceClient: PaymentServiceClient;

  constructor() {
    this.userServiceClient = new UserServiceClient('http://user-service:3000');
    this.orderServiceClient = new OrderServiceClient('http://order-service:3000');
    this.paymentServiceClient = new PaymentServiceClient('http://payment-service:3000');
  }

  setupRoutes() {
    // Route user requests to user service
    this.app.use('/api/users', this.createProxy(this.userServiceClient));
    
    // Route order requests to order service
    this.app.use('/api/orders', this.createProxy(this.orderServiceClient));
    
    // Route payment requests to payment service
    this.app.use('/api/payments', this.createProxy(this.paymentServiceClient));
  }

  private createProxy(serviceClient: ServiceClient) {
    return async (req: Request, res: Response) => {
      try {
        const response = await serviceClient.request({
          method: req.method,
          path: req.path,
          body: req.body,
          headers: req.headers
        });
        
        res.status(response.status).json(response.data);
      } catch (error) {
        res.status(500).json({ error: 'Service unavailable' });
      }
    };
  }
}

// Service client with circuit breaker pattern
class ServiceClient {
  private circuitBreaker: CircuitBreaker;

  constructor(private baseUrl: string) {
    this.circuitBreaker = new CircuitBreaker(this.makeRequest.bind(this), {
      timeout: 5000,
      errorThresholdPercentage: 50,
      resetTimeout: 30000
    });
  }

  async request(options: RequestOptions): Promise<ServiceResponse> {
    return this.circuitBreaker.fire(options);
  }

  private async makeRequest(options: RequestOptions): Promise<ServiceResponse> {
    const response = await fetch(`${this.baseUrl}${options.path}`, {
      method: options.method,
      headers: options.headers,
      body: JSON.stringify(options.body)
    });

    if (!response.ok) {
      throw new Error(`Service request failed: ${response.status}`);
    }

    return {
      status: response.status,
      data: await response.json()
    };
  }
}
```

**Asynchronous Communication (Message Queues):**
```typescript
// Event-driven communication between microservices
import { EventEmitter } from 'events';

interface DomainEvent {
  id: string;
  type: string;
  aggregateId: string;
  data: any;
  timestamp: Date;
  version: number;
}

class EventBus {
  private publisher: MessagePublisher;
  private subscriber: MessageSubscriber;

  constructor() {
    this.publisher = new MessagePublisher('amqp://rabbitmq:5672');
    this.subscriber = new MessageSubscriber('amqp://rabbitmq:5672');
  }

  async publish(eventType: string, data: any): Promise<void> {
    const event: DomainEvent = {
      id: generateId(),
      type: eventType,
      aggregateId: data.id,
      data,
      timestamp: new Date(),
      version: 1
    };

    await this.publisher.publish(eventType, event);
  }

  async subscribe(eventType: string, handler: (event: DomainEvent) => Promise<void>): Promise<void> {
    await this.subscriber.subscribe(eventType, async (message) => {
      try {
        await handler(message);
        // Acknowledge message processing
        message.ack();
      } catch (error) {
        console.error(`Error processing event ${eventType}:`, error);
        // Reject and requeue message
        message.nack(false, true);
      }
    });
  }
}

// Order service publishing events
class OrderService {
  private eventBus: EventBus;

  async createOrder(orderData: OrderData): Promise<Order> {
    const order = await this.database.create(orderData);
    
    // Publish event for other services to react
    await this.eventBus.publish('order.created', {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      totalAmount: order.totalAmount
    });
    
    return order;
  }
}

// Inventory service listening to order events
class InventoryService {
  private eventBus: EventBus;

  constructor() {
    this.eventBus = new EventBus();
    this.setupEventListeners();
  }

  private async setupEventListeners() {
    await this.eventBus.subscribe('order.created', this.handleOrderCreated.bind(this));
    await this.eventBus.subscribe('order.cancelled', this.handleOrderCancelled.bind(this));
  }

  private async handleOrderCreated(event: DomainEvent) {
    const { orderId, items } = event.data;
    
    // Reduce inventory for ordered items
    for (const item of items) {
      await this.reduceStock(item.productId, item.quantity);
    }

    // Publish inventory updated event
    await this.eventBus.publish('inventory.updated', {
      orderId,
      items: items.map(item => ({
        productId: item.productId,
        newStock: this.getStock(item.productId)
      }))
    });
  }

  private async handleOrderCancelled(event: DomainEvent) {
    const { items } = event.data;
    
    // Restore inventory for cancelled items
    for (const item of items) {
      await this.restoreStock(item.productId, item.quantity);
    }
  }
}
```

## Service Discovery and Configuration

### 1. Service Discovery

```typescript
// Service registry for dynamic service discovery
class ServiceRegistry {
  private services = new Map<string, ServiceInstance[]>();
  private healthChecks = new Map<string, NodeJS.Timeout>();

  registerService(serviceName: string, instance: ServiceInstance): void {
    if (!this.services.has(serviceName)) {
      this.services.set(serviceName, []);
    }
    
    this.services.get(serviceName)!.push(instance);
    this.startHealthCheck(serviceName, instance);
    
    console.log(`Service registered: ${serviceName} at ${instance.url}`);
  }

  unregisterService(serviceName: string, instanceId: string): void {
    const instances = this.services.get(serviceName);
    if (instances) {
      const index = instances.findIndex(i => i.id === instanceId);
      if (index !== -1) {
        instances.splice(index, 1);
        this.stopHealthCheck(instanceId);
      }
    }
  }

  getService(serviceName: string): ServiceInstance | null {
    const instances = this.services.get(serviceName);
    if (!instances || instances.length === 0) {
      return null;
    }
    
    // Simple round-robin load balancing
    const instance = instances[Math.floor(Math.random() * instances.length)];
    return instance;
  }

  getAllServices(): Map<string, ServiceInstance[]> {
    return this.services;
  }

  private startHealthCheck(serviceName: string, instance: ServiceInstance): void {
    const interval = setInterval(async () => {
      try {
        const response = await fetch(`${instance.url}/health`);
        if (!response.ok) {
          throw new Error(`Health check failed: ${response.status}`);
        }
        
        instance.lastHealthCheck = new Date();
        instance.status = 'healthy';
      } catch (error) {
        console.error(`Health check failed for ${serviceName}:`, error);
        instance.status = 'unhealthy';
        
        // Remove unhealthy instances after 3 failed checks
        if (instance.failedHealthChecks >= 2) {
          this.unregisterService(serviceName, instance.id);
        } else {
          instance.failedHealthChecks++;
        }
      }
    }, 30000); // Check every 30 seconds

    this.healthChecks.set(instance.id, interval);
  }

  private stopHealthCheck(instanceId: string): void {
    const interval = this.healthChecks.get(instanceId);
    if (interval) {
      clearInterval(interval);
      this.healthChecks.delete(instanceId);
    }
  }
}

interface ServiceInstance {
  id: string;
  url: string;
  version: string;
  status: 'healthy' | 'unhealthy';
  lastHealthCheck: Date;
  failedHealthChecks: number;
  metadata: Record<string, any>;
}

// Service auto-registration
class ServiceRegistration {
  private registry: ServiceRegistry;
  private instance: ServiceInstance;

  constructor(serviceName: string, serviceUrl: string, registry: ServiceRegistry) {
    this.registry = registry;
    this.instance = {
      id: generateId(),
      url: serviceUrl,
      version: process.env.SERVICE_VERSION || '1.0.0',
      status: 'healthy',
      lastHealthCheck: new Date(),
      failedHealthChecks: 0,
      metadata: {
        startTime: new Date(),
        nodeVersion: process.version
      }
    };

    // Register service on startup
    this.registry.registerService(serviceName, this.instance);

    // Graceful shutdown
    process.on('SIGTERM', () => {
      this.registry.unregisterService(serviceName, this.instance.id);
      process.exit(0);
    });
  }
}

// Usage in microservice
const registry = new ServiceRegistry();
const registration = new ServiceRegistration('user-service', 'http://localhost:3001', registry);
```

### 2. Configuration Management

```typescript
// Centralized configuration management
class ConfigurationManager {
  private config: Map<string, any> = new Map();
  private watchers: Map<string, ((value: any) => void)[]> = new Map();

  async loadConfiguration(serviceName: string): Promise<void> {
    try {
      // Load from multiple sources
      const envConfig = this.loadFromEnvironment();
      const fileConfig = await this.loadFromFile(`./config/${serviceName}.json`);
      const remoteConfig = await this.loadFromRemoteService(serviceName);

      // Merge configurations (precedence: remote > file > env)
      const mergedConfig = {
        ...envConfig,
        ...fileConfig,
        ...remoteConfig
      };

      this.config.set(serviceName, mergedConfig);
      this.notifyWatchers(serviceName, mergedConfig);
    } catch (error) {
      console.error(`Failed to load configuration for ${serviceName}:`, error);
    }
  }

  get(serviceName: string, key: string, defaultValue?: any): any {
    const serviceConfig = this.config.get(serviceName);
    return serviceConfig?.[key] ?? defaultValue;
  }

  watch(serviceName: string, callback: (config: any) => void): void {
    if (!this.watchers.has(serviceName)) {
      this.watchers.set(serviceName, []);
    }
    this.watchers.get(serviceName)!.push(callback);
  }

  private loadFromEnvironment(): any {
    return {
      port: process.env.PORT || 3000,
      database: {
        url: process.env.DATABASE_URL,
        pool: {
          min: parseInt(process.env.DB_POOL_MIN || '2'),
          max: parseInt(process.env.DB_POOL_MAX || '10')
        }
      },
      redis: {
        url: process.env.REDIS_URL
      },
      jwt: {
        secret: process.env.JWT_SECRET,
        expiresIn: process.env.JWT_EXPIRES_IN || '24h'
      }
    };
  }

  private async loadFromFile(filePath: string): Promise<any> {
    try {
      const content = await fs.readFile(filePath, 'utf-8');
      return JSON.parse(content);
    } catch (error) {
      return {};
    }
  }

  private async loadFromRemoteService(serviceName: string): Promise<any> {
    try {
      const response = await fetch(`http://config-service:3000/config/${serviceName}`);
      if (response.ok) {
        return await response.json();
      }
    } catch (error) {
      console.warn('Failed to load remote configuration:', error);
    }
    return {};
  }

  private notifyWatchers(serviceName: string, config: any): void {
    const watchers = this.watchers.get(serviceName);
    if (watchers) {
      watchers.forEach(callback => callback(config));
    }
  }
}

// Configuration-aware service base class
abstract class MicroService {
  protected config: ConfigurationManager;
  protected serviceName: string;

  constructor(serviceName: string) {
    this.serviceName = serviceName;
    this.config = new ConfigurationManager();
    this.initialize();
  }

  private async initialize(): Promise<void> {
    await this.config.loadConfiguration(this.serviceName);
    
    // Watch for configuration changes
    this.config.watch(this.serviceName, (newConfig) => {
      this.onConfigurationChanged(newConfig);
    });

    await this.onInitialized();
  }

  protected abstract onInitialized(): Promise<void>;
  protected abstract onConfigurationChanged(config: any): void;

  protected getConfig(key: string, defaultValue?: any): any {
    return this.config.get(this.serviceName, key, defaultValue);
  }
}

// Example service implementation
class UserService extends MicroService {
  private database: Database;
  private server: Express;

  constructor() {
    super('user-service');
  }

  protected async onInitialized(): Promise<void> {
    const dbConfig = this.getConfig('database');
    this.database = new Database(dbConfig);
    
    const port = this.getConfig('port', 3000);
    this.server = express();
    this.setupRoutes();
    this.server.listen(port);
  }

  protected onConfigurationChanged(config: any): void {
    // Handle configuration changes
    if (config.database) {
      this.database.updateConfiguration(config.database);
    }
  }

  private setupRoutes(): void {
    this.server.get('/users/:id', async (req, res) => {
      const user = await this.database.findById(req.params.id);
      res.json(user);
    });
  }
}
```

## Advantages and Disadvantages

### Monolithic Advantages
- **Simplicity** - Single codebase, easier to develop and test
- **Performance** - No network overhead for internal calls
- **ACID Transactions** - Easy to maintain data consistency
- **Debugging** - Single process, easier to debug
- **Deployment** - Single deployment unit

### Monolithic Disadvantages
- **Scalability** - Must scale entire application
- **Technology Lock-in** - Single technology stack
- **Team Dependencies** - Teams must coordinate deployments
- **Failure Impact** - Single point of failure
- **Development Speed** - Slower for large teams

### Microservices Advantages
- **Independent Scaling** - Scale services based on demand
- **Technology Diversity** - Choose best tool for each service
- **Team Autonomy** - Independent development and deployment
- **Fault Isolation** - Service failures don't affect entire system
- **Faster Development** - Parallel development across teams

### Microservices Disadvantages
- **Complexity** - Distributed system challenges
- **Network Overhead** - Inter-service communication latency
- **Data Consistency** - Eventual consistency challenges
- **Testing Complexity** - Integration testing across services
- **Operational Overhead** - More services to monitor and maintain

## When to Choose Each Architecture

### Choose Monolithic When:
- Small team (< 10 developers)
- Simple application requirements
- Strong consistency requirements
- Limited operational expertise
- Rapid prototyping or MVP development
- Well-defined, stable requirements

### Choose Microservices When:
- Large organization with multiple teams
- Need to scale different parts independently
- Want technology diversity
- Have strong DevOps/operational capabilities
- Complex domain with clear service boundaries
- Need to support different release cycles

## Migration Strategies

### Strangler Fig Pattern
```typescript
// Gradual migration from monolith to microservices
class APIGateway {
  private monolithUrl: string;
  private microservices: Map<string, string>;

  constructor() {
    this.monolithUrl = 'http://monolith:3000';
    this.microservices = new Map([
      ['users', 'http://user-service:3000'],
      ['orders', 'http://order-service:3000']
    ]);
  }

  async routeRequest(req: Request, res: Response): Promise<void> {
    const path = req.path;
    
    // Route to microservice if available
    if (path.startsWith('/api/users') && this.microservices.has('users')) {
      await this.proxyToService('users', req, res);
    } else if (path.startsWith('/api/orders') && this.microservices.has('orders')) {
      await this.proxyToService('orders', req, res);
    } else {
      // Fallback to monolith
      await this.proxyToMonolith(req, res);
    }
  }

  private async proxyToService(serviceName: string, req: Request, res: Response): Promise<void> {
    const serviceUrl = this.microservices.get(serviceName);
    // Proxy request to microservice
  }

  private async proxyToMonolith(req: Request, res: Response): Promise<void> {
    // Proxy request to existing monolith
  }
}
```

## Interview-Ready Summary

**Microservices vs Monolithic Architecture:**

**Monolith:** Single deployable unit with shared database, internal function calls, centralized logic. Best for small teams, simple requirements, strong consistency needs.

**Microservices:** Independent services with separate databases, network communication, distributed logic. Best for large teams, complex domains, independent scaling needs.

**Key trade-offs:**
- **Complexity vs Scalability** - Microservices add operational complexity but enable independent scaling
- **Consistency vs Availability** - Monoliths offer strong consistency, microservices eventual consistency
- **Development Speed vs Team Independence** - Monoliths faster for small teams, microservices enable large team autonomy

**Implementation concerns:** Service discovery, configuration management, data consistency, distributed monitoring, network reliability, testing strategies.

**Migration approach:** Start with monolith, extract services gradually using strangler fig pattern when team/complexity reaches appropriate scale.