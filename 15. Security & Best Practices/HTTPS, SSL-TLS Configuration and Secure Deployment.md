# HTTPS, SSL/TLS Configuration and Secure Deployment

This guide covers essential security practices for deploying frontend applications with proper HTTPS configuration, SSL/TLS setup, and secure deployment strategies for production environments.

## HTTPS and SSL/TLS Fundamentals

### 1. Understanding SSL/TLS
```typescript
// SSL/TLS Configuration for Node.js/Express applications
import https from 'https';
import fs from 'fs';
import express from 'express';

// Basic HTTPS server setup
const app = express();

// SSL/TLS certificate configuration
const sslOptions = {
  key: fs.readFileSync(process.env.SSL_PRIVATE_KEY_PATH!),
  cert: fs.readFileSync(process.env.SSL_CERTIFICATE_PATH!),
  ca: fs.readFileSync(process.env.SSL_CA_BUNDLE_PATH!), // Certificate Authority bundle
  
  // Security options
  secureProtocol: 'TLSv1_2_method', // Use TLS 1.2 or higher
  ciphers: [
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES128-SHA256',
    'ECDHE-RSA-AES256-SHA384'
  ].join(':'),
  honorCipherOrder: true,
  secureOptions: crypto.constants.SSL_OP_NO_SSLv2 | 
                 crypto.constants.SSL_OP_NO_SSLv3 |
                 crypto.constants.SSL_OP_NO_TLSv1 |
                 crypto.constants.SSL_OP_NO_TLSv1_1,
};

// Create HTTPS server
const httpsServer = https.createServer(sslOptions, app);

// Redirect HTTP to HTTPS
const httpApp = express();
httpApp.use((req, res) => {
  if (req.header('x-forwarded-proto') !== 'https') {
    res.redirect(301, `https://${req.header('host')}${req.url}`);
  } else {
    res.next();
  }
});

// Start servers
const HTTP_PORT = process.env.HTTP_PORT || 80;
const HTTPS_PORT = process.env.HTTPS_PORT || 443;

httpApp.listen(HTTP_PORT, () => {
  console.log(`HTTP Server running on port ${HTTP_PORT}`);
});

httpsServer.listen(HTTPS_PORT, () => {
  console.log(`HTTPS Server running on port ${HTTPS_PORT}`);
});

// HSTS (HTTP Strict Transport Security) middleware
app.use((req, res, next) => {
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );
  next();
});

// Certificate validation utility
class SSLCertificateValidator {
  static async validateCertificate(hostname: string): Promise<{
    valid: boolean;
    expiryDate: Date;
    issuer: string;
    daysUntilExpiry: number;
  }> {
    return new Promise((resolve, reject) => {
      const options = {
        hostname,
        port: 443,
        method: 'GET',
        agent: false,
        rejectUnauthorized: true,
      };

      const req = https.request(options, (res) => {
        const cert = res.connection.getPeerCertificate();
        
        if (!cert || Object.keys(cert).length === 0) {
          reject(new Error('No certificate found'));
          return;
        }

        const expiryDate = new Date(cert.validTo);
        const now = new Date();
        const daysUntilExpiry = Math.floor(
          (expiryDate.getTime() - now.getTime()) / (1000 * 60 * 60 * 24)
        );

        resolve({
          valid: cert.valid,
          expiryDate,
          issuer: cert.issuer.CN || cert.issuer.O,
          daysUntilExpiry,
        });
      });

      req.on('error', reject);
      req.end();
    });
  }

  static async checkCertificateExpiry(domains: string[]): Promise<void> {
    for (const domain of domains) {
      try {
        const result = await this.validateCertificate(domain);
        
        if (result.daysUntilExpiry < 30) {
          console.warn(
            `⚠️  Certificate for ${domain} expires in ${result.daysUntilExpiry} days`
          );
        } else {
          console.log(
            `✅ Certificate for ${domain} is valid (expires in ${result.daysUntilExpiry} days)`
          );
        }
      } catch (error) {
        console.error(`❌ Failed to validate certificate for ${domain}:`, error.message);
      }
    }
  }
}

