# Monitoring, Observability, and Performance Analytics

Modern frontend applications require comprehensive monitoring and observability strategies to ensure optimal performance, user experience, and system reliability. This guide covers enterprise-grade monitoring solutions including APM, RUM, synthetic monitoring, and advanced analytics patterns for production environments.

## Application Performance Monitoring (APM) Architecture

### Comprehensive Monitoring Stack Implementation

```typescript
// monitoring/performance-tracker.ts
import { LCP, FID, CLS, TTFB, FCP, INP } from 'web-vitals';

interface PerformanceMetric {
  name: string;
  value: number;
  delta?: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  timestamp: number;
  sessionId: string;
  userId?: string;
  url: string;
  userAgent: string;
  connection?: NetworkInformation;
  deviceMemory?: number;
  hardwareConcurrency?: number;
}

interface CustomMetric {
  name: string;
  value: number;
  unit: 'ms' | 'bytes' | 'count' | 'percentage';
  tags: Record<string, string>;
  timestamp: number;
}

interface ErrorMetric {
  message: string;
  stack?: string;
  source: string;
  lineno: number;
  colno: number;
  timestamp: number;
  userId?: string;
  sessionId: string;
  url: string;
  userAgent: string;
  componentStack?: string;
  errorBoundary?: string;
  severity: 'low' | 'medium' | 'high' | 'critical';
}

class AdvancedPerformanceTracker {
  private sessionId: string;
  private userId?: string;
  private metrics: PerformanceMetric[] = [];
  private customMetrics: CustomMetric[] = [];
  private errors: ErrorMetric[] = [];
  private batchSize = 10;
  private flushInterval = 30000; // 30 seconds
  private endpoint: string;
  private apiKey: string;
  private environment: string;
  private version: string;

  constructor(config: {
    endpoint: string;
    apiKey: string;
    environment: string;
    version: string;
    userId?: string;
    batchSize?: number;
    flushInterval?: number;
  }) {
    this.sessionId = this.generateSessionId();
    this.userId = config.userId;
    this.endpoint = config.endpoint;
    this.apiKey = config.apiKey;
    this.environment = config.environment;
    this.version = config.version;
    this.batchSize = config.batchSize || 10;
    this.flushInterval = config.flushInterval || 30000;

    this.initializeTracking();
    this.setupPeriodicFlush();
    this.setupUnloadHandler();
  }

  private initializeTracking(): void {
    // Core Web Vitals tracking
    LCP(this.handleMetric.bind(this));
    FID(this.handleMetric.bind(this));
    CLS(this.handleMetric.bind(this));
    TTFB(this.handleMetric.bind(this));
    FCP(this.handleMetric.bind(this));
    INP(this.handleMetric.bind(this));

    // Custom performance observers
    this.observeResourceTiming();
    this.observeNavigationTiming();
    this.observeLongTasks();
    this.observeLayoutShifts();
    this.observePaintTiming();
    this.observeMemoryUsage();

    // Error tracking
    this.setupErrorTracking();
    this.setupUnhandledRejectionTracking();

    // User interaction tracking
    this.setupInteractionTracking();
  }

  private handleMetric(metric: any): void {
    const performanceMetric: PerformanceMetric = {
      name: metric.name,
      value: metric.value,
      delta: metric.delta,
      rating: metric.rating,
      timestamp: Date.now(),
      sessionId: this.sessionId,
      userId: this.userId,
      url: window.location.href,
      userAgent: navigator.userAgent,
      connection: (navigator as any).connection,
      deviceMemory: (navigator as any).deviceMemory,
      hardwareConcurrency: navigator.hardwareConcurrency
    };

    this.metrics.push(performanceMetric);
    this.checkFlushCondition();
  }

  private observeResourceTiming(): void {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'resource') {
          const resourceEntry = entry as PerformanceResourceTiming;
          
          // Track large resources
          if (resourceEntry.transferSize > 100000) { // 100KB
            this.trackCustomMetric({
              name: 'large_resource_load',
              value: resourceEntry.duration,
              unit: 'ms',
              tags: {
                resource_type: this.getResourceType(resourceEntry.name),
                resource_size: resourceEntry.transferSize.toString(),
                cache_status: resourceEntry.transferSize === 0 ? 'cached' : 'network'
              },
              timestamp: Date.now()
            });
          }

          // Track slow resources
          if (resourceEntry.duration > 1000) { // 1 second
            this.trackCustomMetric({
              name: 'slow_resource_load',
              value: resourceEntry.duration,
              unit: 'ms',
              tags: {
                resource_url: resourceEntry.name,
                resource_type: this.getResourceType(resourceEntry.name)
              },
              timestamp: Date.now()
            });
          }
        }
      }
    });

    observer.observe({ entryTypes: ['resource'] });
  }

  private observeNavigationTiming(): void {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'navigation') {
          const navEntry = entry as PerformanceNavigationTiming;
          
          // Track navigation phases
          this.trackCustomMetric({
            name: 'dns_lookup_time',
            value: navEntry.domainLookupEnd - navEntry.domainLookupStart,
            unit: 'ms',
            tags: { navigation_type: navEntry.type },
            timestamp: Date.now()
          });

          this.trackCustomMetric({
            name: 'tcp_connection_time',
            value: navEntry.connectEnd - navEntry.connectStart,
            unit: 'ms',
            tags: { navigation_type: navEntry.type },
            timestamp: Date.now()
          });

          this.trackCustomMetric({
            name: 'server_response_time',
            value: navEntry.responseEnd - navEntry.requestStart,
            unit: 'ms',
            tags: { navigation_type: navEntry.type },
            timestamp: Date.now()
          });

          this.trackCustomMetric({
            name: 'dom_processing_time',
            value: navEntry.domComplete - navEntry.domLoading,
            unit: 'ms',
            tags: { navigation_type: navEntry.type },
            timestamp: Date.now()
          });
        }
      }
    });

    observer.observe({ entryTypes: ['navigation'] });
  }

  private observeLongTasks(): void {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'longtask') {
          this.trackCustomMetric({
            name: 'long_task',
            value: entry.duration,
            unit: 'ms',
            tags: {
              task_type: (entry as any).attribution?.[0]?.name || 'unknown'
            },
            timestamp: Date.now()
          });
        }
      }
    });

    if ('PerformanceObserver' in window) {
      try {
        observer.observe({ entryTypes: ['longtask'] });
      } catch (e) {
        // Long task API not supported
      }
    }
  }

  private observeLayoutShifts(): void {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'layout-shift' && !(entry as any).hadRecentInput) {
          this.trackCustomMetric({
            name: 'layout_shift',
            value: (entry as any).value,
            unit: 'count',
            tags: {
              sources: (entry as any).sources?.map((s: any) => s.node?.tagName).join(',') || 'unknown'
            },
            timestamp: Date.now()
          });
        }
      }
    });

    observer.observe({ entryTypes: ['layout-shift'] });
  }

  private observePaintTiming(): void {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'paint') {
          this.trackCustomMetric({
            name: entry.name.replace('-', '_'),
            value: entry.startTime,
            unit: 'ms',
            tags: { paint_type: entry.name },
            timestamp: Date.now()
          });
        }
      }
    });

    observer.observe({ entryTypes: ['paint'] });
  }

  private observeMemoryUsage(): void {
    if ('memory' in performance) {
      const checkMemory = () => {
        const memInfo = (performance as any).memory;
        
        this.trackCustomMetric({
          name: 'heap_used',
          value: memInfo.usedJSHeapSize,
          unit: 'bytes',
          tags: { memory_type: 'heap' },
          timestamp: Date.now()
        });

        this.trackCustomMetric({
          name: 'heap_total',
          value: memInfo.totalJSHeapSize,
          unit: 'bytes',
          tags: { memory_type: 'heap' },
          timestamp: Date.now()
        });

        this.trackCustomMetric({
          name: 'heap_limit',
          value: memInfo.jsHeapSizeLimit,
          unit: 'bytes',
          tags: { memory_type: 'heap' },
          timestamp: Date.now()
        });
      };

      // Check memory usage every 30 seconds
      setInterval(checkMemory, 30000);
      checkMemory(); // Initial check
    }
  }

  private setupErrorTracking(): void {
    window.addEventListener('error', (event) => {
      const error: ErrorMetric = {
        message: event.message,
        source: event.filename,
        lineno: event.lineno,
        colno: event.colno,
        stack: event.error?.stack,
        timestamp: Date.now(),
        sessionId: this.sessionId,
        userId: this.userId,
        url: window.location.href,
        userAgent: navigator.userAgent,
        severity: this.calculateErrorSeverity(event.message)
      };

      this.errors.push(error);
      this.checkFlushCondition();
    });
  }

  private setupUnhandledRejectionTracking(): void {
    window.addEventListener('unhandledrejection', (event) => {
      const error: ErrorMetric = {
        message: event.reason?.message || 'Unhandled Promise Rejection',
        stack: event.reason?.stack,
        source: 'promise',
        lineno: 0,
        colno: 0,
        timestamp: Date.now(),
        sessionId: this.sessionId,
        userId: this.userId,
        url: window.location.href,
        userAgent: navigator.userAgent,
        severity: 'medium'
      };

      this.errors.push(error);
      this.checkFlushCondition();
    });
  }

  private setupInteractionTracking(): void {
    const trackInteraction = (eventType: string) => {
      const startTime = performance.now();
      
      setTimeout(() => {
        const duration = performance.now() - startTime;
        
        this.trackCustomMetric({
          name: 'user_interaction',
          value: duration,
          unit: 'ms',
          tags: {
            interaction_type: eventType,
            target_element: document.activeElement?.tagName || 'unknown'
          },
          timestamp: Date.now()
        });
      }, 0);
    };

    ['click', 'keydown', 'touchstart'].forEach(eventType => {
      document.addEventListener(eventType, () => trackInteraction(eventType), { passive: true });
    });
  }

  public trackCustomMetric(metric: CustomMetric): void {
    this.customMetrics.push(metric);
    this.checkFlushCondition();
  }

  public trackError(error: Partial<ErrorMetric>): void {
    const errorMetric: ErrorMetric = {
      message: error.message || 'Unknown error',
      source: error.source || 'manual',
      lineno: error.lineno || 0,
      colno: error.colno || 0,
      timestamp: Date.now(),
      sessionId: this.sessionId,
      userId: this.userId,
      url: window.location.href,
      userAgent: navigator.userAgent,
      severity: error.severity || 'medium',
      ...error
    };

    this.errors.push(errorMetric);
    this.checkFlushCondition();
  }

  public setUser(userId: string): void {
    this.userId = userId;
  }

  private checkFlushCondition(): void {
    const totalMetrics = this.metrics.length + this.customMetrics.length + this.errors.length;
    if (totalMetrics >= this.batchSize) {
      this.flush();
    }
  }

  private setupPeriodicFlush(): void {
    setInterval(() => {
      this.flush();
    }, this.flushInterval);
  }

  private setupUnloadHandler(): void {
    window.addEventListener('beforeunload', () => {
      this.flush(true); // Synchronous flush on unload
    });

    // Use Page Visibility API for better mobile support
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        this.flush(true);
      }
    });
  }

  private async flush(synchronous = false): Promise<void> {
    if (this.metrics.length === 0 && this.customMetrics.length === 0 && this.errors.length === 0) {
      return;
    }

    const payload = {
      sessionId: this.sessionId,
      userId: this.userId,
      environment: this.environment,
      version: this.version,
      timestamp: Date.now(),
      metrics: [...this.metrics],
      customMetrics: [...this.customMetrics],
      errors: [...this.errors],
      userAgent: navigator.userAgent,
      url: window.location.href,
      viewport: {
        width: window.innerWidth,
        height: window.innerHeight
      },
      screen: {
        width: screen.width,
        height: screen.height,
        colorDepth: screen.colorDepth
      },
      connection: (navigator as any).connection
    };

    // Clear metrics after copying
    this.metrics = [];
    this.customMetrics = [];
    this.errors = [];

    try {
      if (synchronous && 'sendBeacon' in navigator) {
        // Use sendBeacon for synchronous sending during page unload
        navigator.sendBeacon(
          this.endpoint,
          new Blob([JSON.stringify(payload)], { type: 'application/json' })
        );
      } else {
        // Use fetch for regular sending
        await fetch(this.endpoint, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.apiKey}`
          },
          body: JSON.stringify(payload),
          keepalive: synchronous
        });
      }
    } catch (error) {
      console.warn('Failed to send performance metrics:', error);
      // In production, you might want to queue failed metrics for retry
    }
  }

  private generateSessionId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  private getResourceType(url: string): string {
    if (url.includes('.js')) return 'script';
    if (url.includes('.css')) return 'stylesheet';
    if (url.match(/\.(png|jpg|jpeg|gif|svg|webp)$/)) return 'image';
    if (url.match(/\.(woff|woff2|ttf|eot)$/)) return 'font';
    if (url.includes('/api/')) return 'api';
    return 'other';
  }

  private calculateErrorSeverity(message: string): 'low' | 'medium' | 'high' | 'critical' {
    const criticalKeywords = ['security', 'auth', 'payment', 'crash'];
    const highKeywords = ['network', 'timeout', 'failed', 'error'];
    const mediumKeywords = ['warning', 'deprecated'];

    const lowerMessage = message.toLowerCase();

    if (criticalKeywords.some(keyword => lowerMessage.includes(keyword))) {
      return 'critical';
    }
    if (highKeywords.some(keyword => lowerMessage.includes(keyword))) {
      return 'high';
    }
    if (mediumKeywords.some(keyword => lowerMessage.includes(keyword))) {
      return 'medium';
    }
    return 'low';
  }
}

