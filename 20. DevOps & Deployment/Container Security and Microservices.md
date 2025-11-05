# Container Security and Microservices Architecture

Container security in frontend applications involves securing Docker containers, Kubernetes deployments, and microservices architectures. This comprehensive guide covers container hardening, image security scanning, runtime protection, service mesh implementation, and advanced security patterns for production environments.

## Container Security Fundamentals

### Secure Docker Image Creation

```dockerfile
# Multi-stage build for minimal attack surface
# Build stage
FROM node:18-alpine AS builder

# Security: Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S frontend -u 1001

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./
COPY --chown=frontend:nodejs package*.json ./

# Install dependencies with security measures
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY --chown=frontend:nodejs . .

# Build the application
RUN npm run build

# Security: Remove development dependencies and source code
RUN rm -rf src/ node_modules/ && \
    npm ci --only=production --omit=dev

# Production stage with minimal base image
FROM nginx:1.25-alpine AS production

# Security: Install security updates
RUN apk update && \
    apk upgrade && \
    apk add --no-cache \
        dumb-init \
        curl && \
    rm -rf /var/cache/apk/*

# Security: Create non-root user for nginx
RUN addgroup -g 1001 -S nginx-group && \
    adduser -S nginx-user -u 1001 -G nginx-group

# Security: Remove default nginx configuration
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom nginx configuration
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/security-headers.conf /etc/nginx/conf.d/security-headers.conf

# Copy built application from builder stage
COPY --from=builder --chown=nginx-user:nginx-group /app/dist /usr/share/nginx/html

# Security: Set proper file permissions
RUN chown -R nginx-user:nginx-group /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html && \
    chown -R nginx-user:nginx-group /var/cache/nginx && \
    chown -R nginx-user:nginx-group /var/log/nginx && \
    chown -R nginx-user:nginx-group /etc/nginx/conf.d

# Security: Create directories with proper permissions
RUN mkdir -p /var/cache/nginx/client_temp && \
    mkdir -p /var/cache/nginx/proxy_temp && \
    mkdir -p /var/cache/nginx/fastcgi_temp && \
    mkdir -p /var/cache/nginx/uwsgi_temp && \
    mkdir -p /var/cache/nginx/scgi_temp && \
    chown -R nginx-user:nginx-group /var/cache/nginx

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Security: Use non-root user
USER nginx-user

# Security: Use dumb-init as PID 1
ENTRYPOINT ["dumb-init", "--"]

# Start nginx
CMD ["nginx", "-g", "daemon off;"]

# Security: Expose only necessary port
EXPOSE 8080

# Metadata
LABEL maintainer="frontend-team@company.com" \
      version="1.0.0" \
      description="Secure frontend application container" \
      security.scan="true"
```

### Secure Nginx Configuration

