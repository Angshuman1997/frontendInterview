# CI/CD Pipelines and Automated Deployment

Modern frontend and full-stack applications require robust CI/CD pipelines for automated testing, building, and deployment. This guide covers enterprise-grade CI/CD implementations across multiple platforms and deployment strategies.

## GitHub Actions for Frontend/Full-Stack Applications

### 1. Advanced Frontend CI/CD Pipeline
```yaml
# .github/workflows/frontend-ci-cd.yml
name: Frontend CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  release:
    types: [published]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Code Quality and Security Checks
  quality-checks:
    name: Code Quality & Security
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Full history for SonarCloud

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: |
          npm ci
          npm audit --audit-level=moderate

      - name: Run ESLint
        run: npm run lint:ci

      - name: Run Prettier check
        run: npm run format:check

      - name: Run TypeScript checks
        run: npm run type-check

      - name: Security scan with Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # Automated Testing
  test:
    name: Test Suite
    runs-on: ubuntu-latest
    needs: quality-checks
    strategy:
      matrix:
        test-type: [unit, integration, e2e]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Setup test environment
        if: matrix.test-type == 'e2e'
        run: |
          docker-compose -f docker-compose.test.yml up -d
          npx wait-on http://localhost:3000

      - name: Run unit tests
        if: matrix.test-type == 'unit'
        run: |
          npm run test:unit -- --coverage --watchAll=false
          npm run test:unit:ci

      - name: Run integration tests
        if: matrix.test-type == 'integration'
        run: npm run test:integration

      - name: Run E2E tests
        if: matrix.test-type == 'e2e'
        run: |
          npm run test:e2e:headless
          npm run test:e2e:mobile

      - name: Upload coverage to Codecov
        if: matrix.test-type == 'unit'
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          file: ./coverage/lcov.info

      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-artifacts-${{ matrix.test-type }}
          path: |
            cypress/screenshots
            cypress/videos
            jest-html-reporters-attach
            coverage

  # Performance and Accessibility Testing
  performance-audit:
    name: Performance & Accessibility Audit
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build
        env:
          NODE_ENV: production

      - name: Start preview server
        run: |
          npm run preview &
          npx wait-on http://localhost:4173

      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          configPath: './lighthouserc.json'
          uploadArtifacts: true
          temporaryPublicStorage: true

      - name: Bundle analyzer
        run: npm run analyze

      - name: Upload bundle analysis
        uses: actions/upload-artifact@v4
        with:
          name: bundle-analysis
          path: |
            dist/report.html
            lighthouse-results

  # Build and Package
  build:
    name: Build Application
    runs-on: ubuntu-latest
    needs: [quality-checks, test]
    strategy:
      matrix:
        environment: [staging, production]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build for ${{ matrix.environment }}
        run: npm run build:${{ matrix.environment }}
        env:
          NODE_ENV: production
          REACT_APP_API_URL: ${{ secrets[format('API_URL_{0}', upper(matrix.environment))] }}
          REACT_APP_SENTRY_DSN: ${{ secrets.SENTRY_DSN }}

      - name: Generate build info
        run: |
          echo "BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_ENV
          echo "GIT_SHA=${GITHUB_SHA::8}" >> $GITHUB_ENV
          echo "BUILD_NUMBER=${GITHUB_RUN_NUMBER}" >> $GITHUB_ENV

      - name: Create build metadata
        run: |
          cat > dist/build-info.json << EOF
          {
            "buildTime": "${{ env.BUILD_TIME }}",
            "gitSha": "${{ env.GIT_SHA }}",
            "buildNumber": "${{ env.BUILD_NUMBER }}",
            "environment": "${{ matrix.environment }}",
            "version": "${{ github.ref_name }}"
          }
          EOF

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.environment }}
          path: dist/
          retention-days: 30

  # Container Building
  build-container:
    name: Build Container Image
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name != 'pull_request'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-production
          path: dist/

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ env.BUILD_TIME }}
            VERSION=${{ github.ref_name }}
            VCS_REF=${{ github.sha }}

  # Staging Deployment
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build, build-container]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-staging
          path: dist/

      - name: Deploy to Vercel Staging
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          working-directory: ./dist
          scope: ${{ secrets.VERCEL_ORG_ID }}

      - name: Deploy to Kubernetes Staging
        run: |
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          
          kubectl set image deployment/frontend-app \
            frontend-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n staging
          
          kubectl rollout status deployment/frontend-app -n staging --timeout=300s

      - name: Run smoke tests
        run: |
          npx wait-on https://staging.example.com
          npm run test:smoke -- --baseUrl=https://staging.example.com

      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#deployments'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  # Production Deployment
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, build-container, deploy-staging]
    if: github.ref == 'refs/heads/main' || github.event_name == 'release'
    environment:
      name: production
      url: https://example.com
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-production
          path: dist/

      - name: Deploy to AWS S3 + CloudFront
        run: |
          aws s3 sync dist/ s3://${{ secrets.S3_BUCKET }} --delete
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1

      - name: Deploy to Kubernetes Production
        run: |
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          
          # Blue-green deployment
          kubectl patch deployment frontend-app \
            -p '{"spec":{"template":{"spec":{"containers":[{"name":"frontend-app","image":"${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}"}]}}}}' \
            -n production
          
          kubectl rollout status deployment/frontend-app -n production --timeout=600s

      - name: Run production health checks
        run: |
          npx wait-on https://example.com
          npm run test:health -- --baseUrl=https://example.com

      - name: Update deployment tracking
        run: |
          curl -X POST "${{ secrets.DATADOG_API_URL }}/api/v1/events" \
            -H "Content-Type: application/json" \
            -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
            -d '{
              "title": "Production Deployment",
              "text": "Frontend app deployed to production",
              "tags": ["environment:production", "service:frontend", "version:${{ github.sha }}"]
            }'

  # Security Scanning
  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: build-container
    if: github.event_name != 'pull_request'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Container structure test
        run: |
          curl -LO https://storage.googleapis.com/container-structure-test/latest/container-structure-test-linux-amd64
          chmod +x container-structure-test-linux-amd64
          ./container-structure-test-linux-amd64 test \
            --image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --config container-structure-test.yaml
```

