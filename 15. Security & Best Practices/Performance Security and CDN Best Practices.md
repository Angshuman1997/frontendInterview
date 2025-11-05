# Performance Security and CDN Best Practices

This guide covers the intersection of performance optimization and security, including secure CDN configuration, content delivery security, asset protection, and performance monitoring without compromising security.

## Secure Content Delivery Network (CDN) Implementation

### 1. CDN Security Configuration
```typescript
// CDN security headers and configuration
interface CDNSecurityConfig {
  domain: string;
  securityHeaders: Record<string, string>;
  corsPolicy: {
    allowedOrigins: string[];
    allowedMethods: string[];
    allowedHeaders: string[];
  };
  rateLimiting: {
    requestsPerSecond: number;
    burstCapacity: number;
  };
  geoBlocking: {
    allowedCountries: string[];
    blockedCountries: string[];
  };
  sriEnabled: boolean;
  hotlinkProtection: boolean;
}

// Cloudflare security configuration
const cloudflareSecurityConfig: CDNSecurityConfig = {
  domain: 'cdn.example.com',
  securityHeaders: {
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Content-Security-Policy': "default-src 'self'; img-src 'self' data: https:; script-src 'self' 'unsafe-inline' 'unsafe-eval'",
    'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
  },
  corsPolicy: {
    allowedOrigins: ['https://example.com', 'https://www.example.com'],
    allowedMethods: ['GET', 'HEAD', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
  },
  rateLimiting: {
    requestsPerSecond: 100,
    burstCapacity: 200,
  },
  geoBlocking: {
    allowedCountries: ['US', 'CA', 'GB', 'DE', 'FR', 'AU'],
    blockedCountries: [],
  },
  sriEnabled: true,
  hotlinkProtection: true,
};

// CDN security manager
class CDNSecurityManager {
  private config: CDNSecurityConfig;

  constructor(config: CDNSecurityConfig) {
    this.config = config;
  }

  // Generate SRI hashes for assets
  generateSRIHash(content: string): string {
    const crypto = require('crypto');
    const hash = crypto.createHash('sha384').update(content).digest('base64');
    return `sha384-${hash}`;
  }

  // Generate secure asset URLs with signed tokens
  generateSecureAssetURL(assetPath: string, expirationTime?: number): string {
    const crypto = require('crypto');
    const secret = process.env.CDN_SIGNING_SECRET!;
    const expiry = expirationTime || Math.floor(Date.now() / 1000) + 3600; // 1 hour default
    
    const stringToSign = `${assetPath}${expiry}`;
    const signature = crypto
      .createHmac('sha256', secret)
      .update(stringToSign)
      .digest('hex')
      .substring(0, 16);

    return `https://${this.config.domain}${assetPath}?expires=${expiry}&signature=${signature}`;
  }

  // Validate asset request signature
  validateAssetSignature(assetPath: string, expires: string, signature: string): boolean {
    const crypto = require('crypto');
    const secret = process.env.CDN_SIGNING_SECRET!;
    const currentTime = Math.floor(Date.now() / 1000);
    const expiryTime = parseInt(expires, 10);

    // Check if URL has expired
    if (currentTime > expiryTime) {
      return false;
    }

    // Verify signature
    const stringToSign = `${assetPath}${expires}`;
    const expectedSignature = crypto
      .createHmac('sha256', secret)
      .update(stringToSign)
      .digest('hex')
      .substring(0, 16);

    return signature === expectedSignature;
  }

  // Configure WAF rules for CDN
  getWAFRules(): any[] {
    return [
      {
        name: 'Block SQL Injection',
        pattern: /(\b(union|select|insert|delete|update|drop|exec|script)\b)|(\b(or|and)\s+\d+\s*=\s*\d+)/i,
        action: 'block',
      },
      {
        name: 'Block XSS Attempts',
        pattern: /<script[^>]*>.*?<\/script>|javascript:|data:text\/html|vbscript:|on\w+\s*=/i,
        action: 'block',
      },
      {
        name: 'Block File Inclusion',
        pattern: /\.\.\//,
        action: 'block',
      },
      {
        name: 'Rate Limit API Calls',
        pattern: /^\/api\//,
        action: 'rate_limit',
        limit: 10,
        window: 60,
      },
      {
        name: 'Block Suspicious User Agents',
        pattern: /(sqlmap|nikto|nessus|openvas|nmap)/i,
        action: 'block',
      },
    ];
  }

  // Implement cache poisoning protection
  getCachePoisoningProtection(): any {
    return {
      // Normalize query parameters to prevent cache key manipulation
      normalizeQueryParams: true,
      
      // Ignore specific headers that shouldn't affect caching
      ignoreHeaders: [
        'X-Forwarded-For',
        'X-Real-IP',
        'User-Agent',
        'Accept-Language',
        'Accept-Encoding',
      ],
      
      // Cache key components
      cacheKeyComponents: [
        'url',
        'method',
        'headers:content-type',
        'headers:authorization',
      ],
      
      // Prevent cache poisoning via headers
      securityHeaders: {
        'Cache-Control': 'public, max-age=31536000, immutable',
        'Vary': 'Accept-Encoding',
      },
    };
  }
}