```nginx
# nginx/nginx.conf
user nginx-user;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /tmp/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    # Security: Hide nginx version
    server_tokens off;
    
    # Security: Prevent clickjacking
    add_header X-Frame-Options "DENY" always;
    
    # Security: MIME type sniffing protection
    add_header X-Content-Type-Options "nosniff" always;
    
    # Security: XSS protection
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Security: Strict Transport Security
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # Security: Content Security Policy
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https:; connect-src 'self' https://api.company.com; frame-ancestors 'none';" always;
    
    # Security: Referrer policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Security: Permissions policy
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
    
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    # Performance optimizations
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Security: Buffer overflow protection
    client_body_buffer_size 1K;
    client_header_buffer_size 1k;
    client_max_body_size 1k;
    large_client_header_buffers 2 1k;
    
    # Security: Timeout protection
    client_body_timeout 10;
    client_header_timeout 10;
    keepalive_timeout 5 5;
    send_timeout 10;
    
    # Security: Rate limiting
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 10240;
    gzip_proxied expired no-cache no-store private must-revalidate auth;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/x-javascript
        application/xml+rss
        application/javascript
        application/json;
    
    server {
        listen 8080;
        server_name _;
        root /usr/share/nginx/html;
        index index.html index.htm;
        
        # Security: Disable access to hidden files
        location ~ /\. {
            deny all;
            access_log off;
            log_not_found off;
        }
        
        # Security: Disable access to backup files
        location ~ ~$ {
            deny all;
            access_log off;
            log_not_found off;
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # API rate limiting
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        # Static assets with long cache
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            add_header X-Content-Type-Options "nosniff";
            
            # Security: Prevent direct access to source maps in production
            location ~* \.map$ {
                deny all;
                access_log off;
                log_not_found off;
            }
        }
        
        # SPA routing - serve index.html for all routes
        location / {
            try_files $uri $uri/ /index.html;
            
            # Security: No cache for HTML files
            add_header Cache-Control "no-cache, no-store, must-revalidate";
            add_header Pragma "no-cache";
            add_header Expires "0";
        }
        
        # Security: Deny access to sensitive files
        location ~* \.(env|log|htaccess|htpasswd|ini|phps|fla|psd|sh)$ {
            deny all;
            access_log off;
            log_not_found off;
        }
        
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

### Container Image Scanning and Security

```yaml
# .github/workflows/container-security.yml
name: Container Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * *'  # Daily scan at 6 AM

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: false
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          load: true

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH,MEDIUM'
          exit-code: '1'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Run Snyk container security test
        uses: snyk/actions/docker@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          args: --severity-threshold=high --file=Dockerfile

      - name: Run Docker Bench Security
        run: |
          docker run --rm --net host --pid host --userns host --cap-add audit_control \
            -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
            -v /var/lib:/var/lib:ro \
            -v /var/run/docker.sock:/var/run/docker.sock:ro \
            -v /usr/lib/systemd:/usr/lib/systemd:ro \
            -v /etc:/etc:ro \
            --label docker_bench_security \
            docker/docker-bench-security

      - name: Run Hadolint Dockerfile linter
        uses: hadolint/hadolint-action@v2.1.0
        with:
          dockerfile: Dockerfile
          format: sarif
          output-file: hadolint-results.sarif
          no-fail: true

      - name: Upload Hadolint scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'hadolint-results.sarif'

      - name: Push Docker image
        if: github.event_name != 'pull_request'
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  container-runtime-security:
    runs-on: ubuntu-latest
    needs: build-and-scan
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Kind cluster
        uses: helm/kind-action@v1.4.0
        with:
          cluster_name: security-test

      - name: Install Falco
        run: |
          helm repo add falcosecurity https://falcosecurity.github.io/charts
          helm repo update
          helm install falco falcosecurity/falco \
            --set falco.grpc.enabled=true \
            --set falco.grpcOutput.enabled=true

      - name: Deploy test application
        run: |
          kubectl apply -f k8s/namespace.yaml
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml

      - name: Run runtime security tests
        run: |
          # Wait for deployment
          kubectl wait --for=condition=available --timeout=300s deployment/frontend-app
          
          # Run security tests
          kubectl exec -it deployment/frontend-app -- /bin/sh -c "ls -la /" || echo "Directory traversal test"
          kubectl exec -it deployment/frontend-app -- /bin/sh -c "cat /etc/passwd" || echo "Sensitive file access test"
          
          # Check Falco alerts
          kubectl logs -l app.kubernetes.io/name=falco | grep "security violation" || true

      - name: Generate security report
        run: |
          echo "## Container Security Report" > security-report.md
          echo "### Vulnerability Scan Results" >> security-report.md
          cat trivy-results.sarif | jq '.runs[0].results | length' >> security-report.md
          echo "### Runtime Security Events" >> security-report.md
          kubectl logs -l app.kubernetes.io/name=falco | tail -20 >> security-report.md

      - name: Upload security report
        uses: actions/upload-artifact@v3
        with:
          name: security-report
          path: security-report.md