### 2. Full-Stack Application Pipeline
```yaml
# .github/workflows/fullstack-ci-cd.yml
name: Full-Stack CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'
  PYTHON_VERSION: '3.11'
  REGISTRY: ghcr.io

jobs:
  # Database Migration and Testing
  database-migrations:
    name: Database Migrations
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: |
          npm run db:migrate
          npm run db:seed:test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb

      - name: Test database schema
        run: npm run db:test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb

  # Backend API Testing
  backend-tests:
    name: Backend API Tests
    runs-on: ubuntu-latest
    needs: database-migrations
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install Python dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run backend tests
        run: |
          pytest tests/ -v --cov=app --cov-report=xml
          python -m pytest tests/integration/ -v
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

      - name: Upload backend coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: backend

  # Frontend + Backend Integration
  integration-tests:
    name: Full-Stack Integration Tests
    runs-on: ubuntu-latest
    needs: [backend-tests]
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
      redis:
        image: redis:7
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Start backend services
        run: |
          pip install -r requirements.txt
          python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 &
          npx wait-on http://localhost:8000/health
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb

      - name: Build and start frontend
        run: |
          npm ci
          npm run build
          npm run preview &
          npx wait-on http://localhost:4173
        env:
          VITE_API_URL: http://localhost:8000

      - name: Run integration tests
        run: |
          npm run test:integration:full
          npm run test:e2e:api

      - name: API contract testing
        run: |
          npx pact-broker can-i-deploy \
            --participant frontend \
            --version ${{ github.sha }} \
            --participant backend \
            --version ${{ github.sha }}

  # Multi-service deployment
  deploy-microservices:
    name: Deploy Microservices
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        service: [api-gateway, user-service, auth-service, frontend]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy ${{ matrix.service }}
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          
          kubectl apply -f k8s/${{ matrix.service }}/
          kubectl rollout status deployment/${{ matrix.service }} -n production

      - name: Service health check
        run: |
          kubectl wait --for=condition=ready pod -l app=${{ matrix.service }} -n production --timeout=300s
```

## GitLab CI/CD Implementation