// Secure asset loading with SRI
class SecureAssetLoader {
  private sriHashes = new Map<string, string>();
  private cdnManager: CDNSecurityManager;

  constructor(cdnManager: CDNSecurityManager) {
    this.cdnManager = cdnManager;
  }

  // Load CSS with SRI
  loadSecureStylesheet(href: string, integrity?: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const link = document.createElement('link');
      link.rel = 'stylesheet';
      link.href = href;
      
      if (integrity) {
        link.integrity = integrity;
        link.crossOrigin = 'anonymous';
      }

      link.onload = () => resolve();
      link.onerror = () => reject(new Error(`Failed to load stylesheet: ${href}`));

      document.head.appendChild(link);
    });
  }

  // Load JavaScript with SRI
  loadSecureScript(src: string, integrity?: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const script = document.createElement('script');
      script.src = src;
      script.type = 'text/javascript';
      
      if (integrity) {
        script.integrity = integrity;
        script.crossOrigin = 'anonymous';
      }

      script.onload = () => resolve();
      script.onerror = () => reject(new Error(`Failed to load script: ${src}`));

      document.head.appendChild(script);
    });
  }

  // Generate and store SRI hash for runtime assets
  async generateAndStoreSRI(url: string): Promise<string> {
    try {
      const response = await fetch(url);
      const content = await response.text();
      const hash = this.cdnManager.generateSRIHash(content);
      this.sriHashes.set(url, hash);
      return hash;
    } catch (error) {
      console.error(`Failed to generate SRI hash for ${url}:`, error);
      throw error;
    }
  }

  // Validate asset integrity
  async validateAssetIntegrity(url: string, expectedHash?: string): Promise<boolean> {
    try {
      const storedHash = this.sriHashes.get(url);
      const hashToValidate = expectedHash || storedHash;
      
      if (!hashToValidate) {
        console.warn(`No SRI hash available for ${url}`);
        return false;
      }

      const response = await fetch(url);
      const content = await response.text();
      const actualHash = this.cdnManager.generateSRIHash(content);
      
      return `sha384-${actualHash}` === hashToValidate;
    } catch (error) {
      console.error(`Failed to validate asset integrity for ${url}:`, error);
      return false;
    }
  }
}

// React component for secure asset loading
import { useState, useEffect, useCallback } from 'react';

interface SecureAssetProps {
  src: string;
  integrity?: string;
  type: 'script' | 'stylesheet' | 'image';
  fallbackSrc?: string;
  onLoad?: () => void;
  onError?: (error: Error) => void;
}

