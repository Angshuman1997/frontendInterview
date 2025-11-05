# BFF Testing Strategies and Quality Assurance

Comprehensive testing strategies for Backend For Frontend (BFF) services require a multi-layered approach covering unit tests, integration tests, contract testing, and end-to-end scenarios. This guide provides enterprise-grade testing patterns for BFF architectures.

## BFF Testing Architecture

### 1. Testing Pyramid for BFF Services
```typescript
// BFF Testing Structure
import { describe, beforeEach, afterEach, it, expect, jest } from '@jest/globals';
import request from 'supertest';
import { BFFApplication } from '../src/bff-application';
import { MockServiceRegistry } from './mocks/service-registry.mock';
import { TestDatabase } from './utils/test-database';

// Test Configuration
interface BFFTestConfig {
  mockServices: boolean;
  useTestDatabase: boolean;
  enableMetrics: boolean;
  logLevel: 'debug' | 'info' | 'warn' | 'error';
}

class BFFTestSuite {
  private app: BFFApplication;
  private testDb: TestDatabase;
  private mockRegistry: MockServiceRegistry;

  constructor(private config: BFFTestConfig) {
    this.testDb = new TestDatabase();
    this.mockRegistry = new MockServiceRegistry();
  }

  async setup(): Promise<void> {
    // Initialize test database
    await this.testDb.initialize();
    
    // Setup service mocks
    if (this.config.mockServices) {
      await this.mockRegistry.startMocks();
    }

    // Initialize BFF application with test config
    this.app = new BFFApplication({
      environment: 'test',
      database: this.testDb.getConnectionString(),
      services: this.mockRegistry.getServiceUrls(),
      cache: { provider: 'memory' },
      auth: { secret: 'test-secret' }
    });

    await this.app.start();
  }

  async teardown(): Promise<void> {
    await this.app.stop();
    await this.mockRegistry.stopMocks();
    await this.testDb.cleanup();
  }

  getApp() {
    return this.app.getExpressApp();
  }
}

// Unit Tests for BFF Service Layer
describe('BFF Service Layer Tests', () => {
  let bffService: BFFService;
  let mockUserService: jest.Mocked<UserServiceClient>;
  let mockAnalyticsService: jest.Mocked<AnalyticsServiceClient>;

  beforeEach(() => {
    mockUserService = {
      getUserProfile: jest.fn(),
      updateUserPreferences: jest.fn(),
      getUserActivity: jest.fn(),
    } as any;

    mockAnalyticsService = {
      getAnalyticsData: jest.fn(),
      trackEvent: jest.fn(),
      getMetrics: jest.fn(),
    } as any;

    bffService = new BFFService({
      userService: mockUserService,
      analyticsService: mockAnalyticsService,
      cache: new MemoryCache(),
      logger: new Logger({ level: 'silent' })
    });
  });

  describe('Dashboard Data Aggregation', () => {
    it('should aggregate user dashboard data successfully', async () => {
      // Arrange
      const userId = 'user-123';
      const platform = 'web';
      
      const mockUser = {
        id: userId,
        name: 'John Doe',
        email: 'john@example.com',
        preferences: { theme: 'dark', language: 'en' }
      };

      const mockAnalytics = {
        pageViews: 1250,
        sessionTime: 45,
        conversionRate: 3.2
      };

      mockUserService.getUserProfile.mockResolvedValue(mockUser);
      mockUserService.getUserActivity.mockResolvedValue([]);
      mockAnalyticsService.getAnalyticsData.mockResolvedValue(mockAnalytics);

      // Act
      const result = await bffService.getDashboardData(userId, platform);

      // Assert
      expect(result).toEqual({
        user: mockUser,
        analytics: mockAnalytics,
        recentActivity: [],
        quickActions: expect.any(Array),
        notifications: expect.any(Array)
      });

      expect(mockUserService.getUserProfile).toHaveBeenCalledWith(userId);
      expect(mockAnalyticsService.getAnalyticsData).toHaveBeenCalledWith(userId, platform);
    });

    it('should handle service failures gracefully', async () => {
      // Arrange
      const userId = 'user-123';
      mockUserService.getUserProfile.mockRejectedValue(new Error('Service unavailable'));

      // Act & Assert
      await expect(bffService.getDashboardData(userId, 'web'))
        .rejects.toThrow('Failed to aggregate dashboard data');
    });

    it('should use cached data when available', async () => {
      // Arrange
      const userId = 'user-123';
      const cachedData = { cached: true };
      
      const cacheKey = `dashboard:${userId}:web`;
      await bffService.cache.set(cacheKey, cachedData, 300);

      // Act
      const result = await bffService.getDashboardData(userId, 'web');

      // Assert
      expect(result).toEqual(cachedData);
      expect(mockUserService.getUserProfile).not.toHaveBeenCalled();
    });
  });

  describe('Data Transformation', () => {
    it('should transform mobile dashboard format correctly', async () => {
      // Arrange
      const userId = 'user-123';
      const rawUserData = {
        id: userId,
        firstName: 'John',
        lastName: 'Doe',
        emailAddress: 'john@example.com',
        userSettings: { colorScheme: 'dark', locale: 'en-US' }
      };

      mockUserService.getUserProfile.mockResolvedValue(rawUserData);

      // Act
      const result = await bffService.getDashboardData(userId, 'mobile');

      // Assert
      expect(result.user).toEqual({
        id: userId,
        name: 'John Doe',
        email: 'john@example.com',
        preferences: {
          theme: 'dark',
          language: 'en'
        }
      });
    });
  });
});

// Integration Tests
describe('BFF API Integration Tests', () => {
  let testSuite: BFFTestSuite;
  let app: any;

  beforeAll(async () => {
    testSuite = new BFFTestSuite({
      mockServices: true,
      useTestDatabase: true,
      enableMetrics: false,
      logLevel: 'warn'
    });
    
    await testSuite.setup();
    app = testSuite.getApp();
  });

  afterAll(async () => {
    await testSuite.teardown();
  });

  describe('Dashboard API', () => {
    it('should return dashboard data for authenticated user', async () => {
      // Arrange
      const authToken = await generateTestToken({ userId: 'user-123' });

      // Act
      const response = await request(app)
        .get('/api/dashboard')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      // Assert
      expect(response.body).toHaveProperty('user');
      expect(response.body).toHaveProperty('analytics');
      expect(response.body).toHaveProperty('recentActivity');
      expect(response.body.user.id).toBe('user-123');
    });

    it('should return 401 for unauthenticated requests', async () => {
      await request(app)
        .get('/api/dashboard')
        .expect(401);
    });

    it('should handle platform-specific responses', async () => {
      const authToken = await generateTestToken({ userId: 'user-123' });

      const mobileResponse = await request(app)
        .get('/api/dashboard?platform=mobile')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      const webResponse = await request(app)
        .get('/api/dashboard?platform=web')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      // Mobile response should have simplified structure
      expect(Object.keys(mobileResponse.body)).toHaveLength(3);
      expect(Object.keys(webResponse.body)).toBeGreaterThan(3);
    });
  });

  describe('Search API', () => {
    it('should perform aggregated search across services', async () => {
      const authToken = await generateTestToken({ userId: 'user-123' });

      const response = await request(app)
        .get('/api/search?q=test&type=all')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(response.body).toHaveProperty('results');
      expect(response.body).toHaveProperty('facets');
      expect(response.body).toHaveProperty('pagination');
    });

    it('should validate search parameters', async () => {
      const authToken = await generateTestToken({ userId: 'user-123' });

      await request(app)
        .get('/api/search?q=') // Empty query
        .set('Authorization', `Bearer ${authToken}`)
        .expect(400);
    });
  });
});

// Contract Testing with Pact
describe('BFF Contract Tests', () => {
  let provider: any;

  beforeEach(() => {
    provider = new Pact({
      consumer: 'bff-service',
      provider: 'user-service',
      port: 1234,
      log: path.resolve(process.cwd(), 'logs', 'pact.log'),
      dir: path.resolve(process.cwd(), 'pacts'),
      logLevel: 'INFO',
    });

    return provider.setup();
  });

  afterEach(() => provider.finalize());

  describe('User Service Contract', () => {
    beforeEach(() => {
      const userProfileResponse = {
        id: like('user-123'),
        name: like('John Doe'),
        email: like('john@example.com'),
        preferences: {
          theme: regex('light|dark', 'dark'),
          language: regex('[a-z]{2}', 'en')
        }
      };

      return provider.addInteraction({
        state: 'user exists',
        uponReceiving: 'a request for user profile',
        withRequest: {
          method: 'GET',
          path: '/api/users/user-123',
          headers: {
            'Accept': 'application/json',
            'Authorization': regex('Bearer .+', 'Bearer token')
          }
        },
        willRespondWith: {
          status: 200,
          headers: {
            'Content-Type': 'application/json'
          },
          body: userProfileResponse
        }
      });
    });

    it('should receive valid user profile from user service', async () => {
      const userService = new UserServiceClient('http://localhost:1234');
      const userProfile = await userService.getUserProfile('user-123', 'Bearer token');

      expect(userProfile).toMatchObject({
        id: 'user-123',
        name: expect.any(String),
        email: expect.any(String),
        preferences: {
          theme: expect.stringMatching(/^(light|dark)$/),
          language: expect.stringMatching(/^[a-z]{2}$/)
        }
      });
    });
  });
});

// Performance Testing
describe('BFF Performance Tests', () => {
  let testSuite: BFFTestSuite;
  let app: any;

  beforeAll(async () => {
    testSuite = new BFFTestSuite({
      mockServices: true,
      useTestDatabase: true,
      enableMetrics: true,
      logLevel: 'error'
    });
    
    await testSuite.setup();
    app = testSuite.getApp();
  });

  afterAll(async () => {
    await testSuite.teardown();
  });

  it('should handle concurrent dashboard requests', async () => {
    const authToken = await generateTestToken({ userId: 'user-123' });
    const concurrentRequests = 100;
    
    const startTime = Date.now();
    
    const promises = Array(concurrentRequests).fill(null).map(() =>
      request(app)
        .get('/api/dashboard')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)
    );

    const responses = await Promise.all(promises);
    const endTime = Date.now();
    
    const averageResponseTime = (endTime - startTime) / concurrentRequests;
    
    expect(responses).toHaveLength(concurrentRequests);
    expect(averageResponseTime).toBeLessThan(100); // Average response time should be under 100ms
  });

  it('should maintain response time under load', async () => {
    const authToken = await generateTestToken({ userId: 'user-123' });
    
    // Warmup
    await request(app)
      .get('/api/dashboard')
      .set('Authorization', `Bearer ${authToken}`);

    // Measure response time
    const startTime = process.hrtime.bigint();
    
    await request(app)
      .get('/api/dashboard')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);
    
    const endTime = process.hrtime.bigint();
    const responseTimeMs = Number(endTime - startTime) / 1_000_000;
    
    expect(responseTimeMs).toBeLessThan(50); // Should respond in under 50ms
  });
});

// Error Handling Tests
describe('BFF Error Handling Tests', () => {
  let testSuite: BFFTestSuite;
  let app: any;

  beforeAll(async () => {
    testSuite = new BFFTestSuite({
      mockServices: false, // Use real services to test error scenarios
      useTestDatabase: true,
      enableMetrics: false,
      logLevel: 'error'
    });
    
    await testSuite.setup();
    app = testSuite.getApp();
  });

  afterAll(async () => {
    await testSuite.teardown();
  });

  it('should handle service timeouts gracefully', async () => {
    // Mock service delay
    const mockServiceWithDelay = nock('http://user-service:3001')
      .get('/api/users/user-123')
      .delay(6000) // 6 second delay (longer than 5s timeout)
      .reply(200, { id: 'user-123' });

    const authToken = await generateTestToken({ userId: 'user-123' });

    const response = await request(app)
      .get('/api/dashboard')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(503); // Service Unavailable

    expect(response.body.error).toContain('timeout');
  });

  it('should handle partial service failures', async () => {
    // Mock user service success but analytics service failure
    nock('http://user-service:3001')
      .get('/api/users/user-123')
      .reply(200, { id: 'user-123', name: 'John' });

    nock('http://analytics-service:3002')
      .get('/api/analytics/user-123')
      .reply(500, { error: 'Internal server error' });

    const authToken = await generateTestToken({ userId: 'user-123' });

    const response = await request(app)
      .get('/api/dashboard')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200); // Should still return partial data

    expect(response.body.user).toBeDefined();
    expect(response.body.analytics).toBeNull();
    expect(response.body.warnings).toContain('analytics service unavailable');
  });
});

// Load Testing Utilities
class LoadTestRunner {
  private app: any;
  private metrics: LoadTestMetrics;

  constructor(app: any) {
    this.app = app;
    this.metrics = new LoadTestMetrics();
  }

  async runLoadTest(config: LoadTestConfig): Promise<LoadTestResults> {
    const { concurrency, duration, endpoint, authToken } = config;
    
    const workers = Array(concurrency).fill(null).map((_, index) => 
      this.createWorker(index, endpoint, authToken)
    );

    const startTime = Date.now();
    await Promise.all(workers.map(worker => 
      this.runWorker(worker, duration)
    ));
    const endTime = Date.now();

    return this.metrics.generateReport(endTime - startTime);
  }

  private createWorker(id: number, endpoint: string, authToken: string) {
    return {
      id,
      endpoint,
      authToken,
      requestCount: 0,
      errorCount: 0,
      responseTimes: [] as number[]
    };
  }

  private async runWorker(worker: any, duration: number): Promise<void> {
    const endTime = Date.now() + duration;
    
    while (Date.now() < endTime) {
      try {
        const startTime = process.hrtime.bigint();
        
        await request(this.app)
          .get(worker.endpoint)
          .set('Authorization', `Bearer ${worker.authToken}`)
          .expect(200);
        
        const endRequestTime = process.hrtime.bigint();
        const responseTime = Number(endRequestTime - startTime) / 1_000_000;
        
        worker.requestCount++;
        worker.responseTimes.push(responseTime);
        this.metrics.recordRequest(responseTime);
      } catch (error) {
        worker.errorCount++;
        this.metrics.recordError();
      }
    }
  }
}

class LoadTestMetrics {
  private requests: number[] = [];
  private errors: number = 0;

  recordRequest(responseTime: number): void {
    this.requests.push(responseTime);
  }

  recordError(): void {
    this.errors++;
  }

  generateReport(totalDuration: number): LoadTestResults {
    const totalRequests = this.requests.length;
    const requestsPerSecond = totalRequests / (totalDuration / 1000);
    const averageResponseTime = this.requests.reduce((a, b) => a + b, 0) / totalRequests;
    
    this.requests.sort((a, b) => a - b);
    const p95 = this.requests[Math.floor(totalRequests * 0.95)];
    const p99 = this.requests[Math.floor(totalRequests * 0.99)];

    return {
      totalRequests,
      totalErrors: this.errors,
      errorRate: this.errors / totalRequests,
      requestsPerSecond,
      averageResponseTime,
      p95ResponseTime: p95,
      p99ResponseTime: p99,
      duration: totalDuration
    };
  }
}

// Test Utilities
async function generateTestToken(payload: any): Promise<string> {
  return jwt.sign(payload, 'test-secret', { expiresIn: '1h' });
}

// Helper to create test data
class TestDataFactory {
  static createUser(overrides: Partial<User> = {}): User {
    return {
      id: 'user-123',
      name: 'Test User',
      email: 'test@example.com',
      preferences: {
        theme: 'light',
        language: 'en',
        notifications: {
          email: true,
          push: false,
          sms: false
        }
      },
      ...overrides
    };
  }

  static createAnalyticsData(overrides: Partial<AnalyticsData> = {}): AnalyticsData {
    return {
      pageViews: 100,
      sessionTime: 25,
      conversionRate: 2.5,
      topPages: ['/dashboard', '/profile'],
      deviceBreakdown: { mobile: 60, desktop: 40 },
      ...overrides
    };
  }
}

// Type Definitions
interface LoadTestConfig {
  concurrency: number;
  duration: number;
  endpoint: string;
  authToken: string;
}

interface LoadTestResults {
  totalRequests: number;
  totalErrors: number;
  errorRate: number;
  requestsPerSecond: number;
  averageResponseTime: number;
  p95ResponseTime: number;
  p99ResponseTime: number;
  duration: number;
}

export {
  BFFTestSuite,
  LoadTestRunner,
  TestDataFactory
};
```

## Interview-Ready Testing Summary

**BFF Testing Strategy covers:**

1. **Unit Testing** - Service layer logic, data transformation, caching behavior
2. **Integration Testing** - API endpoints, authentication, platform-specific responses
3. **Contract Testing** - Service interface validation with Pact
4. **Performance Testing** - Load handling, response times, concurrent requests
5. **Error Handling** - Service failures, timeouts, partial failures

**Key Testing Patterns:**
- **Test Doubles** - Mocked external services for isolation
- **Test Data Factory** - Consistent test data generation
- **Load Testing** - Performance validation under stress
- **Contract Validation** - API compatibility across services

**Production Considerations:**
- **Test Environment Isolation** - Separate test databases and services
- **Metrics Collection** - Performance monitoring during tests
- **Error Scenarios** - Comprehensive failure mode testing
- **Regression Prevention** - Automated test suites in CI/CD

This comprehensive testing approach ensures BFF reliability, performance, and maintainability at enterprise scale.