// Automated certificate renewal with Let's Encrypt
import acme from 'acme-client';

class LetsEncryptManager {
  private client: acme.Client;

  constructor() {
    this.client = new acme.Client({
      directoryUrl: acme.directory.letsencrypt.production,
      accountKey: this.getOrCreateAccountKey(),
    });
  }

  async createCertificate(domains: string[], email: string): Promise<{
    cert: string;
    key: string;
    chain: string;
  }> {
    // Create certificate signing request
    const [key, csr] = await acme.crypto.createCsr({
      commonName: domains[0],
      altNames: domains.slice(1),
    });

    // Request certificate
    const cert = await this.client.auto({
      csr,
      email,
      termsOfServiceAgreed: true,
      challengeCreateFn: this.createChallenge.bind(this),
      challengeRemoveFn: this.removeChallenge.bind(this),
    });

    return {
      cert: cert.toString(),
      key: key.toString(),
      chain: cert.toString(), // Chain is included in cert for Let's Encrypt
    };
  }

  async renewCertificate(domains: string[], email: string): Promise<void> {
    try {
      const { cert, key, chain } = await this.createCertificate(domains, email);
      
      // Save new certificates
      await fs.promises.writeFile(process.env.SSL_CERTIFICATE_PATH!, cert);
      await fs.promises.writeFile(process.env.SSL_PRIVATE_KEY_PATH!, key);
      await fs.promises.writeFile(process.env.SSL_CA_BUNDLE_PATH!, chain);
      
      console.log('✅ Certificate renewed successfully');
      
      // Restart server to use new certificates
      process.kill(process.pid, 'SIGUSR2');
    } catch (error) {
      console.error('❌ Certificate renewal failed:', error);
      throw error;
    }
  }

  private async createChallenge(authz: any, challenge: any, keyAuthorization: string): Promise<void> {
    // HTTP-01 challenge implementation
    if (challenge.type === 'http-01') {
      const challengePath = `/.well-known/acme-challenge/${challenge.token}`;
      
      // Serve challenge response
      app.get(challengePath, (req, res) => {
        res.send(keyAuthorization);
      });
    }
  }

  private async removeChallenge(authz: any, challenge: any, keyAuthorization: string): Promise<void> {
    // Clean up challenge
    console.log('Challenge completed and cleaned up');
  }

  private getOrCreateAccountKey(): string {
    const keyPath = process.env.ACME_ACCOUNT_KEY_PATH!;
    
    try {
      return fs.readFileSync(keyPath, 'utf8');
    } catch {
      // Generate new account key
      const accountKey = acme.crypto.createPrivateKey();
      fs.writeFileSync(keyPath, accountKey);
      return accountKey;
    }
  }
}

// Automated certificate monitoring and renewal
class CertificateMonitor {
  private letsEncrypt = new LetsEncryptManager();
  private domains: string[];
  private email: string;

  constructor(domains: string[], email: string) {
    this.domains = domains;
    this.email = email;
  }

  startMonitoring(): void {
    // Check certificates daily
    setInterval(async () => {
      await this.checkAndRenewCertificates();
    }, 24 * 60 * 60 * 1000);

    // Initial check
    this.checkAndRenewCertificates();
  }

  private async checkAndRenewCertificates(): Promise<void> {
    try {
      const result = await SSLCertificateValidator.validateCertificate(this.domains[0]);
      
      // Renew if certificate expires in 30 days or less
      if (result.daysUntilExpiry <= 30) {
        console.log('🔄 Certificate renewal needed, starting process...');
        await this.letsEncrypt.renewCertificate(this.domains, this.email);
      } else {
        console.log(`✅ Certificate valid for ${result.daysUntilExpiry} more days`);
      }
    } catch (error) {
      console.error('❌ Certificate check failed:', error);
      
      // Send alert (implement your alerting mechanism)
      this.sendAlert(`Certificate check failed: ${error.message}`);
    }
  }