### 1. GitLab CI/CD Pipeline
```yaml
# .gitlab-ci.yml
stages:
  - validate
  - test
  - build
  - security
  - deploy
  - monitor

variables:
  NODE_VERSION: "20"
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

# Template for Node.js jobs
.node_template: &node_template
  image: node:${NODE_VERSION}
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
  before_script:
    - npm ci --cache .npm --prefer-offline

# Code Quality Stage
code_quality:
  <<: *node_template
  stage: validate
  script:
    - npm run lint:ci
    - npm run format:check
    - npm run type-check
  artifacts:
    reports:
      junit: reports/eslint-report.xml
    paths:
      - reports/
    expire_in: 1 week

dependency_scanning:
  stage: validate
  script:
    - npm audit --audit-level=moderate
    - npx better-npm-audit audit
  allow_failure: true

# Testing Stage
unit_tests:
  <<: *node_template
  stage: test
  script:
    - npm run test:unit -- --coverage --watchAll=false
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      junit: reports/jest-report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 1 week

integration_tests:
  <<: *node_template
  stage: test
  services:
    - name: postgres:15
      alias: postgres
      variables:
        POSTGRES_DB: testdb
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: postgres
  variables:
    DATABASE_URL: "postgresql://postgres:postgres@postgres:5432/testdb"
  script:
    - npm run test:integration
  artifacts:
    reports:
      junit: reports/integration-report.xml

e2e_tests:
  <<: *node_template
  stage: test
  services:
    - name: selenium/standalone-chrome:latest
      alias: selenium
  script:
    - npm run build
    - npm run preview &
    - npx wait-on http://localhost:4173
    - npm run test:e2e:headless
  artifacts:
    when: on_failure
    paths:
      - cypress/screenshots/
      - cypress/videos/
    expire_in: 1 week

# Build Stage
build_staging:
  <<: *node_template
  stage: build
  script:
    - npm run build:staging
    - echo "STAGING_BUILD_ID=${CI_PIPELINE_ID}" > dist/.env
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  only:
    - develop

build_production:
  <<: *node_template
  stage: build
  script:
    - npm run build:production
    - echo "PRODUCTION_BUILD_ID=${CI_PIPELINE_ID}" > dist/.env
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  only:
    - main
    - tags

# Container Build
build_docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_IMAGE: $CI_REGISTRY_IMAGE
    DOCKER_TAG: $CI_COMMIT_SHA
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
    - docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:$DOCKER_TAG
    - docker push $DOCKER_IMAGE:latest
  only:
    - main
    - develop

# Security Stage
container_scanning:
  stage: security
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  script:
    - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
        aquasec/trivy image --format template --template "@contrib/sarif.tpl" \
        -o gl-container-scanning-report.json $DOCKER_IMAGE
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
  only:
    - main
    - develop

sast:
  stage: security
  variables:
    SAST_EXCLUDED_PATHS: "spec, test, tests, tmp, node_modules"
  artifacts:
    reports:
      sast: gl-sast-report.json

# Deploy Staging
deploy_staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > kubeconfig
    - export KUBECONFIG=kubeconfig
    - kubectl set image deployment/frontend-app frontend-app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/frontend-app -n staging --timeout=300s
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop_staging
  only:
    - develop

stop_staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > kubeconfig
    - export KUBECONFIG=kubeconfig
    - kubectl delete deployment frontend-app -n staging
  when: manual
  environment:
    name: staging
    action: stop

# Deploy Production
deploy_production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - echo "$KUBE_CONFIG_PRODUCTION" | base64 -d > kubeconfig
    - export KUBECONFIG=kubeconfig
    - kubectl set image deployment/frontend-app frontend-app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/frontend-app -n production --timeout=600s
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main

# Performance Monitoring
lighthouse_audit:
  <<: *node_template
  stage: monitor
  script:
    - npm install -g @lhci/cli
    - lhci autorun
  artifacts:
    reports:
      performance: lhci_reports/manifest.json
  only:
    - main
    - develop

# Deployment Notifications
notify_success:
  stage: monitor
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
  script:
    - |
      curl -X POST "$SLACK_WEBHOOK_URL" \
        -H 'Content-type: application/json' \
        --data "{
          \"text\": \"✅ Deployment successful to $CI_ENVIRONMENT_NAME\",
          \"attachments\": [{
            \"color\": \"good\",
            \"fields\": [{
              \"title\": \"Environment\",
              \"value\": \"$CI_ENVIRONMENT_NAME\",
              \"short\": true
            }, {
              \"title\": \"Commit\",
              \"value\": \"$CI_COMMIT_SHA\",
              \"short\": true
            }]
          }]
        }"
  when: on_success
  only:
    - main
    - develop
```

## Azure DevOps Pipeline

