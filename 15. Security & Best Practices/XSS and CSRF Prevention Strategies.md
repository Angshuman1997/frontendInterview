# XSS and CSRF Prevention in Modern Frontend Applications

Cross-Site Scripting (XSS) and Cross-Site Request Forgery (CSRF) remain critical security vulnerabilities in web applications. This guide provides comprehensive prevention strategies, implementation patterns, and enterprise-grade security practices for modern frontend development.

## XSS Prevention Strategies

### 1. Content Security Policy (CSP) Implementation

```typescript
// Enterprise CSP Configuration
interface CSPConfig {
  directives: {
    defaultSrc: string[];
    scriptSrc: string[];
    styleSrc: string[];
    imgSrc: string[];
    connectSrc: string[];
    fontSrc: string[];
    objectSrc: string[];
    mediaSrc: string[];
    frameSrc: string[];
  };
  reportUri?: string;
  reportOnly?: boolean;
}

class CSPManager {
  private config: CSPConfig;

  constructor(config: CSPConfig) {
    this.config = config;
  }

  generateCSPHeader(): string {
    const directives = Object.entries(this.config.directives)
      .map(([key, values]) => {
        const directiveName = key.replace(/([A-Z])/g, '-$1').toLowerCase();
        return `${directiveName} ${values.join(' ')}`;
      })
      .join('; ');

    let csp = directives;
    
    if (this.config.reportUri) {
      csp += `; report-uri ${this.config.reportUri}`;
    }

    return csp;
  }

  // Production CSP Configuration
  static getProductionConfig(): CSPConfig {
    return {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: [
          "'self'",
          "'unsafe-inline'", // Only if absolutely necessary
          "https://cdn.trusted-domain.com",
          "'nonce-{NONCE}'" // Dynamic nonce for inline scripts
        ],
        styleSrc: [
          "'self'",
          "'unsafe-inline'", // Required for CSS-in-JS libraries
          "https://fonts.googleapis.com"
        ],
        imgSrc: [
          "'self'",
          "data:",
          "https:",
          "blob:"
        ],
        connectSrc: [
          "'self'",
          "https://api.yourapp.com",
          "https://analytics.trusted-service.com"
        ],
        fontSrc: [
          "'self'",
          "https://fonts.gstatic.com"
        ],
        objectSrc: ["'none'"],
        mediaSrc: ["'self'", "blob:"],
        frameSrc: ["'self'", "https://trusted-iframe-provider.com"]
      },
      reportUri: "/api/csp-violation-report",
      reportOnly: false
    };
  }

  // Development CSP Configuration (more permissive)
  static getDevelopmentConfig(): CSPConfig {
    return {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: [
          "'self'",
          "'unsafe-inline'",
          "'unsafe-eval'", // For development tools
          "localhost:*",
          "127.0.0.1:*"
        ],
        styleSrc: [
          "'self'",
          "'unsafe-inline'",
          "localhost:*"
        ],
        imgSrc: ["'self'", "data:", "localhost:*"],
        connectSrc: ["'self'", "localhost:*", "ws:", "wss:"],
        fontSrc: ["'self'", "localhost:*"],
        objectSrc: ["'none'"],
        mediaSrc: ["'self'", "localhost:*"],
        frameSrc: ["'self'", "localhost:*"]
      },
      reportOnly: true
    };
  }
}

// React Hook for CSP Management
import { useEffect, useMemo } from 'react';

export const useCSP = (environment: 'development' | 'production') => {
  const cspManager = useMemo(() => {
    const config = environment === 'production' 
      ? CSPManager.getProductionConfig()
      : CSPManager.getDevelopmentConfig();
    
    return new CSPManager(config);
  }, [environment]);

  useEffect(() => {
    // Set CSP meta tag (for client-side applications)
    const existingMeta = document.querySelector('meta[http-equiv="Content-Security-Policy"]');
    if (existingMeta) {
      existingMeta.remove();
    }

    const meta = document.createElement('meta');
    meta.httpEquiv = 'Content-Security-Policy';
    meta.content = cspManager.generateCSPHeader();
    document.head.appendChild(meta);

    return () => {
      document.head.removeChild(meta);
    };
  }, [cspManager]);

  return cspManager;
};
```

### 2. Input Sanitization and Output Encoding