const SecureAsset: React.FC<SecureAssetProps> = ({
  src,
  integrity,
  type,
  fallbackSrc,
  onLoad,
  onError,
}) => {
  const [isLoaded, setIsLoaded] = useState(false);
  const [hasError, setHasError] = useState(false);
  const [currentSrc, setCurrentSrc] = useState(src);

  const handleLoad = useCallback(() => {
    setIsLoaded(true);
    setHasError(false);
    onLoad?.();
  }, [onLoad]);

  const handleError = useCallback((error: Error) => {
    setHasError(true);
    
    // Try fallback source if available
    if (fallbackSrc && currentSrc !== fallbackSrc) {
      console.warn(`Asset failed to load: ${currentSrc}, trying fallback: ${fallbackSrc}`);
      setCurrentSrc(fallbackSrc);
      return;
    }
    
    console.error(`Asset failed to load: ${currentSrc}`, error);
    onError?.(error);
  }, [currentSrc, fallbackSrc, onError]);

  useEffect(() => {
    if (type === 'script') {
      const script = document.createElement('script');
      script.src = currentSrc;
      script.type = 'text/javascript';
      
      if (integrity) {
        script.integrity = integrity;
        script.crossOrigin = 'anonymous';
      }

      script.onload = handleLoad;
      script.onerror = () => handleError(new Error('Script load failed'));

      document.head.appendChild(script);

      return () => {
        document.head.removeChild(script);
      };
    } else if (type === 'stylesheet') {
      const link = document.createElement('link');
      link.rel = 'stylesheet';
      link.href = currentSrc;
      
      if (integrity) {
        link.integrity = integrity;
        link.crossOrigin = 'anonymous';
      }

      link.onload = handleLoad;
      link.onerror = () => handleError(new Error('Stylesheet load failed'));

      document.head.appendChild(link);

      return () => {
        document.head.removeChild(link);
      };
    }
  }, [currentSrc, integrity, type, handleLoad, handleError]);

  // For images, render directly with security attributes
  if (type === 'image') {
    return (
      <img
        src={currentSrc}
        onLoad={handleLoad}
        onError={() => handleError(new Error('Image load failed'))}
        crossOrigin="anonymous"
        referrerPolicy="strict-origin-when-cross-origin"
        alt=""
        style={{
          display: isLoaded ? 'block' : 'none',
        }}
      />
    );
  }

  return null;
};

// Performance monitoring with security considerations
class SecurePerformanceMonitor {
  private observer: PerformanceObserver | null = null;
  private metrics = new Map<string, any>();

  startMonitoring(): void {
    if ('PerformanceObserver' in window) {
      this.observer = new PerformanceObserver((list) => {
        const entries = list.getEntries();
        
        for (const entry of entries) {
          this.processPerformanceEntry(entry);
        }
      });

      // Monitor various performance entry types
      const entryTypes = ['navigation', 'resource', 'measure', 'paint'];
      
      for (const type of entryTypes) {
        try {
          this.observer.observe({ entryTypes: [type] });
        } catch (error) {
          console.warn(`Performance monitoring not supported for type: ${type}`);
        }
      }
    }
  }

  private processPerformanceEntry(entry: PerformanceEntry): void {
    // Filter out potentially sensitive information
    const sanitizedEntry = this.sanitizePerformanceEntry(entry);
    
    switch (entry.entryType) {
      case 'navigation':
        this.metrics.set('navigation', sanitizedEntry);
        break;
      case 'resource':
        this.trackResourcePerformance(sanitizedEntry as PerformanceResourceTiming);
        break;
      case 'paint':
        this.metrics.set(entry.name, sanitizedEntry);
        break;
    }
  }

  private sanitizePerformanceEntry(entry: PerformanceEntry): any {
    // Remove potentially sensitive data from performance entries
    const sanitized = {
      name: this.sanitizeURL(entry.name),
      entryType: entry.entryType,
      startTime: entry.startTime,
      duration: entry.duration,
    };

    // Add type-specific properties safely
    if (entry.entryType === 'resource') {
      const resourceEntry = entry as PerformanceResourceTiming;
      return {
        ...sanitized,
        transferSize: resourceEntry.transferSize,
        encodedBodySize: resourceEntry.encodedBodySize,
        decodedBodySize: resourceEntry.decodedBodySize,
        responseStart: resourceEntry.responseStart,
        responseEnd: resourceEntry.responseEnd,
      };
    }

    return sanitized;
  }

  private sanitizeURL(url: string): string {
    try {
      const urlObj = new URL(url);
      
      // Remove query parameters that might contain sensitive data
      const sensitiveParams = ['token', 'key', 'secret', 'password', 'auth'];
      
      for (const param of sensitiveParams) {
        urlObj.searchParams.delete(param);
      }

      // Mask user-specific information in path
      let pathname = urlObj.pathname;
      pathname = pathname.replace(/\/users\/[^\/]+/g, '/users/[USER_ID]');
      pathname = pathname.replace(/\/\d+/g, '/[ID]');
      
      return `${urlObj.protocol}//${urlObj.host}${pathname}${urlObj.search}`;
    } catch {
      // If URL parsing fails, return a generic identifier
      return '[INVALID_URL]';
    }
  }