  private sendAlert(message: string): void {
    // Implement alerting (email, Slack, etc.)
    console.error('🚨 ALERT:', message);
  }
}
```

### 2. Frontend HTTPS Enforcement
```typescript
// Client-side HTTPS enforcement utilities
class HTTPSEnforcer {
  static enforceHTTPS(): void {
    // Redirect to HTTPS if on HTTP
    if (location.protocol !== 'https:' && process.env.NODE_ENV === 'production') {
      location.replace(`https:${location.href.substring(location.protocol.length)}`);
    }
  }

  static checkSecureContext(): boolean {
    // Check if running in secure context
    return window.isSecureContext;
  }

  static validateSecureAPIs(): void {
    const requiredSecureAPIs = [
      'serviceWorker',
      'pushManager',
      'geolocation',
      'getUserMedia'
    ];

    requiredSecureAPIs.forEach(api => {
      if (api in navigator && !window.isSecureContext) {
        console.warn(`${api} requires HTTPS to function properly`);
      }
    });
  }

  static setupSecureHeaders(): void {
    // Force HTTPS for all requests
    const originalFetch = window.fetch;
    
    window.fetch = (input: RequestInfo | URL, init?: RequestInit): Promise<Response> => {
      let url: string;
      
      if (typeof input === 'string') {
        url = input;
      } else if (input instanceof URL) {
        url = input.toString();
      } else {
        url = input.url;
      }

      // Convert HTTP URLs to HTTPS
      if (url.startsWith('http://') && process.env.NODE_ENV === 'production') {
        url = url.replace('http://', 'https://');
      }

      // Update input if it was a string
      if (typeof input === 'string') {
        input = url;
      }

      return originalFetch(input, init);
    };
  }
}

// Mixed content detection and prevention
class MixedContentPreventer {
  static scanForMixedContent(): void {
    const elements = [
      { selector: 'img[src^="http:"]', type: 'image' },
      { selector: 'script[src^="http:"]', type: 'script' },
      { selector: 'link[href^="http:"]', type: 'stylesheet' },
      { selector: 'iframe[src^="http:"]', type: 'iframe' },
      { selector: 'video[src^="http:"]', type: 'video' },
      { selector: 'audio[src^="http:"]', type: 'audio' },
    ];

    elements.forEach(({ selector, type }) => {
      const insecureElements = document.querySelectorAll(selector);
      
      if (insecureElements.length > 0) {
        console.warn(`Found ${insecureElements.length} insecure ${type} elements:`, insecureElements);
        
        // Auto-fix by upgrading to HTTPS
        insecureElements.forEach((element: any) => {
          const attr = element.tagName === 'LINK' ? 'href' : 'src';
          const originalUrl = element.getAttribute(attr);
          const secureUrl = originalUrl.replace('http://', 'https://');
          
          element.setAttribute(attr, secureUrl);
          console.log(`Upgraded ${type} from ${originalUrl} to ${secureUrl}`);
        });
      }
    });
  }

  static preventMixedContent(): void {
    // Monitor for dynamically added insecure content
    const observer = new MutationObserver((mutations) => {
      mutations.forEach((mutation) => {
        if (mutation.type === 'childList') {
          mutation.addedNodes.forEach((node) => {
            if (node.nodeType === Node.ELEMENT_NODE) {
              this.checkElementForMixedContent(node as Element);
            }
          });
        }
      });
    });

    observer.observe(document.body, {
      childList: true,
      subtree: true,
    });
  }

  private static checkElementForMixedContent(element: Element): void {
    const insecureAttributes = ['src', 'href', 'action'];
    
    insecureAttributes.forEach(attr => {
      const value = element.getAttribute(attr);
      if (value && value.startsWith('http://')) {
        const secureValue = value.replace('http://', 'https://');
        element.setAttribute(attr, secureValue);
        console.warn(`Auto-upgraded insecure ${attr}: ${value} → ${secureValue}`);
      }
    });
  }
}