```typescript
// Comprehensive Input Sanitization Utility
import DOMPurify from 'dompurify';

interface SanitizationOptions {
  allowedTags?: string[];
  allowedAttributes?: Record<string, string[]>;
  stripTags?: boolean;
  maxLength?: number;
}

class InputSanitizer {
  private static readonly DEFAULT_ALLOWED_TAGS = [
    'p', 'br', 'strong', 'em', 'u', 'ol', 'ul', 'li', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6'
  ];

  private static readonly DEFAULT_ALLOWED_ATTRIBUTES = {
    '*': ['class'],
    'a': ['href', 'title', 'target'],
    'img': ['src', 'alt', 'width', 'height']
  };

  // HTML Content Sanitization
  static sanitizeHTML(
    input: string, 
    options: SanitizationOptions = {}
  ): string {
    if (!input) return '';

    const config = {
      ALLOWED_TAGS: options.allowedTags || this.DEFAULT_ALLOWED_TAGS,
      ALLOWED_ATTR: options.allowedAttributes || this.DEFAULT_ALLOWED_ATTRIBUTES,
      KEEP_CONTENT: !options.stripTags,
      RETURN_DOM: false,
      RETURN_DOM_FRAGMENT: false,
      RETURN_DOM_IMPORT: false
    };

    let sanitized = DOMPurify.sanitize(input, config);

    // Apply length restrictions
    if (options.maxLength && sanitized.length > options.maxLength) {
      sanitized = sanitized.substring(0, options.maxLength) + '...';
    }

    return sanitized;
  }

  // URL Sanitization
  static sanitizeURL(url: string): string {
    if (!url) return '';

    try {
      const parsed = new URL(url);
      
      // Allow only HTTP(S) protocols
      if (!['http:', 'https:'].includes(parsed.protocol)) {
        throw new Error('Invalid protocol');
      }

      // Remove dangerous query parameters
      const dangerousParams = ['javascript', 'vbscript', 'data'];
      dangerousParams.forEach(param => {
        parsed.searchParams.delete(param);
      });

      return parsed.toString();
    } catch {
      return '';
    }
  }

  // Text Input Sanitization
  static sanitizeText(input: string, maxLength: number = 1000): string {
    if (!input) return '';

    return input
      .trim()
      .replace(/[<>&"']/g, (char) => {
        const entities: Record<string, string> = {
          '<': '&lt;',
          '>': '&gt;',
          '&': '&amp;',
          '"': '&quot;',
          "'": '&#x27;'
        };
        return entities[char] || char;
      })
      .substring(0, maxLength);
  }

  // SQL Injection Prevention (for dynamic queries)
  static sanitizeSQLInput(input: string): string {
    if (!input) return '';

    return input.replace(/['";\\]/g, '\\$&');
  }

  // File Name Sanitization
  static sanitizeFileName(fileName: string): string {
    if (!fileName) return '';

    return fileName
      .replace(/[^a-zA-Z0-9._-]/g, '_')
      .replace(/_{2,}/g, '_')
      .substring(0, 255);
  }
}

// React Hook for Input Sanitization
export const useSanitizedInput = (
  initialValue: string = '',
  options: SanitizationOptions = {}
) => {
  const [value, setValue] = useState(initialValue);
  const [sanitizedValue, setSanitizedValue] = useState('');

  useEffect(() => {
    const sanitized = InputSanitizer.sanitizeHTML(value, options);
    setSanitizedValue(sanitized);
  }, [value, options]);

  const handleChange = useCallback((newValue: string) => {
    setValue(newValue);
  }, []);

  return {
    value,
    sanitizedValue,
    onChange: handleChange,
    isValid: value === sanitizedValue
  };
};
```

## CSRF Prevention Implementation

### 1. CSRF Token Management