  private trackResourcePerformance(entry: PerformanceResourceTiming): void {
    const url = this.sanitizeURL(entry.name);
    
    // Track resource loading performance
    const resourceMetrics = {
      url,
      duration: entry.duration,
      transferSize: entry.transferSize,
      cached: entry.transferSize === 0 && entry.decodedBodySize > 0,
      compressionRatio: entry.encodedBodySize > 0 ? 
        entry.decodedBodySize / entry.encodedBodySize : 1,
    };

    // Detect potential security issues
    this.detectSecurityIssues(entry, resourceMetrics);
    
    this.metrics.set(`resource_${url}`, resourceMetrics);
  }

  private detectSecurityIssues(entry: PerformanceResourceTiming, metrics: any): void {
    // Detect mixed content
    if (location.protocol === 'https:' && entry.name.startsWith('http:')) {
      console.warn('🔒 Mixed content detected:', metrics.url);
    }

    // Detect slow loading assets that might indicate tampering
    if (entry.duration > 5000) { // 5 seconds
      console.warn('⚠️ Slow loading asset detected:', metrics.url, entry.duration);
    }

    // Detect unexpected large assets
    if (entry.transferSize > 10 * 1024 * 1024) { // 10MB
      console.warn('📦 Large asset detected:', metrics.url, entry.transferSize);
    }

    // Detect poor compression ratios (potential tampering)
    if (metrics.compressionRatio < 0.5 && entry.transferSize > 1024) {
      console.warn('🗜️ Poor compression ratio:', metrics.url, metrics.compressionRatio);
    }
  }

  // Get performance report with security considerations
  getSecurePerformanceReport(): any {
    const report = {
      timestamp: new Date().toISOString(),
      navigationTiming: this.metrics.get('navigation'),
      paintTiming: {
        firstPaint: this.metrics.get('first-paint'),
        firstContentfulPaint: this.metrics.get('first-contentful-paint'),
      },
      resourceMetrics: Array.from(this.metrics.entries())
        .filter(([key]) => key.startsWith('resource_'))
        .map(([, value]) => value),
      securityIndicators: this.getSecurityIndicators(),
    };

    // Remove any remaining sensitive data
    return this.sanitizeReport(report);
  }

  private getSecurityIndicators(): any {
    return {
      httpsOnly: Array.from(this.metrics.entries())
        .filter(([key]) => key.startsWith('resource_'))
        .every(([, value]) => value.url.startsWith('https:') || value.url.startsWith('data:')),
      
      mixedContentCount: Array.from(this.metrics.entries())
        .filter(([key]) => key.startsWith('resource_'))
        .filter(([, value]) => 
          location.protocol === 'https:' && value.url.startsWith('http:')
        ).length,
      
      largeAssetsCount: Array.from(this.metrics.entries())
        .filter(([key]) => key.startsWith('resource_'))
        .filter(([, value]) => value.transferSize > 1024 * 1024).length,
    };
  }

  private sanitizeReport(report: any): any {
    // Final sanitization pass to ensure no sensitive data leaks
    const sanitized = JSON.parse(JSON.stringify(report));
    
    // Remove any fields that might contain sensitive information
    const sensitiveFields = ['token', 'key', 'secret', 'password', 'auth', 'session'];
    
    const removeSensitiveFields = (obj: any): any => {
      if (typeof obj !== 'object' || obj === null) return obj;
      
      for (const key in obj) {
        if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
          delete obj[key];
        } else if (typeof obj[key] === 'object') {
          obj[key] = removeSensitiveFields(obj[key]);
        }
      }
      
      return obj;
    };

    return removeSensitiveFields(sanitized);
  }

  stopMonitoring(): void {
    if (this.observer) {
      this.observer.disconnect();
      this.observer = null;
    }
  }
}
```

### 2. Secure Caching Strategies
```typescript
// Secure caching implementation
interface CacheSecurityPolicy {
  maxAge: number;
  sMaxAge: number;
  public: boolean;
  private: boolean;
  noCache: boolean;
  noStore: boolean;
  mustRevalidate: boolean;
  immutable: boolean;
  staleWhileRevalidate: number;
  staleIfError: number;
}