```

## Kubernetes Security Configuration

### Secure Pod Security Standards

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-app
  labels:
    name: frontend-app
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
  namespace: frontend-app
spec:
  podSelector:
    matchLabels:
      app: frontend-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: nginx-ingress
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend-services
    ports:
    - protocol: TCP
      port: 443
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
---
# k8s/pod-security-policy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: frontend-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  runAsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

### Secure Deployment Configuration

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: frontend-app
  labels:
    app: frontend-app
    version: v1.0.0
    tier: frontend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
        version: v1.0.0
        tier: frontend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # Security: Use service account with minimal permissions
      serviceAccountName: frontend-service-account
      
      # Security: Run as non-root user
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: frontend-app
        image: ghcr.io/company/frontend-app:latest
        imagePullPolicy: Always
        
        # Security: Container security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1001
          runAsGroup: 1001
          capabilities:
            drop:
            - ALL
          seccompProfile:
            type: RuntimeDefault
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: http
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        
        readinessProbe:
          httpGet:
            path: /health
            port: http
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
        
        startupProbe:
          httpGet:
            path: /health
            port: http
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30
          successThreshold: 1
        
        # Resource limits for security and stability
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
            ephemeral-storage: 1Gi
          requests:
            cpu: 100m
            memory: 128Mi
            ephemeral-storage: 100Mi
        
        # Environment variables
        env:
        - name: NODE_ENV
          value: "production"
        - name: API_BASE_URL
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: api-base-url
        - name: MONITORING_ENDPOINT
          valueFrom:
            secretKeyRef:
              name: frontend-secrets
              key: monitoring-endpoint
        
        # Volume mounts for writable directories
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: var-cache-nginx
          mountPath: /var/cache/nginx
        - name: var-log-nginx
          mountPath: /var/log/nginx
        - name: config-volume
          mountPath: /etc/nginx/conf.d
          readOnly: true
      
      # Volumes
      volumes:
      - name: tmp-volume
        emptyDir:
          sizeLimit: 100Mi
      - name: var-cache-nginx
        emptyDir:
          sizeLimit: 100Mi
      - name: var-log-nginx
        emptyDir:
          sizeLimit: 100Mi
      - name: config-volume
        configMap:
          name: nginx-config
      
      # Security: Use specific node selector for trusted nodes
      nodeSelector:
        node-security-group: "frontend"
      
      # Security: Pod affinity for better security isolation
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - frontend-app
              topologyKey: kubernetes.io/hostname
      
      # Security: Tolerations for dedicated security nodes
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "frontend"
        effect: "NoSchedule"
      
      # Security: DNS policy
      dnsPolicy: ClusterFirst
      dnsConfig:
        options:
        - name: ndots
          value: "2"
        - name: edns0
      
      # Security: Restart policy
      restartPolicy: Always
      
      # Security: Termination grace period
      terminationGracePeriodSeconds: 30
---
# k8s/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend-service-account
  namespace: frontend-app
automountServiceAccountToken: false
---
# k8s/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: frontend-app
  name: frontend-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-role-binding
  namespace: frontend-app
subjects:
- kind: ServiceAccount
  name: frontend-service-account
  namespace: frontend-app
roleRef:
  kind: Role
  name: frontend-role
  apiGroup: rbac.authorization.k8s.io
```

## Service Mesh Security with Istio

### Istio Security Configuration