```typescript
// CSRF Token Management System
interface CSRFConfig {
  tokenName: string;
  headerName: string;
  cookieName: string;
  expiration: number; // milliseconds
  domain?: string;
  secure: boolean;
  sameSite: 'strict' | 'lax' | 'none';
}

class CSRFTokenManager {
  private config: CSRFConfig;
  private currentToken: string | null = null;

  constructor(config: CSRFConfig) {
    this.config = config;
  }

  // Generate cryptographically secure token
  private generateToken(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('');
  }

  // Get or create CSRF token
  async getToken(): Promise<string> {
    if (this.currentToken && this.isTokenValid()) {
      return this.currentToken;
    }

    try {
      // Try to get token from server
      const response = await fetch('/api/csrf-token', {
        method: 'GET',
        credentials: 'include'
      });

      if (response.ok) {
        const data = await response.json();
        this.currentToken = data.token;
        this.storeToken(this.currentToken);
        return this.currentToken;
      }
    } catch (error) {
      console.warn('Failed to fetch CSRF token from server:', error);
    }

    // Fallback: generate client-side token
    this.currentToken = this.generateToken();
    this.storeToken(this.currentToken);
    return this.currentToken;
  }

  // Store token in session storage with expiration
  private storeToken(token: string): void {
    const tokenData = {
      token,
      expiration: Date.now() + this.config.expiration
    };

    sessionStorage.setItem(this.config.tokenName, JSON.stringify(tokenData));
  }

  // Check if current token is valid
  private isTokenValid(): boolean {
    if (!this.currentToken) return false;

    try {
      const stored = sessionStorage.getItem(this.config.tokenName);
      if (!stored) return false;

      const tokenData = JSON.parse(stored);
      return Date.now() < tokenData.expiration && tokenData.token === this.currentToken;
    } catch {
      return false;
    }
  }

  // Clear token
  clearToken(): void {
    this.currentToken = null;
    sessionStorage.removeItem(this.config.tokenName);
  }

  // Get token for request headers
  getHeaderValue(): string {
    return this.currentToken || '';
  }
}

// CSRF-Protected HTTP Client
class CSRFProtectedClient {
  private csrfManager: CSRFTokenManager;
  private baseURL: string;

  constructor(baseURL: string, csrfConfig: CSRFConfig) {
    this.baseURL = baseURL;
    this.csrfManager = new CSRFTokenManager(csrfConfig);
  }

  // Protected fetch wrapper
  async fetch(endpoint: string, options: RequestInit = {}): Promise<Response> {
    const url = `${this.baseURL}${endpoint}`;
    
    // Get CSRF token for state-changing operations
    const needsCSRF = ['POST', 'PUT', 'PATCH', 'DELETE'].includes(
      options.method?.toUpperCase() || 'GET'
    );

    if (needsCSRF) {
      const token = await this.csrfManager.getToken();
      
      options.headers = {
        ...options.headers,
        [this.csrfManager.config.headerName]: token,
        'Content-Type': 'application/json'
      };
    }

    // Include credentials for cookie-based authentication
    options.credentials = 'include';

    try {
      const response = await fetch(url, options);

      // Handle CSRF token expiration
      if (response.status === 403) {
        const errorText = await response.text();
        if (errorText.includes('CSRF')) {
          this.csrfManager.clearToken();
          
          // Retry once with new token
          if (needsCSRF) {
            const newToken = await this.csrfManager.getToken();
            options.headers = {
              ...options.headers,
              [this.csrfManager.config.headerName]: newToken
            };
            
            return fetch(url, options);
          }
        }
      }

      return response;
    } catch (error) {
      console.error('CSRF-protected request failed:', error);
      throw error;
    }
  }

  // Convenience methods
  async get(endpoint: string): Promise<Response> {
    return this.fetch(endpoint, { method: 'GET' });
  }

  async post(endpoint: string, data: any): Promise<Response> {
    return this.fetch(endpoint, {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  async put(endpoint: string, data: any): Promise<Response> {
    return this.fetch(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }

  async delete(endpoint: string): Promise<Response> {
    return this.fetch(endpoint, { method: 'DELETE' });
  }
}

// React Hook for CSRF Protection
export const useCSRFProtection = () => {
  const [client] = useState(() => {
    const config: CSRFConfig = {
      tokenName: 'csrf_token',
      headerName: 'X-CSRF-Token',
      cookieName: 'csrf_token',
      expiration: 30 * 60 * 1000, // 30 minutes
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict'
    };

    return new CSRFProtectedClient('/api', config);
  });

  return {
    client,
    get: client.get.bind(client),
    post: client.post.bind(client),
    put: client.put.bind(client),
    delete: client.delete.bind(client)
  };
};
```

### 2. SameSite Cookie Configuration