class SecureCacheManager {
  private cacheStorage: CacheStorage | null = null;
  private encryption = new (require('crypto-js')).AES;

  constructor() {
    this.initializeCacheStorage();
  }

  private async initializeCacheStorage(): Promise<void> {
    if ('caches' in window) {
      this.cacheStorage = window.caches;
    }
  }

  // Define security-aware cache policies
  getCachePolicy(resourceType: string, sensitivity: 'public' | 'private' | 'sensitive'): CacheSecurityPolicy {
    const basePolicies = {
      public: {
        maxAge: 31536000, // 1 year
        sMaxAge: 31536000,
        public: true,
        private: false,
        noCache: false,
        noStore: false,
        mustRevalidate: false,
        immutable: true,
        staleWhileRevalidate: 86400, // 1 day
        staleIfError: 604800, // 1 week
      },
      private: {
        maxAge: 3600, // 1 hour
        sMaxAge: 0,
        public: false,
        private: true,
        noCache: false,
        noStore: false,
        mustRevalidate: true,
        immutable: false,
        staleWhileRevalidate: 300, // 5 minutes
        staleIfError: 3600, // 1 hour
      },
      sensitive: {
        maxAge: 0,
        sMaxAge: 0,
        public: false,
        private: true,
        noCache: true,
        noStore: true,
        mustRevalidate: true,
        immutable: false,
        staleWhileRevalidate: 0,
        staleIfError: 0,
      },
    };

    const policy = basePolicies[sensitivity];

    // Adjust based on resource type
    if (resourceType === 'css' || resourceType === 'js') {
      policy.immutable = true;
      policy.maxAge = 31536000; // 1 year for versioned assets
    } else if (resourceType === 'html') {
      policy.maxAge = 300; // 5 minutes for HTML
      policy.mustRevalidate = true;
    } else if (resourceType === 'api') {
      policy.maxAge = 0; // No caching for API responses by default
      policy.mustRevalidate = true;
    }

    return policy;
  }

  // Generate cache control header
  generateCacheControlHeader(policy: CacheSecurityPolicy): string {
    const directives: string[] = [];

    if (policy.public) directives.push('public');
    if (policy.private) directives.push('private');
    if (policy.noCache) directives.push('no-cache');
    if (policy.noStore) directives.push('no-store');
    if (policy.mustRevalidate) directives.push('must-revalidate');
    if (policy.immutable) directives.push('immutable');

    if (policy.maxAge > 0) directives.push(`max-age=${policy.maxAge}`);
    if (policy.sMaxAge > 0) directives.push(`s-maxage=${policy.sMaxAge}`);
    if (policy.staleWhileRevalidate > 0) {
      directives.push(`stale-while-revalidate=${policy.staleWhileRevalidate}`);
    }
    if (policy.staleIfError > 0) {
      directives.push(`stale-if-error=${policy.staleIfError}`);
    }

    return directives.join(', ');
  }

  // Secure cache storage with encryption
  async securelyCache(
    cacheName: string,
    request: Request,
    response: Response,
    encrypt = false
  ): Promise<void> {
    if (!this.cacheStorage) return;

    try {
      const cache = await this.cacheStorage.open(cacheName);
      
      if (encrypt) {
        // Encrypt response body for sensitive data
        const responseClone = response.clone();
        const body = await responseClone.text();
        const encryptedBody = this.encryption.encrypt(
          body,
          process.env.CACHE_ENCRYPTION_KEY || 'default-key'
        ).toString();

        const encryptedResponse = new Response(encryptedBody, {
          status: response.status,
          statusText: response.statusText,
          headers: {
            ...Object.fromEntries(response.headers.entries()),
            'x-encrypted': 'true',
          },
        });

        await cache.put(request, encryptedResponse);
      } else {
        await cache.put(request, response);
      }
    } catch (error) {
      console.error('Failed to cache response:', error);
    }
  }