// Service Worker for HTTPS upgrade
// sw.js
const CACHE_NAME = 'https-upgrade-v1';

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  
  // Upgrade HTTP requests to HTTPS
  if (url.protocol === 'http:' && url.hostname !== 'localhost') {
    url.protocol = 'https:';
    
    const upgradedRequest = new Request(url.toString(), {
      method: event.request.method,
      headers: event.request.headers,
      body: event.request.body,
      mode: 'cors',
      credentials: event.request.credentials,
      cache: event.request.cache,
      redirect: event.request.redirect,
      referrer: event.request.referrer,
    });

    event.respondWith(fetch(upgradedRequest));
  }
});

// React component for HTTPS status indicator
import { useState, useEffect } from 'react';

const HTTPSIndicator: React.FC = () => {
  const [isSecure, setIsSecure] = useState(false);
  const [certificateInfo, setCertificateInfo] = useState<any>(null);

  useEffect(() => {
    setIsSecure(window.isSecureContext);
    
    // Get certificate information if available
    if (window.isSecureContext && 'serviceWorker' in navigator) {
      // Certificate info would typically come from backend API
      fetch('/api/certificate-info')
        .then(res => res.json())
        .then(setCertificateInfo)
        .catch(console.error);
    }
  }, []);

  if (!isSecure) {
    return (
      <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded">
        <div className="flex items-center">
          <div className="w-4 h-4 mr-2">⚠️</div>
          <div>
            <strong>Insecure Connection</strong>
            <p className="text-sm">This site is not served over HTTPS. Your connection is not secure.</p>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded">
      <div className="flex items-center">
        <div className="w-4 h-4 mr-2">🔒</div>
        <div>
          <strong>Secure Connection</strong>
          {certificateInfo && (
            <p className="text-sm">
              Certificate issued by {certificateInfo.issuer} 
              (expires {new Date(certificateInfo.expiryDate).toLocaleDateString()})
            </p>
          )}
        </div>
      </div>
    </div>
  );
};

// HTTPS configuration for different frameworks
export const httpsConfigurations = {
  // Next.js configuration
  nextjs: {
    // next.config.js
    async redirects() {
      return [
        {
          source: '/(.*)',
          has: [
            {
              type: 'header',
              key: 'x-forwarded-proto',
              value: 'http',
            },
          ],
          destination: 'https://${host}${pathname}${query}',
          permanent: true,
        },
      ];
    },
    
    async headers() {
      return [
        {
          source: '/(.*)',
          headers: [
            {
              key: 'Strict-Transport-Security',
              value: 'max-age=63072000; includeSubDomains; preload',
            },
            {
              key: 'X-Frame-Options',
              value: 'DENY',
            },
            {
              key: 'X-Content-Type-Options',
              value: 'nosniff',
            },
          ],
        },
      ];
    },
  },

  // Vite configuration
  vite: {
    // vite.config.ts
    server: {
      https: {
        key: fs.readFileSync('path/to/private-key.pem'),
        cert: fs.readFileSync('path/to/certificate.pem'),
      },
      proxy: {
        '/api': {
          target: 'https://api.example.com',
          changeOrigin: true,
          secure: true,
        },
      },
    },
  },

  // Create React App (development)
  cra: {
    // .env
    HTTPS: true,
    SSL_CRT_FILE: 'path/to/certificate.crt',
    SSL_KEY_FILE: 'path/to/private-key.key',
  },
};
```

## Secure Deployment Strategies

### 1. Environment Security Configuration
```typescript
// Environment-specific security configurations
interface SecurityConfig {
  environment: 'development' | 'staging' | 'production';
  httpsOnly: boolean;
  strictTransportSecurity: boolean;
  contentSecurityPolicy: string;
  corsOrigins: string[];
  cookieSecure: boolean;
  sessionTimeout: number;
  rateLimiting: {
    windowMs: number;
    max: number;
  };
}

const securityConfigs: Record<string, SecurityConfig> = {
  development: {
    environment: 'development',
    httpsOnly: false,
    strictTransportSecurity: false,
    contentSecurityPolicy: "default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src * data: blob:;",
    corsOrigins: ['http://localhost:3000', 'http://localhost:3001'],
    cookieSecure: false,
    sessionTimeout: 60 * 60 * 1000, // 1 hour
    rateLimiting: {
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 1000, // High limit for development
    },
  },

  staging: {
    environment: 'staging',
    httpsOnly: true,
    strictTransportSecurity: true,
    contentSecurityPolicy: "default-src 'self'; script-src 'self' 'unsafe-eval'; style-src 'self' 'unsafe-inline';",
    corsOrigins: ['https://staging.example.com'],
    cookieSecure: true,
    sessionTimeout: 30 * 60 * 1000, // 30 minutes
    rateLimiting: {
      windowMs: 15 * 60 * 1000,
      max: 500,
    },
  },

  production: {
    environment: 'production',
    httpsOnly: true,
    strictTransportSecurity: true,
    contentSecurityPolicy: "default-src 'self'; script-src 'self'; style-src 'self' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com;",
    corsOrigins: ['https://example.com', 'https://www.example.com'],
    cookieSecure: true,
    sessionTimeout: 15 * 60 * 1000, // 15 minutes
    rateLimiting: {
      windowMs: 15 * 60 * 1000,
      max: 100,
    },
  },
};

// Security configuration manager
class SecurityConfigManager {
  private config: SecurityConfig;

  constructor() {
    const env = process.env.NODE_ENV as keyof typeof securityConfigs || 'development';
    this.config = securityConfigs[env];
  }

  getConfig(): SecurityConfig {
    return this.config;
  }

  applySecurityHeaders(app: any): void {
    const config = this.getConfig();

    // HTTPS enforcement
    if (config.httpsOnly) {
      app.use((req: any, res: any, next: any) => {
        if (req.header('x-forwarded-proto') !== 'https') {
          res.redirect(301, `https://${req.header('host')}${req.url}`);
        } else {
          next();
        }
      });
    }

    // HSTS
    if (config.strictTransportSecurity) {
      app.use((req: any, res: any, next: any) => {
        res.setHeader(
          'Strict-Transport-Security',
          'max-age=31536000; includeSubDomains; preload'
        );
        next();
      });
    }

    // CSP
    app.use((req: any, res: any, next: any) => {
      res.setHeader('Content-Security-Policy', config.contentSecurityPolicy);
      next();
    });

    // CORS
    app.use((req: any, res: any, next: any) => {
      const origin = req.headers.origin;
      if (config.corsOrigins.includes(origin)) {
        res.setHeader('Access-Control-Allow-Origin', origin);
      }
      res.setHeader('Access-Control-Allow-Credentials', 'true');
      next();
    });
  }

  getCookieOptions(): any {
    return {
      secure: this.config.cookieSecure,
      httpOnly: true,
      sameSite: 'strict',
      maxAge: this.config.sessionTimeout,
    };
  }
}

// Infrastructure as Code for secure deployment
export const deploymentTemplates = {
  // Docker configuration for secure deployment
  dockerfile: `
FROM node:18-alpine AS builder

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package*.json ./
RUN npm ci --only=production

# Bundle app source
COPY . .

# Build the application
RUN npm run build

# Production image
FROM node:18-alpine AS production

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Set working directory
WORKDIR /usr/src/app

# Copy built application
COPY --from=builder --chown=nextjs:nodejs /usr/src/app/.next ./.next
COPY --from=builder /usr/src/app/node_modules ./node_modules
COPY --from=builder /usr/src/app/package.json ./package.json

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/api/health || exit 1

# Start the application
CMD ["npm", "start"]
`,

  // Kubernetes deployment with security contexts
  kubernetesDeployment: `
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  labels:
    app: frontend-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
      containers:
      - name: frontend-app
        image: your-registry/frontend-app:latest
        ports:
        - containerPort: 3000
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "250m"
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /usr/src/app/.next/cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
`,

  // Nginx configuration for secure proxying
  nginxConfig: `
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL Configuration
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private-key.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-eval'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com;" always;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private must-revalidate auth;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;

    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=static:10m rate=50r/s;

    # API Routes
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend-service:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Static Assets
    location /static/ {
        limit_req zone=static burst=100 nodelay;
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri $uri/ =404;
    }

    # Frontend Application
    location / {
        proxy_pass http://frontend-service:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Security.txt
    location /.well-known/security.txt {
        return 200 "Contact: security@example.com\\nExpires: 2024-12-31T23:59:59.000Z\\nPreferred-Languages: en\\n";
        add_header Content-Type text/plain;
    }
}
`,
};
```

### 2. Automated Security Deployment Pipeline
```typescript
// CI/CD security pipeline configuration
export const securityPipeline = {
  // GitHub Actions workflow for secure deployment
  githubActions: `
name: Secure Deployment Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run security audit
      run: npm audit --audit-level=high
    
    - name: Run SAST scan
      uses: github/codeql-action/init@v2
      with:
        languages: javascript
    
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v2
    
    - name: Run ESLint security rules
      run: npx eslint . --ext .js,.jsx,.ts,.tsx --config .eslintrc.security.js
    
    - name: Check for secrets
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: main
        head: HEAD

  build-and-test:
    runs-on: ubuntu-latest
    needs: security-scan
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm test -- --coverage
    
    - name: Build application
      run: npm run build
    
    - name: Run security tests
      run: npm run test:security

  deploy:
    runs-on: ubuntu-latest
    needs: [security-scan, build-and-test]
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: |
        docker build -t ${{ secrets.REGISTRY_URL }}/frontend-app:${{ github.sha }} .
        docker tag ${{ secrets.REGISTRY_URL }}/frontend-app:${{ github.sha }} ${{ secrets.REGISTRY_URL }}/frontend-app:latest
    
    - name: Scan Docker image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ secrets.REGISTRY_URL }}/frontend-app:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Push to registry
      run: |
        echo ${{ secrets.REGISTRY_PASSWORD }} | docker login ${{ secrets.REGISTRY_URL }} -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin
        docker push ${{ secrets.REGISTRY_URL }}/frontend-app:${{ github.sha }}
        docker push ${{ secrets.REGISTRY_URL }}/frontend-app:latest
    
    - name: Deploy to staging
      run: |
        kubectl set image deployment/frontend-app frontend-app=${{ secrets.REGISTRY_URL }}/frontend-app:${{ github.sha }} -n staging
        kubectl rollout status deployment/frontend-app -n staging --timeout=300s
    
    - name: Run security verification
      run: |
        npm run verify:security -- --environment=staging
    
    - name: Deploy to production
      if: success()
      run: |
        kubectl set image deployment/frontend-app frontend-app=${{ secrets.REGISTRY_URL }}/frontend-app:${{ github.sha }} -n production
        kubectl rollout status deployment/frontend-app -n production --timeout=300s
`,

  // Security verification script
  securityVerification: `
#!/bin/bash

ENVIRONMENT=${1:-staging}
BASE_URL=${2:-https://staging.example.com}

echo "🔍 Running security verification for $ENVIRONMENT environment..."

# Check HTTPS
echo "Checking HTTPS configuration..."
SSL_RESULT=$(curl -s -I "$BASE_URL" | grep -i "strict-transport-security")
if [ -z "$SSL_RESULT" ]; then
  echo "❌ HSTS header not found"
  exit 1
else
  echo "✅ HSTS header present"
fi

# Check security headers
echo "Checking security headers..."
HEADERS=(
  "X-Content-Type-Options: nosniff"
  "X-Frame-Options: DENY"
  "X-XSS-Protection: 1; mode=block"
  "Content-Security-Policy"
)

for header in "\${HEADERS[@]}"; do
  HEADER_RESULT=$(curl -s -I "$BASE_URL" | grep -i "$header")
  if [ -z "$HEADER_RESULT" ]; then
    echo "❌ Missing header: $header"
    exit 1
  else
    echo "✅ Header present: $header"
  fi
done

# Check for mixed content
echo "Checking for mixed content..."
MIXED_CONTENT=$(curl -s "$BASE_URL" | grep -i "http://")
if [ ! -z "$MIXED_CONTENT" ]; then
  echo "⚠️  Potential mixed content found"
  echo "$MIXED_CONTENT"
fi

# SSL Labs API check (if available)
echo "Running SSL Labs check..."
SSL_LABS_RESULT=$(curl -s "https://api.ssllabs.com/api/v3/analyze?host=${BASE_URL#https://}&publish=off&startNew=on&all=done")
GRADE=$(echo "$SSL_LABS_RESULT" | jq -r '.endpoints[0].grade // "N/A"')
echo "SSL Labs Grade: $GRADE"

if [[ "$GRADE" != "A" && "$GRADE" != "A+" ]]; then
  echo "⚠️  SSL Labs grade is below A: $GRADE"
fi

echo "✅ Security verification completed"
`,
};

// Deployment security checklist
export const deploymentSecurityChecklist = [
  {
    category: 'SSL/TLS Configuration',
    items: [
      'Valid SSL certificate installed',
      'TLS 1.2+ protocols enabled',
      'Strong cipher suites configured',
      'HSTS headers implemented',
      'Certificate auto-renewal configured',
      'Mixed content eliminated',
    ],
  },
  {
    category: 'Security Headers',
    items: [
      'Content Security Policy configured',
      'X-Frame-Options set to DENY',
      'X-Content-Type-Options set to nosniff',
      'X-XSS-Protection enabled',
      'Referrer-Policy configured',
      'Permissions-Policy configured',
    ],
  },
  {
    category: 'Infrastructure Security',
    items: [
      'Non-root user in containers',
      'Read-only root filesystem',
      'Resource limits configured',
      'Health checks implemented',
      'Secrets management configured',
      'Network policies applied',
    ],
  },
  {
    category: 'Monitoring & Logging',
    items: [
      'Security event logging',
      'Certificate expiry monitoring',
      'Performance monitoring',
      'Error tracking configured',
      'Alerting configured',
      'Audit logging enabled',
    ],
  },
  {
    category: 'Access Control',
    items: [
      'WAF (Web Application Firewall) configured',
      'Rate limiting implemented',
      'IP whitelisting (if applicable)',
      'DDoS protection enabled',
      'Admin access restricted',
      'Two-factor authentication required',
    ],
  },
];
```

## Interview-Ready Summary

**HTTPS/SSL Implementation requires:**

1. **Certificate Management** - SSL/TLS certificates, auto-renewal, Let's Encrypt integration, certificate monitoring
2. **Protocol Security** - TLS 1.2+, strong ciphers, HSTS, perfect forward secrecy
3. **Mixed Content Prevention** - HTTPS enforcement, automatic upgrades, content scanning
4. **Deployment Security** - Container security, infrastructure hardening, secure CI/CD pipelines

**Key security measures:**
- **HTTPS enforcement** - Redirect HTTP to HTTPS, secure cookies
- **Certificate automation** - Auto-renewal, monitoring, alerting
- **Security headers** - HSTS, CSP, frame options, content type
- **Infrastructure security** - Non-root containers, resource limits, secrets management

**Common SSL/TLS issues:** Certificate expiry, mixed content, weak ciphers, missing HSTS, improper certificate validation.

**Best practices:** Use modern TLS versions, implement HSTS with preload, automate certificate management, monitor certificate health, secure deployment pipelines, implement comprehensive security headers.