### 1. Azure DevOps YAML Pipeline
```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    exclude:
      - README.md
      - docs/*

pr:
  branches:
    include:
      - main
      - develop

variables:
  nodeVersion: '20.x'
  buildConfiguration: 'Release'
  azureSubscription: 'Production'

stages:
  # Build and Test Stage
  - stage: BuildAndTest
    displayName: 'Build and Test'
    jobs:
      - job: QualityGate
        displayName: 'Code Quality Gate'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)
            displayName: 'Install Node.js'

          - task: Cache@2
            inputs:
              key: 'npm | "$(Agent.OS)" | package-lock.json'
              restoreKeys: |
                npm | "$(Agent.OS)"
              path: '~/.npm'
            displayName: 'Cache npm'

          - script: |
              npm ci
              npm run lint:ci
              npm run format:check
              npm run type-check
            displayName: 'Install dependencies and run quality checks'

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: 'reports/eslint-report.xml'
              testRunTitle: 'ESLint Results'
            condition: always()

      - job: UnitTests
        displayName: 'Unit Tests'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - script: |
              npm ci
              npm run test:unit -- --coverage --watchAll=false --ci
            displayName: 'Run unit tests'

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: 'reports/jest-report.xml'
              testRunTitle: 'Unit Test Results'

          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: 'coverage/cobertura-coverage.xml'
              reportDirectory: 'coverage/lcov-report'

      - job: E2ETests
        displayName: 'E2E Tests'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - script: |
              npm ci
              npm run build
              npm run preview &
              npx wait-on http://localhost:4173
              npm run test:e2e:headless
            displayName: 'Run E2E tests'

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: 'cypress/results/test-results.xml'
              testRunTitle: 'E2E Test Results'

  # Security and Compliance
  - stage: Security
    displayName: 'Security Scanning'
    dependsOn: BuildAndTest
    jobs:
      - job: SecurityScan
        displayName: 'Security Scanning'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - script: |
              npm ci
              npm audit --audit-level=moderate
              npx better-npm-audit audit
            displayName: 'Dependency security scan'

          - task: CredScan@3
            inputs:
              toolMajorVersion: 'V2'

          - task: AntiMalware@4
            inputs:
              InputType: 'Basic'
              ScanType: 'CustomScan'
              FileDirPath: '$(Build.StagingDirectory)'

  # Build Stage
  - stage: Build
    displayName: 'Build Application'
    dependsOn: [BuildAndTest, Security]
    jobs:
      - job: BuildApp
        displayName: 'Build Application'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - script: |
              npm ci
              npm run build:production
            displayName: 'Build production app'
            env:
              NODE_ENV: production

          - task: Docker@2
            displayName: 'Build Docker image'
            inputs:
              containerRegistry: 'ACR Connection'
              repository: 'frontend-app'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'dist'
              ArtifactName: 'frontend-build'

  # Deploy to Staging
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployToStaging
        displayName: 'Deploy to Staging Environment'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadBuildArtifacts@0
                  inputs:
                    buildType: 'current'
                    downloadType: 'single'
                    artifactName: 'frontend-build'

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: $(azureSubscription)
                    appName: 'frontend-staging'
                    package: '$(System.ArtifactsDirectory)/frontend-build'

                - task: KubernetesManifest@0
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'k8s-staging'
                    namespace: 'staging'
                    manifests: |
                      k8s/staging/*.yaml

  # Deploy to Production
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToProduction
        displayName: 'Deploy to Production Environment'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadBuildArtifacts@0
                  inputs:
                    buildType: 'current'
                    downloadType: 'single'
                    artifactName: 'frontend-build'

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: $(azureSubscription)
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az storage blob upload-batch \
                        --destination '$web' \
                        --source $(System.ArtifactsDirectory)/frontend-build \
                        --account-name frontendstorage

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: $(azureSubscription)
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az cdn endpoint purge \
                        --content-paths "/*" \
                        --profile-name frontend-cdn \
                        --name frontend-endpoint \
                        --resource-group frontend-rg

      # Post-deployment validation
      - job: PostDeploymentValidation
        displayName: 'Post-deployment Validation'
        dependsOn: DeployToProduction
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: |
              npx wait-on https://example.com
              npm run test:smoke -- --baseUrl=https://example.com
            displayName: 'Run smoke tests'

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: 'smoke-test-results.xml'
              testRunTitle: 'Smoke Test Results'
```

## Jenkins Pipeline