  // Retrieve and decrypt cached response
  async getFromSecureCache(
    cacheName: string,
    request: Request
  ): Promise<Response | undefined> {
    if (!this.cacheStorage) return undefined;

    try {
      const cache = await this.cacheStorage.open(cacheName);
      const cachedResponse = await cache.match(request);

      if (!cachedResponse) return undefined;

      // Check if response is encrypted
      if (cachedResponse.headers.get('x-encrypted') === 'true') {
        const encryptedBody = await cachedResponse.text();
        const decryptedBody = this.encryption.decrypt(
          encryptedBody,
          process.env.CACHE_ENCRYPTION_KEY || 'default-key'
        ).toString();

        return new Response(decryptedBody, {
          status: cachedResponse.status,
          statusText: cachedResponse.statusText,
          headers: cachedResponse.headers,
        });
      }

      return cachedResponse;
    } catch (error) {
      console.error('Failed to retrieve from cache:', error);
      return undefined;
    }
  }

  // Cache invalidation with security considerations
  async invalidateCache(
    pattern: string | RegExp,
    reason: string = 'security_update'
  ): Promise<void> {
    if (!this.cacheStorage) return;

    try {
      const cacheNames = await this.cacheStorage.keys();
      
      for (const cacheName of cacheNames) {
        const cache = await this.cacheStorage.open(cacheName);
        const requests = await cache.keys();

        for (const request of requests) {
          const url = request.url;
          
          if (
            (typeof pattern === 'string' && url.includes(pattern)) ||
            (pattern instanceof RegExp && pattern.test(url))
          ) {
            await cache.delete(request);
            console.log(`Cache invalidated for ${url} (reason: ${reason})`);
          }
        }
      }
    } catch (error) {
      console.error('Failed to invalidate cache:', error);
    }
  }

  // Secure cache cleanup
  async cleanupExpiredCache(): Promise<void> {
    if (!this.cacheStorage) return;

    const currentTime = Date.now();
    const cacheNames = await this.cacheStorage.keys();

    for (const cacheName of cacheNames) {
      try {
        const cache = await this.cacheStorage.open(cacheName);
        const requests = await cache.keys();

        for (const request of requests) {
          const response = await cache.match(request);
          
          if (response) {
            const cacheDate = response.headers.get('date');
            const maxAge = this.extractMaxAge(response.headers.get('cache-control') || '');
            
            if (cacheDate && maxAge > 0) {
              const expiryTime = new Date(cacheDate).getTime() + (maxAge * 1000);
              
              if (currentTime > expiryTime) {
                await cache.delete(request);
                console.log(`Expired cache entry removed: ${request.url}`);
              }
            }
          }
        }
      } catch (error) {
        console.error(`Failed to cleanup cache ${cacheName}:`, error);
      }
    }
  }

  private extractMaxAge(cacheControl: string): number {
    const maxAgeMatch = cacheControl.match(/max-age=(\d+)/);
    return maxAgeMatch ? parseInt(maxAgeMatch[1], 10) : 0;
  }
}

// Service Worker for secure caching
// sw.js content for secure caching
const CACHE_VERSION = 'v1';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;
const API_CACHE = `api-${CACHE_VERSION}`;

// Cache security policies
const CACHE_POLICIES = {
  static: {
    maxAge: 31536000, // 1 year
    strategy: 'cacheFirst',
    allowedOrigins: ['https://cdn.example.com'],
  },
  dynamic: {
    maxAge: 3600, // 1 hour
    strategy: 'staleWhileRevalidate',
    allowedOrigins: ['https://example.com'],
  },
  api: {
    maxAge: 300, // 5 minutes
    strategy: 'networkFirst',
    allowedOrigins: ['https://api.example.com'],
    requireAuth: true,
  },
};

// Secure request validation
function isRequestSecure(request) {
  const url = new URL(request.url);
  
  // Only cache HTTPS requests (except localhost in development)
  if (url.protocol !== 'https:' && url.hostname !== 'localhost') {
    return false;
  }

  // Check if origin is allowed
  const policy = getCachePolicy(request);
  if (policy.allowedOrigins && !policy.allowedOrigins.includes(url.origin)) {
    return false;
  }

  // Check if authentication is required
  if (policy.requireAuth && !request.headers.get('authorization')) {
    return false;
  }

  return true;
}