// React Hook for Performance Tracking
export function usePerformanceTracker() {
  const [tracker] = useState(() => new AdvancedPerformanceTracker({
    endpoint: process.env.REACT_APP_METRICS_ENDPOINT!,
    apiKey: process.env.REACT_APP_METRICS_API_KEY!,
    environment: process.env.NODE_ENV,
    version: process.env.REACT_APP_VERSION || '1.0.0'
  }));

  const trackPageView = useCallback((pageName: string) => {
    tracker.trackCustomMetric({
      name: 'page_view',
      value: 1,
      unit: 'count',
      tags: { page: pageName },
      timestamp: Date.now()
    });
  }, [tracker]);

  const trackUserAction = useCallback((action: string, value?: number) => {
    tracker.trackCustomMetric({
      name: 'user_action',
      value: value || 1,
      unit: value ? 'ms' : 'count',
      tags: { action },
      timestamp: Date.now()
    });
  }, [tracker]);

  const trackError = useCallback((error: Error, context?: Record<string, string>) => {
    tracker.trackError({
      message: error.message,
      stack: error.stack,
      severity: 'high',
      componentStack: context?.componentStack,
      errorBoundary: context?.errorBoundary
    });
  }, [tracker]);

  return { trackPageView, trackUserAction, trackError, tracker };
}