### 1. Jenkinsfile for Frontend/Full-Stack
```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: node
                    image: node:20
                    command:
                    - cat
                    tty: true
                  - name: docker
                    image: docker:24
                    command:
                    - cat
                    tty: true
                    volumeMounts:
                    - mountPath: /var/run/docker.sock
                      name: docker-sock
                  volumes:
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            """
        }
    }

    environment {
        NODE_VERSION = '20'
        DOCKER_REGISTRY = 'your-registry.com'
        APP_NAME = 'frontend-app'
        SONAR_TOKEN = credentials('sonar-token')
        SLACK_WEBHOOK = credentials('slack-webhook')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.BUILD_TIME = sh(
                        script: "date -u +%Y-%m-%dT%H:%M:%SZ",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                container('node') {
                    sh '''
                        npm ci
                        npm ls --depth=0
                    '''
                }
            }
        }

        stage('Code Quality') {
            parallel {
                stage('Lint') {
                    steps {
                        container('node') {
                            sh 'npm run lint:ci'
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'reports',
                                reportFiles: 'eslint-report.html',
                                reportName: 'ESLint Report'
                            ])
                        }
                    }
                }

                stage('Type Check') {
                    steps {
                        container('node') {
                            sh 'npm run type-check'
                        }
                    }
                }

                stage('Security Audit') {
                    steps {
                        container('node') {
                            sh '''
                                npm audit --audit-level=moderate
                                npx better-npm-audit audit --level moderate
                            '''
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('node') {
                    script {
                        def scannerHome = tool 'SonarQubeScanner'
                        withSonarQubeEnv('SonarQube') {
                            sh """
                                ${scannerHome}/bin/sonar-scanner \\
                                    -Dsonar.projectKey=${APP_NAME} \\
                                    -Dsonar.sources=src \\
                                    -Dsonar.tests=src \\
                                    -Dsonar.test.inclusions='**/*.test.ts,**/*.test.tsx' \\
                                    -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info \\
                                    -Dsonar.testExecutionReportPaths=reports/test-report.xml
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('node') {
                            sh '''
                                npm run test:unit -- --coverage --watchAll=false --ci
                                npm run test:unit:ci
                            '''
                            
                            publishTestResults([
                                testResultsPattern: 'reports/jest-report.xml'
                            ])
                            
                            publishCoverageGoberturaReport([
                                coberturaReportFile: 'coverage/cobertura-coverage.xml'
                            ])
                        }
                    }
                }

                stage('Integration Tests') {
                    steps {
                        container('node') {
                            sh '''
                                docker-compose -f docker-compose.test.yml up -d
                                npm run test:integration
                                docker-compose -f docker-compose.test.yml down
                            '''
                        }
                    }
                }

                stage('E2E Tests') {
                    steps {
                        container('node') {
                            sh '''
                                npm run build
                                npm run preview &
                                npx wait-on http://localhost:4173
                                npm run test:e2e:headless
                            '''
                            
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'cypress/reports',
                                reportFiles: 'index.html',
                                reportName: 'Cypress Report'
                            ])
                        }
                        
                        archiveArtifacts artifacts: 'cypress/screenshots/**/*', allowEmptyArchive: true
                        archiveArtifacts artifacts: 'cypress/videos/**/*', allowEmptyArchive: true
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'cypress/results/*.xml'
                        }
                    }
                }
            }
        }

        stage('Build') {
            parallel {
                stage('Build Staging') {
                    when {
                        branch 'develop'
                    }
                    steps {
                        container('node') {
                            sh '''
                                npm run build:staging
                                echo "BUILD_ID=${BUILD_NUMBER}" > dist/.env
                                echo "GIT_COMMIT=${GIT_COMMIT_SHORT}" >> dist/.env
                                echo "BUILD_TIME=${BUILD_TIME}" >> dist/.env
                            '''
                            
                            archiveArtifacts artifacts: 'dist/**/*', fingerprint: true
                        }
                    }
                }

                stage('Build Production') {
                    when {
                        anyOf {
                            branch 'main'
                            buildingTag()
                        }
                    }
                    steps {
                        container('node') {
                            sh '''
                                npm run build:production
                                echo "BUILD_ID=${BUILD_NUMBER}" > dist/.env
                                echo "GIT_COMMIT=${GIT_COMMIT_SHORT}" >> dist/.env
                                echo "BUILD_TIME=${BUILD_TIME}" >> dist/.env
                            '''
                            
                            archiveArtifacts artifacts: 'dist/**/*', fingerprint: true
                        }
                    }
                }
            }
        }

        stage('Performance Audit') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                container('node') {
                    sh '''
                        npm install -g @lhci/cli
                        lhci autorun
                    '''
                    
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'lhci_reports',
                        reportFiles: '*.html',
                        reportName: 'Lighthouse Report'
                    ])
                }
            }
        }

        stage('Docker Build') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                container('docker') {
                    script {
                        def image = docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT}")
                        
                        docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                            image.push()
                            image.push('latest')
                        }
                    }
                }
            }
        }

        stage('Security Scan') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                container('docker') {
                    sh '''
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \\
                            aquasec/trivy image --format json --output trivy-report.json \\
                            ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT}
                    '''
                    
                    archiveArtifacts artifacts: 'trivy-report.json'
                }
            }
        }

        stage('Deploy Staging') {
            when {
                branch 'develop'
            }
            environment {
                KUBECONFIG = credentials('kubeconfig-staging')
            }
            steps {
                container('kubectl') {
                    sh '''
                        kubectl set image deployment/${APP_NAME} \\
                            ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT} \\
                            -n staging
                        
                        kubectl rollout status deployment/${APP_NAME} \\
                            -n staging --timeout=300s
                    '''
                }
            }
        }

        stage('Deploy Production') {
            when {
                branch 'main'
            }
            environment {
                KUBECONFIG = credentials('kubeconfig-production')
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                
                container('kubectl') {
                    sh '''
                        kubectl set image deployment/${APP_NAME} \\
                            ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT} \\
                            -n production
                        
                        kubectl rollout status deployment/${APP_NAME} \\
                            -n production --timeout=600s
                    '''
                }
            }
        }

        stage('Smoke Tests') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                container('node') {
                    script {
                        def baseUrl = env.BRANCH_NAME == 'main' ? 
                            'https://example.com' : 'https://staging.example.com'
                        
                        sh """
                            npx wait-on ${baseUrl}
                            npm run test:smoke -- --baseUrl=${baseUrl}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            container('node') {
                // Clean up
                sh 'docker-compose -f docker-compose.test.yml down || true'
            }
        }
        
        success {
            script {
                def message = """
                    ✅ *Deployment Successful*
                    
                    *Environment:* ${env.BRANCH_NAME == 'main' ? 'Production' : 'Staging'}
                    *Commit:* ${env.GIT_COMMIT_SHORT}
                    *Build:* ${env.BUILD_NUMBER}
                    *Duration:* ${currentBuild.durationString}
                """
                
                sh """
                    curl -X POST "${SLACK_WEBHOOK}" \\
                        -H 'Content-type: application/json' \\
                        --data '{"text": "${message}"}'
                """
            }
        }
        
        failure {
            script {
                def message = """
                    ❌ *Deployment Failed*
                    
                    *Branch:* ${env.BRANCH_NAME}
                    *Commit:* ${env.GIT_COMMIT_SHORT}
                    *Build:* ${env.BUILD_NUMBER}
                    *Stage:* ${env.STAGE_NAME}
                """
                
                sh """
                    curl -X POST "${SLACK_WEBHOOK}" \\
                        -H 'Content-type: application/json' \\
                        --data '{"text": "${message}"}'
                """
            }
        }
    }
}
```

