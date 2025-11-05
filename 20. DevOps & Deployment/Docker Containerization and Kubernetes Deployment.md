# Docker Containerization and Kubernetes Deployment

Modern application deployment requires containerization and orchestration for scalability, reliability, and consistent environments. This guide covers Docker best practices and Kubernetes deployment strategies for frontend and full-stack applications.

## Docker Implementation for Frontend Applications

### 1. Optimized Dockerfile for React/Next.js Applications
```dockerfile
# Multi-stage build for production-ready React application
# Stage 1: Build environment
FROM node:20-alpine AS builder

# Set working directory
WORKDIR /app

# Add package files
COPY package*.json ./
COPY yarn.lock* ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY . .

# Build arguments
ARG REACT_APP_API_URL
ARG REACT_APP_VERSION
ARG REACT_APP_ENVIRONMENT
ARG BUILD_TIME
ARG VCS_REF

# Set environment variables
ENV REACT_APP_API_URL=${REACT_APP_API_URL}
ENV REACT_APP_VERSION=${REACT_APP_VERSION}
ENV REACT_APP_ENVIRONMENT=${REACT_APP_ENVIRONMENT}
ENV NODE_ENV=production

# Build application
RUN npm run build

# Add build metadata
RUN echo "{\"buildTime\":\"${BUILD_TIME}\",\"version\":\"${REACT_APP_VERSION}\",\"vcsRef\":\"${VCS_REF}\"}" > build/build-info.json

# Stage 2: Production environment
FROM nginx:1.25-alpine AS production

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache \
    curl \
    ca-certificates \
    && rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Copy built application
COPY --from=builder /app/build /usr/share/nginx/html

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf /etc/nginx/conf.d/default.conf

# Add health check script
COPY healthcheck.sh /usr/local/bin/healthcheck.sh
RUN chmod +x /usr/local/bin/healthcheck.sh

# Security: Remove default nginx user and use custom user
RUN chown -R appuser:appgroup /usr/share/nginx/html && \
    chown -R appuser:appgroup /var/cache/nginx && \
    chown -R appuser:appgroup /var/log/nginx && \
    chown -R appuser:appgroup /etc/nginx/conf.d

# Create PID directory for nginx
RUN mkdir -p /var/run/nginx && \
    chown -R appuser:appgroup /var/run/nginx

# Security labels
LABEL maintainer="devops@company.com"
LABEL version="${REACT_APP_VERSION}"
LABEL description="Production React application"
LABEL security.scanning="enabled"

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD /usr/local/bin/healthcheck.sh

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

### 2. Next.js Dockerfile with App Router
```dockerfile
# Next.js 14+ with App Router optimized Dockerfile
FROM node:20-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Copy package files
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build arguments
ARG NEXT_PUBLIC_API_URL
ARG NEXT_PUBLIC_ENVIRONMENT
ARG DATABASE_URL
ARG NEXTAUTH_SECRET

ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
ENV NEXT_PUBLIC_ENVIRONMENT=${NEXT_PUBLIC_ENVIRONMENT}
ENV DATABASE_URL=${DATABASE_URL}
ENV NEXTAUTH_SECRET=${NEXTAUTH_SECRET}

# Disable telemetry during build
ENV NEXT_TELEMETRY_DISABLED 1

# Build Next.js application
RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

# Create system group and user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy public assets
COPY --from=builder /app/public ./public

# Set correct permissions for prerender cache
RUN mkdir .next
RUN chown nextjs:nodejs .next

# Copy built application
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Security: Run as non-root user
USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

# Health check for Next.js
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/api/health || exit 1

CMD ["node", "server.js"]
```

### 3. Full-Stack Application with Docker Compose
```yaml
# docker-compose.yml for full-stack development
version: '3.8'