export default AdvancedPerformanceTracker;
```

## Real User Monitoring (RUM) Implementation

### Advanced RUM with Session Replay

```typescript
// monitoring/rum-tracker.ts
interface SessionReplayEvent {
  type: 'dom' | 'input' | 'scroll' | 'click' | 'resize';
  timestamp: number;
  data: any;
  selector?: string;
  value?: string;
  coordinates?: { x: number; y: number };
}

interface UserSession {
  sessionId: string;
  userId?: string;
  startTime: number;
  endTime?: number;
  pageViews: string[];
  events: SessionReplayEvent[];
  errors: ErrorMetric[];
  performance: PerformanceMetric[];
  userAgent: string;
  viewport: { width: number; height: number };
  referrer: string;
  location: string;
}

class RealUserMonitoring {
  private session: UserSession;
  private recordingEnabled: boolean = true;
  private maxEvents: number = 1000;
  private maxSessionDuration: number = 30 * 60 * 1000; // 30 minutes
  private domObserver?: MutationObserver;
  private lastActivity: number = Date.now();
  private sessionTimeout: number = 30 * 60 * 1000; // 30 minutes of inactivity

  constructor(private config: {
    endpoint: string;
    apiKey: string;
    sampleRate: number;
    enableSessionReplay: boolean;
    enableHeatmaps: boolean;
    privacyMode: 'strict' | 'balanced' | 'permissive';
  }) {
    this.session = this.initializeSession();
    
    if (Math.random() > config.sampleRate) {
      this.recordingEnabled = false;
      return;
    }

    this.startTracking();
  }