function getCachePolicy(request) {
  const url = new URL(request.url);
  
  if (url.pathname.startsWith('/api/')) {
    return CACHE_POLICIES.api;
  } else if (url.pathname.match(/\.(css|js|png|jpg|jpeg|gif|svg|woff|woff2)$/)) {
    return CACHE_POLICIES.static;
  } else {
    return CACHE_POLICIES.dynamic;
  }
}

// Cache strategy implementations
async function cacheFirstStrategy(request, cacheName) {
  const cache = await caches.open(cacheName);
  const cachedResponse = await cache.match(request);
  
  if (cachedResponse) {
    // Check if cached response is still valid
    const cacheDate = new Date(cachedResponse.headers.get('date'));
    const maxAge = extractMaxAge(cachedResponse.headers.get('cache-control') || '');
    const isExpired = Date.now() - cacheDate.getTime() > maxAge * 1000;
    
    if (!isExpired) {
      return cachedResponse;
    }
  }

  try {
    const networkResponse = await fetch(request);
    
    if (networkResponse.ok) {
      // Only cache secure responses
      if (isRequestSecure(request)) {
        await cache.put(request, networkResponse.clone());
      }
    }
    
    return networkResponse;
  } catch (error) {
    // Return stale cache if network fails
    if (cachedResponse) {
      return cachedResponse;
    }
    throw error;
  }
}

async function networkFirstStrategy(request, cacheName) {
  const cache = await caches.open(cacheName);
  
  try {
    const networkResponse = await fetch(request);
    
    if (networkResponse.ok && isRequestSecure(request)) {
      await cache.put(request, networkResponse.clone());
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

async function staleWhileRevalidateStrategy(request, cacheName) {
  const cache = await caches.open(cacheName);
  const cachedResponse = await cache.match(request);
  
  // Fetch from network in background
  const networkResponsePromise = fetch(request).then(async (response) => {
    if (response.ok && isRequestSecure(request)) {
      await cache.put(request, response.clone());
    }
    return response;
  });

  // Return cached version immediately if available
  if (cachedResponse) {
    return cachedResponse;
  }
  
  // Otherwise wait for network
  return networkResponsePromise;
}

// Service Worker event handlers
self.addEventListener('fetch', (event) => {
  const request = event.request;
  
  // Only handle GET requests
  if (request.method !== 'GET') {
    return;
  }

  // Skip non-secure requests
  if (!isRequestSecure(request)) {
    return;
  }

  const policy = getCachePolicy(request);
  let cacheName;
  let strategy;

  if (request.url.includes('/api/')) {
    cacheName = API_CACHE;
    strategy = networkFirstStrategy;
  } else if (request.destination === 'document') {
    cacheName = DYNAMIC_CACHE;
    strategy = staleWhileRevalidateStrategy;
  } else {
    cacheName = STATIC_CACHE;
    strategy = cacheFirstStrategy;
  }

  event.respondWith(strategy(request, cacheName));
});

// Cache cleanup on activation
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          // Delete old cache versions
          if (!cacheName.includes(CACHE_VERSION)) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```

## Interview-Ready Summary

**Performance Security Integration requires:**

1. **Secure CDN Configuration** - Security headers, CORS policies, SRI implementation, signed URLs, WAF rules
2. **Asset Protection** - Integrity validation, secure loading, fallback mechanisms, hotlink protection
3. **Secure Caching** - Cache policies by sensitivity, encrypted storage, proper invalidation, secure service workers
4. **Performance Monitoring** - Sanitized metrics, security indicators, mixed content detection, asset validation

**Key security measures:**
- **Content Integrity** - SRI hashes, signature validation, asset verification
- **Access Control** - Signed URLs, rate limiting, geo-blocking, origin validation
- **Cache Security** - Encryption for sensitive data, proper TTL, secure invalidation
- **Monitoring** - Security-aware performance tracking, threat detection

**Common security risks in performance:** CDN hijacking, cache poisoning, asset tampering, mixed content, insecure caching of sensitive data.

**Best practices:** Implement SRI for all external assets, use signed URLs for sensitive content, encrypt cached sensitive data, monitor for security anomalies, validate all performance metrics, secure service worker implementations.