## Interview-Ready Summary

**CI/CD Pipeline Components:**

1. **Quality Gates** - Code quality, security scanning, dependency auditing
2. **Testing Strategy** - Unit, integration, E2E, performance, smoke tests
3. **Build Optimization** - Caching, parallel jobs, artifact management
4. **Security Integration** - SAST, dependency scanning, container security
5. **Deployment Strategies** - Blue-green, rolling, canary deployments
6. **Monitoring Integration** - Health checks, performance monitoring, alerting

**Platform Differences:**
- **GitHub Actions**: YAML-based, excellent GitHub integration, marketplace ecosystem
- **GitLab CI/CD**: Built-in security scanning, comprehensive DevOps platform
- **Azure DevOps**: Enterprise features, Microsoft ecosystem integration
- **Jenkins**: Highly customizable, plugin ecosystem, on-premise flexibility

**Best Practices:**
- **Pipeline as Code** - Version controlled, reviewable, reproducible
- **Environment Parity** - Consistent environments across stages
- **Fast Feedback** - Quick pipeline execution, early failure detection
- **Security First** - Security scanning at every stage
- **Observability** - Comprehensive logging, monitoring, alerting

**Key Interview Topics:** Pipeline optimization, deployment strategies, security integration, testing automation, monitoring setup, rollback procedures, environment management, infrastructure as code.