  private initializeSession(): UserSession {
    return {
      sessionId: this.generateSessionId(),
      startTime: Date.now(),
      pageViews: [window.location.href],
      events: [],
      errors: [],
      performance: [],
      userAgent: navigator.userAgent,
      viewport: {
        width: window.innerWidth,
        height: window.innerHeight
      },
      referrer: document.referrer,
      location: window.location.href
    };
  }

  private startTracking(): void {
    if (!this.recordingEnabled) return;

    // Track DOM mutations for session replay
    if (this.config.enableSessionReplay) {
      this.setupDOMObserver();
    }

    // Track user interactions
    this.setupInteractionTracking();

    // Track page visibility changes
    this.setupVisibilityTracking();

    // Track form interactions
    this.setupFormTracking();

    // Track scroll behavior
    this.setupScrollTracking();

    // Track network requests
    this.setupNetworkTracking();

    // Setup session timeout
    this.setupSessionTimeout();

    // Track page unload
    this.setupUnloadTracking();
  }

  private setupDOMObserver(): void {
    this.domObserver = new MutationObserver((mutations) => {
      if (this.session.events.length >= this.maxEvents) return;

      const relevantMutations = mutations.filter(mutation => {
        const target = mutation.target as Element;
        return !this.shouldIgnoreElement(target);
      });

      if (relevantMutations.length > 0) {
        this.recordEvent({
          type: 'dom',
          timestamp: Date.now(),
          data: {
            mutations: relevantMutations.map(mutation => ({
              type: mutation.type,
              target: this.getElementSelector(mutation.target as Element),
              addedNodes: Array.from(mutation.addedNodes).map(node => 
                node.nodeType === Node.ELEMENT_NODE ? 
                this.getElementSelector(node as Element) : null
              ).filter(Boolean),
              removedNodes: Array.from(mutation.removedNodes).map(node => 
                node.nodeType === Node.ELEMENT_NODE ? 
                this.getElementSelector(node as Element) : null
              ).filter(Boolean)
            }))
          }
        });
      }
    });

    this.domObserver.observe(document.body, {
      childList: true,
      subtree: true,
      attributes: true,
      attributeOldValue: true,
      characterData: true,
      characterDataOldValue: true
    });
  }

  private setupInteractionTracking(): void {
    const trackInteraction = (event: Event) => {
      this.updateLastActivity();
      
      if (this.session.events.length >= this.maxEvents) return;

      const target = event.target as Element;
      if (this.shouldIgnoreElement(target)) return;

      let eventData: any = {
        selector: this.getElementSelector(target),
        tagName: target.tagName.toLowerCase()
      };

      if (event.type === 'click') {
        const rect = target.getBoundingClientRect();
        eventData.coordinates = {
          x: (event as MouseEvent).clientX - rect.left,
          y: (event as MouseEvent).clientY - rect.top
        };
      }

      if (event.type === 'input' && target instanceof HTMLInputElement) {
        if (this.config.privacyMode !== 'permissive') {
          // Mask sensitive input data
          eventData.value = this.maskSensitiveData(target);
        } else {
          eventData.value = target.value;
        }
      }

      this.recordEvent({
        type: event.type as any,
        timestamp: Date.now(),
        data: eventData
      });
    };

    ['click', 'input', 'change', 'submit'].forEach(eventType => {
      document.addEventListener(eventType, trackInteraction, { 
        passive: true, 
        capture: true 
      });
    });
  }