```typescript
// Secure Cookie Management
interface CookieOptions {
  maxAge?: number;
  expires?: Date;
  path?: string;
  domain?: string;
  secure?: boolean;
  httpOnly?: boolean;
  sameSite?: 'strict' | 'lax' | 'none';
}

class SecureCookieManager {
  private static readonly DEFAULT_OPTIONS: CookieOptions = {
    secure: true,
    httpOnly: true,
    sameSite: 'strict',
    path: '/'
  };

  // Set secure cookie
  static setCookie(
    name: string, 
    value: string, 
    options: CookieOptions = {}
  ): void {
    const opts = { ...this.DEFAULT_OPTIONS, ...options };
    
    let cookieString = `${encodeURIComponent(name)}=${encodeURIComponent(value)}`;

    if (opts.maxAge) {
      cookieString += `; Max-Age=${opts.maxAge}`;
    }

    if (opts.expires) {
      cookieString += `; Expires=${opts.expires.toUTCString()}`;
    }

    if (opts.path) {
      cookieString += `; Path=${opts.path}`;
    }

    if (opts.domain) {
      cookieString += `; Domain=${opts.domain}`;
    }

    if (opts.secure) {
      cookieString += '; Secure';
    }

    if (opts.httpOnly) {
      cookieString += '; HttpOnly';
    }

    if (opts.sameSite) {
      cookieString += `; SameSite=${opts.sameSite}`;
    }

    document.cookie = cookieString;
  }

  // Get cookie value
  static getCookie(name: string): string | null {
    const nameEQ = encodeURIComponent(name) + '=';
    const cookies = document.cookie.split(';');

    for (let cookie of cookies) {
      cookie = cookie.trim();
      if (cookie.indexOf(nameEQ) === 0) {
        return decodeURIComponent(cookie.substring(nameEQ.length));
      }
    }

    return null;
  }

  // Delete cookie
  static deleteCookie(name: string, path: string = '/', domain?: string): void {
    this.setCookie(name, '', {
      maxAge: 0,
      path,
      domain
    });
  }

  // Session cookie for authentication
  static setAuthCookie(token: string, remember: boolean = false): void {
    const options: CookieOptions = {
      secure: process.env.NODE_ENV === 'production',
      httpOnly: true,
      sameSite: 'strict',
      path: '/'
    };

    if (remember) {
      options.maxAge = 30 * 24 * 60 * 60; // 30 days
    }

    this.setCookie('auth_token', token, options);
  }

  // CSRF cookie
  static setCSRFCookie(token: string): void {
    this.setCookie('csrf_token', token, {
      secure: process.env.NODE_ENV === 'production',
      httpOnly: false, // Must be accessible to JavaScript
      sameSite: 'strict',
      path: '/',
      maxAge: 30 * 60 // 30 minutes
    });
  }
}
```

## Security Headers and Middleware

### 1. Express.js Security Middleware

```typescript
// Comprehensive Security Middleware for Express.js
import express from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import cors from 'cors';

interface SecurityConfig {
  corsOrigins: string[];
  rateLimitWindow: number;
  rateLimitMax: number;
  csrfSecret: string;
  sessionSecret: string;
  trustProxy: boolean;
}

export const createSecurityMiddleware = (config: SecurityConfig) => {
  const app = express();

  // Trust proxy settings
  if (config.trustProxy) {
    app.set('trust proxy', 1);
  }

  // Helmet for security headers
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:", "https:"],
        connectSrc: ["'self'"],
        frameSrc: ["'none'"],
        objectSrc: ["'none'"]
      }
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true
    },
    noSniff: true,
    frameguard: { action: 'deny' },
    xssFilter: true,
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
  }));

  // CORS configuration
  app.use(cors({
    origin: config.corsOrigins,
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
    exposedHeaders: ['X-CSRF-Token']
  }));

  // Rate limiting
  const limiter = rateLimit({
    windowMs: config.rateLimitWindow,
    max: config.rateLimitMax,
    message: 'Too many requests from this IP',
    standardHeaders: true,
    legacyHeaders: false,
    handler: (req, res) => {
      res.status(429).json({
        error: 'Rate limit exceeded',
        retryAfter: Math.round(config.rateLimitWindow / 1000)
      });
    }
  });

  app.use('/api/', limiter);

  return app;
};
```

## Interview-Ready Security Summary

**XSS Prevention encompasses:**
1. **Content Security Policy** - Restricting resource loading and script execution
2. **Input Sanitization** - Cleaning user input before processing/storage
3. **Output Encoding** - Properly encoding data before rendering
4. **Template Security** - Using secure templating engines

**CSRF Prevention includes:**
1. **CSRF Tokens** - Unique tokens for state-changing operations
2. **SameSite Cookies** - Preventing cross-site cookie transmission
3. **Origin Validation** - Verifying request origins
4. **Double Submit Cookies** - Additional token validation layer

**Enterprise Security Features:**
- Comprehensive CSP management with environment-specific configs
- Automated token generation and rotation
- Rate limiting and abuse prevention
- Secure cookie handling with proper attributes
- Security header management via middleware

**Common Interview Questions:**
- How do you prevent XSS in a React application?
- Explain CSRF attacks and prevention strategies
- What security headers should every web application have?
- How do you handle authentication tokens securely?
- Describe the difference between stored and reflected XSS