services:
  # Database
  postgres:
    image: postgres:15-alpine
    container_name: app-postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-appdb}
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis for caching and sessions
  redis:
    image: redis:7-alpine
    container_name: app-redis
    command: redis-server --requirepass ${REDIS_PASSWORD:-redis123}
    volumes:
      - redis_data:/data
      - ./docker/redis/redis.conf:/usr/local/etc/redis/redis.conf
    ports:
      - "6379:6379"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Backend API
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: development
    container_name: app-backend
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD:-postgres}@postgres:5432/${POSTGRES_DB:-appdb}
      - REDIS_URL=redis://:${REDIS_PASSWORD:-redis123}@redis:6379
      - JWT_SECRET=${JWT_SECRET:-your-jwt-secret}
      - CORS_ORIGIN=http://localhost:3000
    volumes:
      - ./backend:/app
      - /app/node_modules
      - backend_logs:/app/logs
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # Frontend Application
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: development
    container_name: app-frontend
    environment:
      - NODE_ENV=development
      - REACT_APP_API_URL=http://localhost:8000
      - REACT_APP_ENVIRONMENT=development
      - CHOKIDAR_USEPOLLING=true
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - frontend_build:/app/build
    ports:
      - "3000:3000"
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  # Nginx Reverse Proxy
  nginx:
    image: nginx:1.25-alpine
    container_name: app-nginx
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./docker/ssl:/etc/nginx/ssl
      - nginx_logs:/var/log/nginx
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - frontend
      - backend
    networks:
      - app-network
    restart: unless-stopped

  # Monitoring with Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: app-prometheus
    volumes:
      - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - app-network
    restart: unless-stopped

  # Grafana for visualization
  grafana:
    image: grafana/grafana:latest
    container_name: app-grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./docker/grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3001:3000"
    networks:
      - app-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  backend_logs:
  frontend_build:
  nginx_logs:
  prometheus_data:
  grafana_data:

networks:
  app-network:
    driver: bridge
```

### 4. Production Docker Compose
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
      restart_policy:
        condition: on-failure
        max_attempts: 3

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - app-network
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

  backend:
    image: ${DOCKER_REGISTRY}/backend:${VERSION}
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
    networks:
      - app-network
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 1G
          cpus: '1'
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s

  frontend:
    image: ${DOCKER_REGISTRY}/frontend:${VERSION}
    networks:
      - app-network
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
      restart_policy:
        condition: on-failure

  nginx:
    image: ${DOCKER_REGISTRY}/nginx:${VERSION}
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ssl_certs:/etc/nginx/ssl
    networks:
      - app-network
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

volumes:
  postgres_data:
  redis_data:
  ssl_certs:

networks:
  app-network:
    driver: overlay
    attachable: true
```

## Kubernetes Deployment Strategies

### 1. Frontend Application Kubernetes Manifests
```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-app
  labels:
    name: frontend-app
    environment: production

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: frontend-app
data:
  API_URL: "https://api.example.com"
  ENVIRONMENT: "production"
  LOG_LEVEL: "info"
  FEATURE_FLAGS: "new-ui:true,analytics:true"

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: frontend-secrets
  namespace: frontend-app
type: Opaque
data:
  # Base64 encoded values
  SENTRY_DSN: <base64-encoded-sentry-dsn>
  API_KEY: <base64-encoded-api-key>

---
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
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: frontend-app
        image: your-registry.com/frontend-app:v1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        envFrom:
        - configMapRef:
            name: frontend-config
        - secretRef:
            name: frontend-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: var-cache-volume
          mountPath: /var/cache/nginx
        - name: var-run-volume
          mountPath: /var/run
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: var-cache-volume
        emptyDir: {}
      - name: var-run-volume
        emptyDir: {}
      imagePullSecrets:
      - name: registry-secret

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: frontend-app
  labels:
    app: frontend-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: frontend-app

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: frontend-app
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
    nginx.ingress.kubernetes.io/proxy-buffers-number: "8"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: frontend-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: frontend-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max

---
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: frontend-app
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: frontend-app
```

### 2. Full-Stack Kubernetes Deployment
```yaml
# k8s/full-stack/database.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: fullstack-app
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: database
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 10
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
      storageClassName: fast-ssd

---
# k8s/full-stack/backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: fullstack-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
        version: v1.0.0
    spec:
      initContainers:
      - name: migrate-database
        image: your-registry.com/backend-api:v1.0.0
        command: ['npm', 'run', 'db:migrate']
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: database-url
      containers:
      - name: backend-api
        image: your-registry.com/backend-api:v1.0.0
        ports:
        - containerPort: 8000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: redis-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10

---
# k8s/full-stack/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-web
  namespace: fullstack-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: frontend-web
  template:
    metadata:
      labels:
        app: frontend-web
        version: v1.0.0
    spec:
      containers:
      - name: frontend-web
        image: your-registry.com/frontend-web:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: API_URL
          value: "http://backend-service:8000"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
```