  private setupVisibilityTracking(): void {
    document.addEventListener('visibilitychange', () => {
      this.recordEvent({
        type: 'dom',
        timestamp: Date.now(),
        data: {
          visibility: document.visibilityState,
          hidden: document.hidden
        }
      });

      if (document.visibilityState === 'hidden') {
        this.endSession();
      }
    });
  }

  private setupFormTracking(): void {
    document.addEventListener('submit', (event) => {
      const form = event.target as HTMLFormElement;
      const formData = new FormData(form);
      const fields: Record<string, string> = {};

      for (const [key, value] of formData.entries()) {
        if (this.config.privacyMode === 'strict') {
          fields[key] = '[REDACTED]';
        } else {
          fields[key] = this.maskSensitiveData({ name: key, value: value.toString() });
        }
      }

      this.recordEvent({
        type: 'dom',
        timestamp: Date.now(),
        data: {
          eventType: 'form_submit',
          selector: this.getElementSelector(form),
          fields
        }
      });
    });
  }

  private setupScrollTracking(): void {
    let scrollTimeout: number;
    let lastScrollY = window.scrollY;

    const trackScroll = () => {
      const scrollY = window.scrollY;
      const scrollDirection = scrollY > lastScrollY ? 'down' : 'up';
      const scrollPercentage = Math.round(
        (scrollY / (document.documentElement.scrollHeight - window.innerHeight)) * 100
      );

      this.recordEvent({
        type: 'scroll',
        timestamp: Date.now(),
        data: {
          scrollY,
          direction: scrollDirection,
          percentage: scrollPercentage
        }
      });

      lastScrollY = scrollY;
    };

    window.addEventListener('scroll', () => {
      this.updateLastActivity();
      
      clearTimeout(scrollTimeout);
      scrollTimeout = window.setTimeout(trackScroll, 100);
    }, { passive: true });
  }

  private setupNetworkTracking(): void {
    // Intercept fetch requests
    const originalFetch = window.fetch;
    window.fetch = async (...args) => {
      const startTime = Date.now();
      const url = args[0] instanceof Request ? args[0].url : args[0];
      
      try {
        const response = await originalFetch(...args);
        
        this.recordEvent({
          type: 'dom',
          timestamp: Date.now(),
          data: {
            eventType: 'network_request',
            url,
            method: args[1]?.method || 'GET',
            status: response.status,
            duration: Date.now() - startTime,
            size: parseInt(response.headers.get('content-length') || '0')
          }
        });

        return response;
      } catch (error) {
        this.recordEvent({
          type: 'dom',
          timestamp: Date.now(),
          data: {
            eventType: 'network_error',
            url,
            method: args[1]?.method || 'GET',
            error: error.message,
            duration: Date.now() - startTime
          }
        });
        throw error;
      }
    };

    // Intercept XMLHttpRequest
    const originalXHROpen = XMLHttpRequest.prototype.open;
    const originalXHRSend = XMLHttpRequest.prototype.send;

    XMLHttpRequest.prototype.open = function(method: string, url: string | URL) {
      (this as any).__url = url;
      (this as any).__method = method;
      (this as any).__startTime = Date.now();
      return originalXHROpen.apply(this, arguments as any);
    };

    XMLHttpRequest.prototype.send = function() {
      const xhr = this;
      
      xhr.addEventListener('loadend', () => {
        this.recordEvent({
          type: 'dom',
          timestamp: Date.now(),
          data: {
            eventType: 'xhr_request',
            url: (xhr as any).__url,
            method: (xhr as any).__method,
            status: xhr.status,
            duration: Date.now() - (xhr as any).__startTime
          }
        });
      });

      return originalXHRSend.apply(this, arguments as any);
    };
  }

  private setupSessionTimeout(): void {
    setInterval(() => {
      if (Date.now() - this.lastActivity > this.sessionTimeout) {
        this.endSession();
      }
    }, 60000); // Check every minute
  }

  private setupUnloadTracking(): void {
    window.addEventListener('beforeunload', () => {
      this.endSession();
    });
  }

  private recordEvent(event: SessionReplayEvent): void {
    if (!this.recordingEnabled || this.session.events.length >= this.maxEvents) return;

    this.session.events.push(event);
    
    // Send data if session is too long or has too many events
    if (Date.now() - this.session.startTime > this.maxSessionDuration ||
        this.session.events.length >= this.maxEvents) {
      this.endSession();
    }
  }

  private shouldIgnoreElement(element: Element): boolean {
    if (!element || !element.tagName) return true;

    // Ignore script tags, style tags, and hidden elements
    const ignoredTags = ['SCRIPT', 'STYLE', 'NOSCRIPT'];
    if (ignoredTags.includes(element.tagName)) return true;

    // Ignore elements with data-rum-ignore attribute
    if (element.hasAttribute('data-rum-ignore')) return true;

    // Ignore password fields and other sensitive inputs
    if (element instanceof HTMLInputElement) {
      const sensitiveTypes = ['password', 'credit-card-number', 'social-security-number'];
      if (sensitiveTypes.includes(element.type) || 
          sensitiveTypes.some(type => element.autocomplete.includes(type))) {
        return true;
      }
    }

    return false;
  }