```yaml
# istio/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-app
  labels:
    istio-injection: enabled
    name: frontend-app
---
# istio/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: frontend-app
spec:
  mtls:
    mode: STRICT
---
# istio/authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-authz
  namespace: frontend-app
spec:
  selector:
    matchLabels:
      app: frontend-app
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
  - to:
    - operation:
        methods: ["GET", "POST", "PUT", "DELETE"]
  - when:
    - key: source.ip
      notValues: ["10.0.0.0/8"]
---
# istio/destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: frontend-destination
  namespace: frontend-app
spec:
  host: frontend-app.frontend-app.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    circuitBreaker:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
---
# istio/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: frontend-vs
  namespace: frontend-app
spec:
  hosts:
  - frontend-app.frontend-app.svc.cluster.local
  http:
  - match:
    - headers:
        user-agent:
          regex: ".*bot.*|.*crawler.*"
    fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 5s
    route:
    - destination:
        host: frontend-app.frontend-app.svc.cluster.local
  - match:
    - uri:
        prefix: "/api/"
    route:
    - destination:
        host: backend-api.backend.svc.cluster.local
      weight: 100
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
  - match:
    - uri:
        prefix: "/"
    route:
    - destination:
        host: frontend-app.frontend-app.svc.cluster.local
    headers:
      response:
        add:
          X-Frame-Options: "DENY"
          X-Content-Type-Options: "nosniff"
          Strict-Transport-Security: "max-age=31536000; includeSubDomains"
---
# istio/security-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: frontend-app
spec:
  selector:
    matchLabels:
      app: frontend-app
  jwtRules:
  - issuer: "https://auth.company.com"
    jwksUri: "https://auth.company.com/.well-known/jwks.json"
    audiences:
    - "frontend-app"
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: frontend-app
spec:
  selector:
    matchLabels:
      app: frontend-app
  rules:
  - from:
    - source:
        requestPrincipals: ["https://auth.company.com/*"]
  - to:
    - operation:
        paths: ["/api/*"]
```

## Microservices Security Patterns

### API Gateway Security Implementation