### 3. Advanced Deployment Strategies
```yaml
# k8s/canary-deployment.yaml
# Canary deployment with Istio
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: frontend-rollout
  namespace: frontend-app
spec:
  replicas: 5
  strategy:
    canary:
      canaryService: frontend-canary
      stableService: frontend-stable
      trafficRouting:
        istio:
          virtualService:
            name: frontend-vs
            routes:
            - primary
      steps:
      - setWeight: 10
      - pause: {duration: 10m}
      - setWeight: 20
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: frontend-canary
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
    spec:
      containers:
      - name: frontend-app
        image: your-registry.com/frontend-app:v1.1.0
        ports:
        - containerPort: 8080

---
# k8s/analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: frontend-app
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 5m
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(irate(
            istio_requests_total{reporter="destination",destination_service_name="{{args.service-name}}",response_code!~"5.*"}[5m]
          )) / 
          sum(irate(
            istio_requests_total{reporter="destination",destination_service_name="{{args.service-name}}"}[5m]
          ))

---
# k8s/blue-green-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: frontend-blue-green
  namespace: frontend-app
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: frontend-active
      previewService: frontend-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: frontend-preview
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: frontend-active
  selector:
    matchLabels:
      app: frontend-app
  template:
    metadata:
      labels:
        app: frontend-app
    spec:
      containers:
      - name: frontend-app
        image: your-registry.com/frontend-app:v1.2.0
        ports:
        - containerPort: 8080
```

### 4. Monitoring and Observability
```yaml
# k8s/monitoring/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: frontend-metrics
  namespace: frontend-app
  labels:
    app: frontend-app
spec:
  selector:
    matchLabels:
      app: frontend-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s

---
# k8s/monitoring/grafana-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  frontend-dashboard.json: |
    {
      "dashboard": {
        "title": "Frontend Application Dashboard",
        "panels": [
          {
            "title": "Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "sum(rate(nginx_http_requests_total[5m])) by (instance)",
                "legendFormat": "{{instance}}"
              }
            ]
          },
          {
            "title": "Response Time",
            "type": "graph",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, rate(nginx_http_request_duration_seconds_bucket[5m]))",
                "legendFormat": "95th percentile"
              }
            ]
          }
        ]
      }
    }

---
# k8s/monitoring/alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: frontend-alerts
  namespace: frontend-app
spec:
  groups:
  - name: frontend.rules
    rules:
    - alert: FrontendHighErrorRate
      expr: |
        (
          sum(rate(nginx_http_requests_total{status=~"5.."}[5m])) /
          sum(rate(nginx_http_requests_total[5m]))
        ) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Frontend error rate is {{ $value }} which is above the threshold of 5%"

    - alert: FrontendHighLatency
      expr: |
        histogram_quantile(0.95, rate(nginx_http_request_duration_seconds_bucket[5m])) > 2
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High latency detected"
        description: "95th percentile latency is {{ $value }}s which is above 2s"

    - alert: FrontendPodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace="frontend-app"}[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod is crash looping"
        description: "Pod {{ $labels.pod }} is restarting frequently"
```

## Container Security and Best Practices

### 1. Security Scanning and Policies
```yaml
# k8s/security/network-policy.yaml
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
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend-app
    ports:
    - protocol: TCP
      port: 8000
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53

---
# k8s/security/pod-security-policy.yaml
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
      - min: 1000
        max: 65535
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'

---
# k8s/security/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: frontend-app
  name: frontend-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-rolebinding
  namespace: frontend-app
subjects:
- kind: ServiceAccount
  name: frontend-serviceaccount
  namespace: frontend-app
roleRef:
  kind: Role
  name: frontend-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend-serviceaccount
  namespace: frontend-app
```

## Interview-Ready Summary

**Docker Best Practices:**

1. **Multi-stage builds** - Minimize final image size and attack surface
2. **Security hardening** - Non-root users, minimal base images, security scanning
3. **Layer optimization** - Strategic layer ordering, dependency caching
4. **Health checks** - Proper liveness and readiness probes
5. **Resource limits** - Memory and CPU constraints for stability

**Kubernetes Deployment Strategies:**

1. **Rolling Updates** - Zero-downtime deployments with gradual rollout
2. **Blue-Green** - Instant switchover with full environment duplication
3. **Canary** - Risk mitigation with gradual traffic shifting
4. **A/B Testing** - Feature testing with traffic splitting

**Production Considerations:**
- **Resource Management** - Requests, limits, HPA, VPA
- **Security** - Network policies, RBAC, PSP, image scanning
- **Monitoring** - Metrics, logging, alerting, tracing
- **Storage** - Persistent volumes, backup strategies
- **Networking** - Ingress, service mesh, load balancing

**Key Interview Topics:** Container orchestration, deployment strategies, scaling patterns, security hardening, monitoring integration, disaster recovery, cost optimization, multi-environment management.