  private getElementSelector(element: Element): string {
    if (!element || element === document.body) return 'body';

    // Try to get a unique selector
    if (element.id) return `#${element.id}`;

    let selector = element.tagName.toLowerCase();
    
    if (element.className) {
      selector += `.${element.className.split(' ').join('.')}`;
    }

    // Add parent context for uniqueness
    const parent = element.parentElement;
    if (parent && parent !== document.body) {
      const parentSelector = this.getElementSelector(parent);
      return `${parentSelector} > ${selector}`;
    }

    return selector;
  }

  private maskSensitiveData(field: { name?: string; value: string; type?: string }): string {
    const sensitivePatterns = [
      /password/i,
      /email/i,
      /credit.*card/i,
      /ssn/i,
      /social.*security/i,
      /phone/i,
      /address/i
    ];

    const fieldName = field.name || field.type || '';
    if (sensitivePatterns.some(pattern => pattern.test(fieldName))) {
      return '[REDACTED]';
    }

    // Mask email addresses
    if (field.value.includes('@')) {
      return field.value.replace(/[^\s@]+@[^\s@]+\.[^\s@]+/g, '[EMAIL]');
    }

    // Mask phone numbers
    if (/\d{3,}/.test(field.value)) {
      return field.value.replace(/\d/g, '*');
    }

    return field.value;
  }

  private updateLastActivity(): void {
    this.lastActivity = Date.now();
  }

  private endSession(): void {
    if (!this.recordingEnabled) return;

    this.session.endTime = Date.now();
    this.sendSessionData();
    
    // Cleanup
    this.domObserver?.disconnect();
    this.recordingEnabled = false;
  }