```typescript
// microservices/api-gateway/security-middleware.ts
import express from 'express';
import jwt from 'jsonwebtoken';
import rateLimit from 'express-rate-limit';
import helmet from 'helmet';
import cors from 'cors';
import compression from 'compression';

interface SecurityConfig {
  jwtSecret: string;
  corsOrigins: string[];
  rateLimitWindowMs: number;
  rateLimitMax: number;
  enableCSP: boolean;
}

export class APIGatewaySecurity {
  private app: express.Application;
  private config: SecurityConfig;

  constructor(app: express.Application, config: SecurityConfig) {
    this.app = app;
    this.config = config;
    this.setupSecurityMiddleware();
  }

  private setupSecurityMiddleware(): void {
    // Helmet for security headers
    this.app.use(helmet({
      contentSecurityPolicy: this.config.enableCSP ? {
        directives: {
          defaultSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
          fontSrc: ["'self'", "https://fonts.gstatic.com"],
          imgSrc: ["'self'", "data:", "https:"],
          scriptSrc: ["'self'"],
          connectSrc: ["'self'", "https://api.company.com"],
          frameSrc: ["'none'"],
          objectSrc: ["'none'"],
          upgradeInsecureRequests: [],
        },
      } : false,
      hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true
      }
    }));

    // CORS configuration
    this.app.use(cors({
      origin: (origin, callback) => {
        if (!origin || this.config.corsOrigins.includes(origin)) {
          callback(null, true);
        } else {
          callback(new Error('Not allowed by CORS'));
        }
      },
      credentials: true,
      optionsSuccessStatus: 200
    }));

    // Compression
    this.app.use(compression());

    // Rate limiting
    const limiter = rateLimit({
      windowMs: this.config.rateLimitWindowMs,
      max: this.config.rateLimitMax,
      message: 'Too many requests from this IP',
      standardHeaders: true,
      legacyHeaders: false,
    });
    this.app.use(limiter);

    // API-specific rate limiting
    const apiLimiter = rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // limit each IP to 100 requests per windowMs
      message: 'Too many API requests from this IP',
      skip: (req) => {
        // Skip rate limiting for health checks
        return req.path === '/health';
      }
    });
    this.app.use('/api/', apiLimiter);

    // JWT Authentication middleware
    this.app.use('/api/', this.authenticateToken.bind(this));

    // Request logging
    this.app.use(this.requestLogger.bind(this));

    // Input validation
    this.app.use(this.inputValidation.bind(this));
  }

  private authenticateToken(req: express.Request, res: express.Response, next: express.NextFunction): void {
    // Skip authentication for public endpoints
    const publicPaths = ['/api/health', '/api/status', '/api/public'];
    if (publicPaths.includes(req.path)) {
      return next();
    }

    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (!token) {
      return res.status(401).json({ error: 'Access token required' });
    }

    jwt.verify(token, this.config.jwtSecret, (err: any, user: any) => {
      if (err) {
        return res.status(403).json({ error: 'Invalid or expired token' });
      }

      // Add user info to request
      (req as any).user = user;
      next();
    });
  }

  private requestLogger(req: express.Request, res: express.Response, next: express.NextFunction): void {
    const start = Date.now();
    
    res.on('finish', () => {
      const duration = Date.now() - start;
      const logData = {
        method: req.method,
        url: req.url,
        status: res.statusCode,
        duration,
        userAgent: req.get('User-Agent'),
        ip: req.ip,
        timestamp: new Date().toISOString()
      };

      // Log security events
      if (res.statusCode === 401 || res.statusCode === 403) {
        console.warn('Security event:', logData);
      } else {
        console.log('Request:', logData);
      }
    });

    next();
  }

  private inputValidation(req: express.Request, res: express.Response, next: express.NextFunction): void {
    // SQL injection protection
    const sqlInjectionPattern = /(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC|UNION|SCRIPT)\b)|(['"`;])/i;
    
    const checkForSQLInjection = (value: string): boolean => {
      return sqlInjectionPattern.test(value);
    };

    // XSS protection
    const xssPattern = /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi;
    
    const checkForXSS = (value: string): boolean => {
      return xssPattern.test(value);
    };

    // Validate query parameters
    for (const [key, value] of Object.entries(req.query)) {
      if (typeof value === 'string') {
        if (checkForSQLInjection(value) || checkForXSS(value)) {
          return res.status(400).json({ error: 'Invalid input detected' });
        }
      }
    }

    // Validate request body
    if (req.body && typeof req.body === 'object') {
      const bodyStr = JSON.stringify(req.body);
      if (checkForSQLInjection(bodyStr) || checkForXSS(bodyStr)) {
        return res.status(400).json({ error: 'Invalid input detected' });
      }
    }

    next();
  }

  public setupServiceProxy(serviceName: string, serviceUrl: string): void {
    this.app.use(`/api/${serviceName}`, this.createServiceProxy(serviceUrl));
  }

  private createServiceProxy(serviceUrl: string): express.RequestHandler {
    return async (req: express.Request, res: express.Response) => {
      try {
        const targetUrl = `${serviceUrl}${req.path}`;
        
        // Forward request to microservice
        const response = await fetch(targetUrl, {
          method: req.method,
          headers: {
            'Content-Type': 'application/json',
            'Authorization': req.headers.authorization || '',
            'X-User-ID': (req as any).user?.id || '',
            'X-Request-ID': req.headers['x-request-id'] || this.generateRequestId(),
          },
          body: req.method !== 'GET' ? JSON.stringify(req.body) : undefined,
        });

        const data = await response.json();
        res.status(response.status).json(data);
      } catch (error) {
        console.error('Service proxy error:', error);
        res.status(500).json({ error: 'Internal server error' });
      }
    };
  }

  private generateRequestId(): string {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Usage example
const app = express();
const security = new APIGatewaySecurity(app, {
  jwtSecret: process.env.JWT_SECRET!,
  corsOrigins: ['https://app.company.com', 'https://admin.company.com'],
  rateLimitWindowMs: 15 * 60 * 1000,
  rateLimitMax: 100,
  enableCSP: true
});

// Setup service proxies
security.setupServiceProxy('auth', 'http://auth-service:3001');
security.setupServiceProxy('users', 'http://user-service:3002');
security.setupServiceProxy('analytics', 'http://analytics-service:3003');

export default security;
```

## Runtime Security Monitoring

### Falco Security Rules

```yaml
# falco/falco-rules.yaml
- rule: Container Privilege Escalation
  desc: Detect attempts to escalate privileges in containers
  condition: >
    spawned_process and container and
    (proc.name in (sudo, su, doas) or
     proc.args contains "--privileged" or
     proc.args contains "setuid" or
     proc.args contains "setgid")
  output: >
    Privilege escalation attempt detected (user=%user.name command=%proc.cmdline 
    container=%container.name image=%container.image)
  priority: WARNING
  tags: [container, privilege_escalation]

- rule: Unexpected Network Activity
  desc: Detect unexpected network connections from frontend containers
  condition: >
    inbound_outbound and container and
    container.image contains "frontend" and
    not fd.sport in (80, 443, 8080, 8443) and
    not fd.dport in (80, 443, 8080, 8443, 53, 3000, 3001, 3002, 3003)
  output: >
    Unexpected network activity from frontend container (user=%user.name 
    command=%proc.cmdline connection=%fd.name container=%container.name)
  priority: WARNING
  tags: [container, network, frontend]

- rule: Sensitive File Access
  desc: Detect access to sensitive files
  condition: >
    open_read and container and
    (fd.name contains "/etc/passwd" or
     fd.name contains "/etc/shadow" or
     fd.name contains "/proc/self/environ" or
     fd.name contains "/.env" or
     fd.name contains "/run/secrets")
  output: >
    Sensitive file access detected (user=%user.name file=%fd.name 
    command=%proc.cmdline container=%container.name)
  priority: HIGH
  tags: [container, filesystem, sensitive_data]

- rule: Container Escape Attempt
  desc: Detect potential container escape attempts
  condition: >
    spawned_process and container and
    (proc.name in (nsenter, chroot, unshare) or
     proc.args contains "/proc/1/root" or
     proc.args contains "docker.sock" or
     proc.args contains "/var/run/docker.sock")
  output: >
    Container escape attempt detected (user=%user.name command=%proc.cmdline 
    container=%container.name image=%container.image)
  priority: CRITICAL
  tags: [container, escape_attempt]

- rule: Suspicious Shell Activity
  desc: Detect suspicious shell activity in frontend containers
  condition: >
    spawned_process and container and
    container.image contains "frontend" and
    shell_procs and
    (proc.args contains "curl" or
     proc.args contains "wget" or
     proc.args contains "nc" or
     proc.args contains "netcat")
  output: >
    Suspicious shell activity in frontend container (user=%user.name 
    command=%proc.cmdline container=%container.name)
  priority: WARNING
  tags: [container, shell, frontend, suspicious]
```

## Interview-Ready Container Security Summary

**Container Security Fundamentals:**
1. **Secure Image Building** - Multi-stage builds, minimal base images, non-root users, and security scanning
2. **Runtime Security** - Pod Security Standards, network policies, resource limits, and RBAC
3. **Service Mesh Security** - mTLS, authorization policies, traffic management, and observability
4. **Microservices Security** - API gateway patterns, JWT authentication, input validation, and service-to-service encryption

**Enterprise Security Patterns:**
- Defense in depth with multiple security layers and validation points
- Zero-trust architecture with explicit authentication and authorization
- Continuous security monitoring with runtime threat detection
- Compliance automation with policy-as-code and security scanning

**Production Capabilities:**
- Automated vulnerability scanning in CI/CD pipelines
- Runtime security monitoring with Falco and threat detection
- Secure service communication with Istio service mesh
- Container image signing and supply chain security

**Key Interview Topics:** Container security best practices, Kubernetes security model, service mesh architecture, microservices authentication patterns, runtime security monitoring, vulnerability management, compliance automation, zero-trust networking, container image security, and DevSecOps integration.