  private async sendSessionData(): Promise<void> {
    try {
      const payload = {
        ...this.session,
        environment: process.env.NODE_ENV,
        version: process.env.REACT_APP_VERSION
      };

      await fetch(this.config.endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.config.apiKey}`
        },
        body: JSON.stringify(payload),
        keepalive: true
      });
    } catch (error) {
      console.warn('Failed to send RUM data:', error);
    }
  }

  private generateSessionId(): string {
    return `rum-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  public trackPageView(url: string): void {
    if (!this.recordingEnabled) return;

    this.session.pageViews.push(url);
    this.updateLastActivity();
    
    this.recordEvent({
      type: 'dom',
      timestamp: Date.now(),
      data: {
        eventType: 'page_view',
        url,
        title: document.title,
        referrer: document.referrer
      }
    });
  }

  public addUserContext(userId: string, properties?: Record<string, any>): void {
    this.session.userId = userId;
    
    if (properties) {
      this.recordEvent({
        type: 'dom',
        timestamp: Date.now(),
        data: {
          eventType: 'user_context',
          userId,
          properties
        }
      });
    }
  }
}

export default RealUserMonitoring;
```

## Synthetic Monitoring and Alerting

### Automated Performance Testing Pipeline

```typescript
// monitoring/synthetic-monitoring.ts
import puppeteer from 'puppeteer';
import lighthouse from 'lighthouse';

interface SyntheticTestConfig {
  url: string;
  device: 'desktop' | 'mobile';
  location: string;
  frequency: number; // minutes
  thresholds: {
    fcp: number;
    lcp: number;
    cls: number;
    fid: number;
    speed_index: number;
    availability: number;
  };
  alerts: {
    email: string[];
    slack?: string;
    webhook?: string;
  };
}

interface TestResult {
  timestamp: number;
  url: string;
  device: string;
  location: string;
  metrics: {
    fcp: number;
    lcp: number;
    cls: number;
    fid: number;
    speed_index: number;
    total_blocking_time: number;
    cumulative_layout_shift: number;
    first_meaningful_paint: number;
    time_to_interactive: number;
  };
  lighthouse_score: number;
  availability: boolean;
  response_time: number;
  error?: string;
  screenshots: {
    desktop?: string;
    mobile?: string;
  };
}

class SyntheticMonitoring {
  private tests: Map<string, SyntheticTestConfig> = new Map();
  private intervals: Map<string, NodeJS.Timeout> = new Map();
  private results: TestResult[] = [];

  constructor(private config: {
    endpoint: string;
    apiKey: string;
    alerting: {
      email: {
        service: string;
        user: string;
        pass: string;
      };
      slack?: {
        webhook: string;
      };
    };
  }) {}

  public addTest(testId: string, testConfig: SyntheticTestConfig): void {
    this.tests.set(testId, testConfig);
    this.scheduleTest(testId, testConfig);
  }

  public removeTest(testId: string): void {
    const interval = this.intervals.get(testId);
    if (interval) {
      clearInterval(interval);
      this.intervals.delete(testId);
    }
    this.tests.delete(testId);
  }

  private scheduleTest(testId: string, config: SyntheticTestConfig): void {
    const interval = setInterval(() => {
      this.runTest(testId, config);
    }, config.frequency * 60 * 1000);

    this.intervals.set(testId, interval);
    
    // Run initial test immediately
    this.runTest(testId, config);
  }

  private async runTest(testId: string, config: SyntheticTestConfig): Promise<void> {
    console.log(`Running synthetic test: ${testId} for ${config.url}`);

    try {
      const result = await this.performLighthouseTest(config);
      this.results.push(result);
      
      // Check thresholds and send alerts if needed
      await this.checkThresholds(testId, config, result);
      
      // Send results to monitoring endpoint
      await this.sendResults([result]);
      
    } catch (error) {
      console.error(`Synthetic test failed for ${testId}:`, error);
      
      const errorResult: TestResult = {
        timestamp: Date.now(),
        url: config.url,
        device: config.device,
        location: config.location,
        metrics: {
          fcp: 0,
          lcp: 0,
          cls: 0,
          fid: 0,
          speed_index: 0,
          total_blocking_time: 0,
          cumulative_layout_shift: 0,
          first_meaningful_paint: 0,
          time_to_interactive: 0
        },
        lighthouse_score: 0,
        availability: false,
        response_time: 0,
        error: error.message,
        screenshots: {}
      };
      
      this.results.push(errorResult);
      await this.sendAlert(testId, config, errorResult, 'Test execution failed');
    }
  }

  private async performLighthouseTest(config: SyntheticTestConfig): Promise<TestResult> {
    const browser = await puppeteer.launch({
      headless: true,
      args: ['--no-sandbox', '--disable-dev-shm-usage']
    });

    try {
      const page = await browser.newPage();
      
      // Configure device emulation
      if (config.device === 'mobile') {
        await page.emulate(puppeteer.devices['iPhone X']);
      } else {
        await page.setViewport({ width: 1920, height: 1080 });
      }

      const startTime = Date.now();
      
      // Navigate to page and wait for load
      const response = await page.goto(config.url, { 
        waitUntil: 'networkidle0',
        timeout: 30000 
      });
      
      const responseTime = Date.now() - startTime;
      const availability = response?.ok() || false;

      // Take screenshot
      const screenshot = await page.screenshot({
        fullPage: true,
        encoding: 'base64'
      });

      // Run Lighthouse audit
      const { lhr } = await lighthouse(config.url, {
        port: (await browser.wsEndpoint()).split(':')[2],
        output: 'json',
        logLevel: 'error',
        onlyCategories: ['performance'],
        settings: {
          emulatedFormFactor: config.device,
          throttling: config.device === 'mobile' ? 
            lighthouse.constants.throttling.mobileSlow4G : 
            lighthouse.constants.throttling.desktopDense4G
        }
      });

      const metrics = lhr.audits;
      
      return {
        timestamp: Date.now(),
        url: config.url,
        device: config.device,
        location: config.location,
        metrics: {
          fcp: metrics['first-contentful-paint']?.numericValue || 0,
          lcp: metrics['largest-contentful-paint']?.numericValue || 0,
          cls: metrics['cumulative-layout-shift']?.numericValue || 0,
          fid: metrics['max-potential-fid']?.numericValue || 0,
          speed_index: metrics['speed-index']?.numericValue || 0,
          total_blocking_time: metrics['total-blocking-time']?.numericValue || 0,
          cumulative_layout_shift: metrics['cumulative-layout-shift']?.numericValue || 0,
          first_meaningful_paint: metrics['first-meaningful-paint']?.numericValue || 0,
          time_to_interactive: metrics['interactive']?.numericValue || 0
        },
        lighthouse_score: lhr.categories.performance.score * 100,
        availability,
        response_time: responseTime,
        screenshots: {
          [config.device]: screenshot
        }
      };

    } finally {
      await browser.close();
    }
  }

  private async checkThresholds(
    testId: string, 
    config: SyntheticTestConfig, 
    result: TestResult
  ): Promise<void> {
    const failures: string[] = [];

    // Check availability
    if (!result.availability) {
      failures.push(`Site is down (availability: ${result.availability})`);
    }

    // Check performance thresholds
    if (result.metrics.fcp > config.thresholds.fcp) {
      failures.push(`FCP too slow: ${result.metrics.fcp}ms (threshold: ${config.thresholds.fcp}ms)`);
    }

    if (result.metrics.lcp > config.thresholds.lcp) {
      failures.push(`LCP too slow: ${result.metrics.lcp}ms (threshold: ${config.thresholds.lcp}ms)`);
    }

    if (result.metrics.cls > config.thresholds.cls) {
      failures.push(`CLS too high: ${result.metrics.cls} (threshold: ${config.thresholds.cls})`);
    }

    if (result.lighthouse_score < config.thresholds.speed_index) {
      failures.push(`Lighthouse score too low: ${result.lighthouse_score} (threshold: ${config.thresholds.speed_index})`);
    }

    if (failures.length > 0) {
      await this.sendAlert(testId, config, result, failures.join(', '));
    }
  }

  private async sendAlert(
    testId: string,
    config: SyntheticTestConfig,
    result: TestResult,
    message: string
  ): Promise<void> {
    const alertData = {
      testId,
      url: config.url,
      device: config.device,
      location: config.location,
      timestamp: new Date(result.timestamp).toISOString(),
      message,
      metrics: result.metrics,
      lighthouse_score: result.lighthouse_score,
      availability: result.availability
    };

    // Send email alerts
    if (config.alerts.email.length > 0) {
      await this.sendEmailAlert(config.alerts.email, alertData);
    }

    // Send Slack alert
    if (config.alerts.slack) {
      await this.sendSlackAlert(config.alerts.slack, alertData);
    }

    // Send webhook alert
    if (config.alerts.webhook) {
      await this.sendWebhookAlert(config.alerts.webhook, alertData);
    }
  }

  private async sendEmailAlert(emails: string[], alertData: any): Promise<void> {
    // Email sending implementation would go here
    // Using nodemailer or similar service
    console.log('Sending email alert to:', emails, alertData);
  }

  private async sendSlackAlert(webhook: string, alertData: any): Promise<void> {
    const slackMessage = {
      text: `🚨 Performance Alert: ${alertData.testId}`,
      attachments: [{
        color: 'danger',
        fields: [
          { title: 'URL', value: alertData.url, short: true },
          { title: 'Device', value: alertData.device, short: true },
          { title: 'Message', value: alertData.message, short: false },
          { title: 'Lighthouse Score', value: `${alertData.lighthouse_score}/100`, short: true },
          { title: 'Availability', value: alertData.availability ? '✅' : '❌', short: true }
        ],
        ts: Math.floor(Date.now() / 1000)
      }]
    };

    try {
      await fetch(webhook, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(slackMessage)
      });
    } catch (error) {
      console.error('Failed to send Slack alert:', error);
    }
  }

  private async sendWebhookAlert(webhook: string, alertData: any): Promise<void> {
    try {
      await fetch(webhook, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(alertData)
      });
    } catch (error) {
      console.error('Failed to send webhook alert:', error);
    }
  }

  private async sendResults(results: TestResult[]): Promise<void> {
    try {
      await fetch(this.config.endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.config.apiKey}`
        },
        body: JSON.stringify({
          type: 'synthetic_monitoring',
          results,
          timestamp: Date.now()
        })
      });
    } catch (error) {
      console.error('Failed to send synthetic monitoring results:', error);
    }
  }

  public getResults(testId?: string, limit = 100): TestResult[] {
    let filtered = this.results;
    
    if (testId) {
      const config = this.tests.get(testId);
      if (config) {
        filtered = this.results.filter(r => r.url === config.url);
      }
    }
    
    return filtered
      .sort((a, b) => b.timestamp - a.timestamp)
      .slice(0, limit);
  }

  public getAverageMetrics(testId: string, hours = 24): any {
    const config = this.tests.get(testId);
    if (!config) return null;

    const cutoff = Date.now() - (hours * 60 * 60 * 1000);
    const recentResults = this.results.filter(r => 
      r.url === config.url && r.timestamp > cutoff && !r.error
    );

    if (recentResults.length === 0) return null;

    const metrics = recentResults.reduce((acc, result) => {
      Object.keys(result.metrics).forEach(key => {
        acc[key] = (acc[key] || 0) + result.metrics[key];
      });
      acc.lighthouse_score = (acc.lighthouse_score || 0) + result.lighthouse_score;
      return acc;
    }, {} as any);

    Object.keys(metrics).forEach(key => {
      metrics[key] = Math.round(metrics[key] / recentResults.length);
    });

    return {
      ...metrics,
      sample_count: recentResults.length,
      time_range: `${hours} hours`
    };
  }
}

export default SyntheticMonitoring;
```

## Interview-Ready Monitoring & Observability Summary

**Core Monitoring Components:**
1. **APM (Application Performance Monitoring)** - Real-time performance tracking with Core Web Vitals, custom metrics, and error monitoring
2. **RUM (Real User Monitoring)** - Session replay, user interaction tracking, and privacy-aware data collection
3. **Synthetic Monitoring** - Automated testing with Lighthouse integration and intelligent alerting
4. **Observability Pipeline** - Comprehensive data collection, analysis, and visualization

**Enterprise Features:**
- Multi-dimensional performance tracking with device, network, and user context
- Privacy-compliant session recording with configurable data masking
- Automated performance regression detection and alerting
- Real-time error tracking with severity classification and context capture

**Production Capabilities:**
- Scalable event batching and reliable data transmission
- Session timeout management and resource cleanup
- Network request interception and performance correlation
- Form interaction tracking with PII protection

**Key Interview Topics:** Performance monitoring strategies, observability architecture design, privacy and compliance in user tracking, alerting and incident response automation, synthetic vs real user monitoring trade-offs, performance budgets and SLA management.