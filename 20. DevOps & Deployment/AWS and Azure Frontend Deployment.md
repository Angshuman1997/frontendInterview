# AWS and Azure Frontend Deployment Strategies

Cloud deployment for frontend applications requires understanding of CDN distribution, edge computing, serverless architectures, and multi-region strategies. This comprehensive guide covers enterprise-grade deployment patterns for AWS and Azure, with Infrastructure as Code examples and production best practices.

## AWS Frontend Deployment Architecture

### CloudFront + S3 Static Website Hosting

```typescript
// AWS CDK Infrastructure for Frontend Deployment
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as route53 from 'aws-cdk-lib/aws-route53';
import * as targets from 'aws-cdk-lib/aws-route53-targets';
import * as certificatemanager from 'aws-cdk-lib/aws-certificatemanager';

interface FrontendDeploymentProps extends cdk.StackProps {
  domainName: string;
  certificateArn: string;
  environment: 'development' | 'staging' | 'production';
  enableWAF?: boolean;
  enableLogging?: boolean;
  cachingStrategy?: 'aggressive' | 'standard' | 'minimal';
}

export class FrontendDeploymentStack extends cdk.Stack {
  public readonly distribution: cloudfront.Distribution;
  public readonly bucket: s3.Bucket;
  public readonly originAccessIdentity: cloudfront.OriginAccessIdentity;

  constructor(scope: cdk.App, id: string, props: FrontendDeploymentProps) {
    super(scope, id, props);

    // S3 Bucket for static assets
    this.bucket = new s3.Bucket(this, 'FrontendAssetsBucket', {
      bucketName: `${props.domainName.replace(/\./g, '-')}-${props.environment}`,
      removalPolicy: props.environment === 'production' 
        ? cdk.RemovalPolicy.RETAIN 
        : cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: props.environment !== 'production',
      versioned: true,
      publicReadAccess: false,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      lifecycleRules: [{
        id: 'DeleteIncompleteMultipartUploads',
        abortIncompleteMultipartUploadAfter: cdk.Duration.days(7),
      }],
      cors: [{
        allowedMethods: [s3.HttpMethods.GET, s3.HttpMethods.HEAD],
        allowedOrigins: ['*'],
        allowedHeaders: ['*'],
        maxAge: 3000,
      }]
    });

    // Origin Access Identity for secure S3 access
    this.originAccessIdentity = new cloudfront.OriginAccessIdentity(this, 'OAI', {
      comment: `OAI for ${props.domainName} frontend`
    });

    // Grant CloudFront access to S3 bucket
    this.bucket.grantRead(this.originAccessIdentity);

    // CloudFront Distribution
    this.distribution = new cloudfront.Distribution(this, 'FrontendDistribution', {
      defaultRootObject: 'index.html',
      domainNames: [props.domainName],
      certificate: certificatemanager.Certificate.fromCertificateArn(
        this, 'Certificate', props.certificateArn
      ),
      
      // Origin configuration
      defaultBehavior: {
        origin: new origins.S3Origin(this.bucket, {
          originAccessIdentity: this.originAccessIdentity
        }),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cachePolicy: this.createCachePolicy(props.cachingStrategy || 'standard'),
        compress: true,
        allowedMethods: cloudfront.AllowedMethods.ALLOW_GET_HEAD_OPTIONS,
        cachedMethods: cloudfront.CachedMethods.CACHE_GET_HEAD_OPTIONS,
        responseHeadersPolicy: this.createResponseHeadersPolicy()
      },

      // Additional behaviors for different file types
      additionalBehaviors: {
        '/api/*': {
          origin: new origins.HttpOrigin('api.example.com'),
          viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.HTTPS_ONLY,
          cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
          originRequestPolicy: cloudfront.OriginRequestPolicy.CORS_S3_ORIGIN,
          allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL,
          cachedMethods: cloudfront.CachedMethods.CACHE_GET_HEAD_OPTIONS
        },
        '/static/*': {
          origin: new origins.S3Origin(this.bucket, {
            originAccessIdentity: this.originAccessIdentity
          }),
          viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
          cachePolicy: this.createStaticAssetsCachePolicy(),
          compress: true
        }
      },

      // Error pages
      errorResponses: [
        {
          httpStatus: 404,
          responseHttpStatus: 200,
          responsePagePath: '/index.html',
          ttl: cdk.Duration.minutes(5)
        },
        {
          httpStatus: 403,
          responseHttpStatus: 200,
          responsePagePath: '/index.html',
          ttl: cdk.Duration.minutes(5)
        }
      ],

      // Geo restriction
      geoRestriction: props.environment === 'production' 
        ? cloudfront.GeoRestriction.allowlist('US', 'CA', 'GB', 'DE', 'FR')
        : undefined,

      // Logging
      enableLogging: props.enableLogging,
      logBucket: props.enableLogging ? this.createLogsBucket() : undefined,
      logFilePrefix: `cloudfront-logs/${props.environment}/`,

      // Security
      minimumProtocolVersion: cloudfront.SecurityPolicyProtocol.TLS_V1_2_2021,
      httpVersion: cloudfront.HttpVersion.HTTP2,

      // Price class
      priceClass: props.environment === 'production' 
        ? cloudfront.PriceClass.PRICE_CLASS_ALL
        : cloudfront.PriceClass.PRICE_CLASS_100
    });

    // Route53 DNS Record
    const hostedZone = route53.HostedZone.fromLookup(this, 'HostedZone', {
      domainName: this.extractRootDomain(props.domainName)
    });

    new route53.ARecord(this, 'AliasRecord', {
      zone: hostedZone,
      recordName: props.domainName,
      target: route53.RecordTarget.fromAlias(
        new targets.CloudFrontTarget(this.distribution)
      )
    });

    // WAF (Web Application Firewall) - Optional
    if (props.enableWAF) {
      this.attachWAF();
    }

    // Outputs
    new cdk.CfnOutput(this, 'DistributionDomainName', {
      value: this.distribution.distributionDomainName,
      description: 'CloudFront Distribution Domain Name'
    });

    new cdk.CfnOutput(this, 'DistributionId', {
      value: this.distribution.distributionId,
      description: 'CloudFront Distribution ID'
    });

    new cdk.CfnOutput(this, 'S3BucketName', {
      value: this.bucket.bucketName,
      description: 'S3 Bucket Name'
    });
  }

  private createCachePolicy(strategy: 'aggressive' | 'standard' | 'minimal'): cloudfront.CachePolicy {
    const cachePolicyProps: cloudfront.CachePolicyProps = {
      cachePolicyName: `${this.stackName}-cache-policy`,
      comment: `Cache policy for ${strategy} caching`,
      enableAcceptEncodingBrotli: true,
      enableAcceptEncodingGzip: true,
      queryStringBehavior: cloudfront.CacheQueryStringBehavior.none(),
      headerBehavior: cloudfront.CacheHeaderBehavior.allowList(
        'Accept',
        'Accept-Language',
        'Authorization'
      ),
      cookieBehavior: cloudfront.CacheCookieBehavior.none()
    };

    switch (strategy) {
      case 'aggressive':
        cachePolicyProps.defaultTtl = cdk.Duration.days(1);
        cachePolicyProps.maxTtl = cdk.Duration.days(365);
        cachePolicyProps.minTtl = cdk.Duration.hours(1);
        break;
      case 'standard':
        cachePolicyProps.defaultTtl = cdk.Duration.hours(24);
        cachePolicyProps.maxTtl = cdk.Duration.days(30);
        cachePolicyProps.minTtl = cdk.Duration.minutes(1);
        break;
      case 'minimal':
        cachePolicyProps.defaultTtl = cdk.Duration.minutes(5);
        cachePolicyProps.maxTtl = cdk.Duration.hours(1);
        cachePolicyProps.minTtl = cdk.Duration.seconds(0);
        break;
    }

    return new cloudfront.CachePolicy(this, 'CachePolicy', cachePolicyProps);
  }

  private createStaticAssetsCachePolicy(): cloudfront.CachePolicy {
    return new cloudfront.CachePolicy(this, 'StaticAssetsCachePolicy', {
      cachePolicyName: `${this.stackName}-static-assets-cache-policy`,
      comment: 'Cache policy for static assets with long TTL',
      defaultTtl: cdk.Duration.days(30),
      maxTtl: cdk.Duration.days(365),
      minTtl: cdk.Duration.days(1),
      enableAcceptEncodingBrotli: true,
      enableAcceptEncodingGzip: true,
      queryStringBehavior: cloudfront.CacheQueryStringBehavior.none(),
      headerBehavior: cloudfront.CacheHeaderBehavior.none(),
      cookieBehavior: cloudfront.CacheCookieBehavior.none()
    });
  }

  private createResponseHeadersPolicy(): cloudfront.ResponseHeadersPolicy {
    return new cloudfront.ResponseHeadersPolicy(this, 'ResponseHeadersPolicy', {
      responseHeadersPolicyName: `${this.stackName}-security-headers`,
      comment: 'Security headers for frontend application',
      securityHeadersBehavior: {
        contentTypeOptions: { override: true },
        frameOptions: { frameOption: cloudfront.HeadersFrameOption.DENY, override: true },
        referrerPolicy: { 
          referrerPolicy: cloudfront.HeadersReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN, 
          override: true 
        },
        strictTransportSecurity: {
          accessControlMaxAge: cdk.Duration.seconds(31536000),
          includeSubdomains: true,
          preload: true,
          override: true
        },
        xssProtection: { 
          modeBlock: true, 
          protection: true, 
          override: true 
        }
      },
      customHeadersBehavior: {
        customHeaders: [
          {
            header: 'Permissions-Policy',
            value: 'camera=(), microphone=(), geolocation=()',
            override: true
          },
          {
            header: 'X-Robots-Tag',
            value: 'index, follow',
            override: true
          }
        ]
      }
    });
  }

  private createLogsBucket(): s3.Bucket {
    return new s3.Bucket(this, 'LogsBucket', {
      bucketName: `${this.bucket.bucketName}-logs`,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      lifecycleRules: [{
        id: 'DeleteOldLogs',
        expiration: cdk.Duration.days(90)
      }]
    });
  }

  private attachWAF(): void {
    // WAF implementation would go here
    // This would include rate limiting, geo-blocking, and common attack protection
  }

  private extractRootDomain(domainName: string): string {
    const parts = domainName.split('.');
    return parts.slice(-2).join('.');
  }
}

// Lambda@Edge for Dynamic Routing
export class EdgeRoutingFunction {
  static createViewerRequestFunction(): cloudfront.Function {
    return new cloudfront.Function(this, 'ViewerRequestFunction', {
      code: cloudfront.FunctionCode.fromInline(`
        function handler(event) {
          var request = event.request;
          var uri = request.uri;
          
          // Handle SPA routing
          if (uri !== '/' && !uri.includes('.') && !uri.includes('/api/')) {
            request.uri = '/index.html';
          }
          
          // Add security headers
          if (!request.headers['x-forwarded-proto'] || 
              request.headers['x-forwarded-proto'].value !== 'https') {
            return {
              statusCode: 301,
              statusDescription: 'Moved Permanently',
              headers: {
                'location': { value: 'https://' + request.headers.host.value + request.uri }
              }
            };
          }
          
          return request;
        }
      `),
      comment: 'SPA routing and security headers'
    });
  }

  static createOriginResponseFunction(): cloudfront.Function {
    return new cloudfront.Function(this, 'OriginResponseFunction', {
      code: cloudfront.FunctionCode.fromInline(`
        function handler(event) {
          var response = event.response;
          var headers = response.headers;
          
          // Add cache control headers based on content type
          var contentType = headers['content-type'] ? headers['content-type'].value : '';
          
          if (contentType.includes('text/html')) {
            headers['cache-control'] = { value: 'no-cache, no-store, must-revalidate' };
          } else if (contentType.includes('application/javascript') || 
                     contentType.includes('text/css')) {
            headers['cache-control'] = { value: 'public, max-age=31536000, immutable' };
          } else if (contentType.includes('image/')) {
            headers['cache-control'] = { value: 'public, max-age=2592000' };
          }
          
          return response;
        }
      `),
      comment: 'Dynamic cache control headers'
    });
  }
}
```

## Azure Frontend Deployment Architecture

### Azure Static Web Apps + Azure CDN

```typescript
// Azure Resource Manager Template (using Bicep)
// main.bicep
param appName string
param environment string = 'production'
param location string = resourceGroup().location
param customDomainName string = ''
param enableCDN bool = true
param enableApplicationInsights bool = true

// Variables
var staticWebAppName = '${appName}-${environment}'
var cdnProfileName = '${appName}-cdn-${environment}'
var cdnEndpointName = '${appName}-endpoint-${environment}'
var applicationInsightsName = '${appName}-insights-${environment}'

// Static Web App
resource staticWebApp 'Microsoft.Web/staticSites@2022-03-01' = {
  name: staticWebAppName
  location: location
  sku: {
    name: environment == 'production' ? 'Standard' : 'Free'
    tier: environment == 'production' ? 'Standard' : 'Free'
  }
  properties: {
    repositoryUrl: 'https://github.com/your-org/your-repo'
    branch: environment == 'production' ? 'main' : 'develop'
    buildProperties: {
      appLocation: '/'
      apiLocation: 'api'
      outputLocation: 'dist'
      appBuildCommand: 'npm run build'
      apiBuildCommand: 'npm run build:api'
    }
    allowConfigFileUpdates: true
    enterpriseGradeCdnStatus: 'Enabled'
  }
}

// Application Insights
resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = if (enableApplicationInsights) {
  name: applicationInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

// CDN Profile
resource cdnProfile 'Microsoft.Cdn/profiles@2021-06-01' = if (enableCDN) {
  name: cdnProfileName
  location: 'Global'
  sku: {
    name: environment == 'production' ? 'Premium_AzureFrontDoor' : 'Standard_Microsoft'
  }
  properties: {
    originResponseTimeoutSeconds: 30
  }
}

// CDN Endpoint
resource cdnEndpoint 'Microsoft.Cdn/profiles/endpoints@2021-06-01' = if (enableCDN) {
  parent: cdnProfile
  name: cdnEndpointName
  location: 'Global'
  properties: {
    origins: [
      {
        name: 'staticwebapp'
        properties: {
          hostName: staticWebApp.properties.defaultHostname
          httpsPort: 443
          originHostHeader: staticWebApp.properties.defaultHostname
          priority: 1
          weight: 1000
          enabled: true
        }
      }
    ]
    originGroups: []
    defaultOriginGroup: {
      id: resourceId('Microsoft.Cdn/profiles/endpoints/originGroups', cdnProfile.name, cdnEndpointName, 'default')
    }
    deliveryPolicy: {
      rules: [
        {
          name: 'SPA-routing'
          order: 1
          conditions: [
            {
              name: 'UrlPath'
              parameters: {
                operator: 'DoesNotContain'
                matchValues: ['.']
                transforms: []
              }
            }
            {
              name: 'UrlPath'
              parameters: {
                operator: 'DoesNotBeginWith'
                matchValues: ['/api/']
                transforms: []
              }
            }
          ]
          actions: [
            {
              name: 'UrlRewrite'
              parameters: {
                sourcePattern: '/'
                destination: '/index.html'
                preserveUnmatchedPath: false
              }
            }
          ]
        }
        {
          name: 'Cache-static-assets'
          order: 2
          conditions: [
            {
              name: 'UrlFileExtension'
              parameters: {
                operator: 'Equal'
                matchValues: ['js', 'css', 'png', 'jpg', 'jpeg', 'gif', 'svg', 'woff', 'woff2']
                transforms: []
              }
            }
          ]
          actions: [
            {
              name: 'CacheExpiration'
              parameters: {
                cacheBehavior: 'Override'
                cacheType: 'All'
                cacheDuration: '365.00:00:00'
              }
            }
          ]
        }
      ]
    }
    optimizationType: 'GeneralWebDelivery'
    queryStringCachingBehavior: 'IgnoreQueryString'
    isHttpAllowed: false
    isHttpsAllowed: true
    isCompressionEnabled: true
    contentTypesToCompress: [
      'application/eot'
      'application/font'
      'application/font-sfnt'
      'application/javascript'
      'application/json'
      'application/opentype'
      'application/otf'
      'application/pkcs7-mime'
      'application/truetype'
      'application/ttf'
      'application/vnd.ms-fontobject'
      'application/xhtml+xml'
      'application/xml'
      'application/xml+rss'
      'application/x-font-opentype'
      'application/x-font-truetype'
      'application/x-font-ttf'
      'application/x-httpd-cgi'
      'application/x-javascript'
      'application/x-mpegurl'
      'application/x-opentype'
      'application/x-otf'
      'application/x-perl'
      'application/x-ttf'
      'font/eot'
      'font/ttf'
      'font/otf'
      'font/opentype'
      'image/svg+xml'
      'text/css'
      'text/csv'
      'text/html'
      'text/javascript'
      'text/js'
      'text/plain'
      'text/richtext'
      'text/tab-separated-values'
      'text/xml'
      'text/x-script'
      'text/x-component'
      'text/x-java-source'
    ]
  }
}

// Custom Domain (if provided)
resource customDomain 'Microsoft.Cdn/profiles/endpoints/customDomains@2021-06-01' = if (enableCDN && !empty(customDomainName)) {
  parent: cdnEndpoint
  name: replace(customDomainName, '.', '-')
  properties: {
    hostName: customDomainName
    httpsParameters: {
      certificateSource: 'Cdn'
      protocolType: 'ServerNameIndication'
      minimumTlsVersion: 'TLS12'
    }
  }
}

// Output values
output staticWebAppUrl string = staticWebApp.properties.defaultHostname
output staticWebAppApiKey string = staticWebApp.listSecrets().properties.apiKey
output cdnEndpointUrl string = enableCDN ? cdnEndpoint.properties.hostName : ''
output applicationInsightsInstrumentationKey string = enableApplicationInsights ? applicationInsights.properties.InstrumentationKey : ''
```

## Multi-Cloud Deployment Strategy

### Terraform Configuration for Multi-Cloud Setup

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

# Variables
variable "app_name" {
  description = "Application name"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
}

variable "domain_name" {
  description = "Custom domain name"
  type        = string
}

variable "enable_multi_cloud" {
  description = "Enable multi-cloud deployment"
  type        = bool
  default     = false
}

# AWS Provider
provider "aws" {
  region = var.aws_region
}

# Azure Provider
provider "azurerm" {
  features {}
}

# AWS S3 + CloudFront Module
module "aws_frontend" {
  source = "./modules/aws-frontend"
  
  app_name        = var.app_name
  environment     = var.environment
  domain_name     = var.domain_name
  certificate_arn = var.aws_certificate_arn
  
  enable_waf     = var.environment == "production"
  enable_logging = true
  
  tags = {
    Environment = var.environment
    Application = var.app_name
    ManagedBy   = "Terraform"
  }
}

# Azure Static Web Apps Module
module "azure_frontend" {
  count  = var.enable_multi_cloud ? 1 : 0
  source = "./modules/azure-frontend"
  
  app_name         = var.app_name
  environment      = var.environment
  location         = var.azure_location
  custom_domain    = var.domain_name
  
  enable_cdn                   = true
  enable_application_insights  = true
  
  tags = {
    Environment = var.environment
    Application = var.app_name
    ManagedBy   = "Terraform"
  }
}

# Global Load Balancer for Multi-Cloud (using AWS Route 53)
resource "aws_route53_health_check" "aws_primary" {
  count                           = var.enable_multi_cloud ? 1 : 0
  fqdn                           = module.aws_frontend.cloudfront_domain_name
  port                           = 443
  type                           = "HTTPS"
  resource_path                  = "/health"
  failure_threshold              = 3
  request_interval               = 30
  cloudwatch_logs_region         = var.aws_region
  cloudwatch_alarm_region        = var.aws_region
  insufficient_data_health_status = "Failure"

  tags = {
    Name = "${var.app_name}-aws-health-check"
  }
}

resource "aws_route53_health_check" "azure_secondary" {
  count                           = var.enable_multi_cloud ? 1 : 0
  fqdn                           = module.azure_frontend[0].static_web_app_url
  port                           = 443
  type                           = "HTTPS"
  resource_path                  = "/health"
  failure_threshold              = 3
  request_interval               = 30
  cloudwatch_logs_region         = var.aws_region
  cloudwatch_alarm_region        = var.aws_region
  insufficient_data_health_status = "Failure"

  tags = {
    Name = "${var.app_name}-azure-health-check"
  }
}

# DNS Failover Configuration
resource "aws_route53_record" "primary" {
  count           = var.enable_multi_cloud ? 1 : 0
  zone_id         = data.aws_route53_zone.main.zone_id
  name            = var.domain_name
  type            = "A"
  set_identifier  = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  health_check_id = aws_route53_health_check.aws_primary[0].id
  
  alias {
    name                   = module.aws_frontend.cloudfront_domain_name
    zone_id                = module.aws_frontend.cloudfront_hosted_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "secondary" {
  count           = var.enable_multi_cloud ? 1 : 0
  zone_id         = data.aws_route53_zone.main.zone_id
  name            = var.domain_name
  type            = "CNAME"
  ttl             = 60
  set_identifier  = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  health_check_id = aws_route53_health_check.azure_secondary[0].id
  records         = [module.azure_frontend[0].static_web_app_url]
}

# Outputs
output "primary_url" {
  value = module.aws_frontend.cloudfront_domain_name
}

output "secondary_url" {
  value = var.enable_multi_cloud ? module.azure_frontend[0].static_web_app_url : null
}

output "deployment_info" {
  value = {
    aws_distribution_id     = module.aws_frontend.distribution_id
    aws_s3_bucket          = module.aws_frontend.s3_bucket_name
    azure_app_name         = var.enable_multi_cloud ? module.azure_frontend[0].static_web_app_name : null
    azure_cdn_endpoint     = var.enable_multi_cloud ? module.azure_frontend[0].cdn_endpoint_url : null
  }
}
```

## Deployment Automation and CI/CD Integration

### GitHub Actions Workflow for Multi-Cloud Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy Frontend Application

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18.x'
  AWS_REGION: 'us-east-1'
  AZURE_REGION: 'East US'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test:ci

      - name: Run security audit
        run: npm audit --audit-level high

      - name: Build application
        run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-files
          path: dist/

  deploy-aws:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-files
          path: dist/

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to S3
        run: |
          aws s3 sync dist/ s3://${{ secrets.AWS_S3_BUCKET }} --delete
          
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.AWS_CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"

  deploy-azure:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-files
          path: dist/

      - name: Deploy to Azure Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: 'upload'
          app_location: 'dist'
          api_location: ''
          output_location: ''

  infrastructure:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init
        working-directory: ./terraform

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        working-directory: ./terraform
        env:
          TF_VAR_app_name: ${{ secrets.APP_NAME }}
          TF_VAR_environment: production
          TF_VAR_domain_name: ${{ secrets.DOMAIN_NAME }}

      - name: Terraform Apply
        if: github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
        working-directory: ./terraform

  performance-test:
    needs: [deploy-aws, deploy-azure]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install Lighthouse CI
        run: npm install -g @lhci/cli@0.12.x

      - name: Run Lighthouse CI
        run: lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}

  security-scan:
    needs: [deploy-aws]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Security Headers Check
        run: |
          curl -s -I https://${{ secrets.DOMAIN_NAME }} | grep -E "(Strict-Transport-Security|Content-Security-Policy|X-Frame-Options)"
          
      - name: SSL Labs Test
        run: |
          curl -s "https://api.ssllabs.com/api/v3/analyze?host=${{ secrets.DOMAIN_NAME }}" | jq '.grade'
```

## Edge Computing and Global Distribution

### CloudFlare Workers for Edge Computing

```typescript
// cloudflare-worker.ts
import { Router } from 'itty-router';

// Create router
const router = Router();

// Edge caching configuration
interface CacheConfig {
  html: number;
  static: number;
  api: number;
}

const CACHE_CONFIG: CacheConfig = {
  html: 300,      // 5 minutes
  static: 86400,  // 24 hours
  api: 0          // No cache
};

// Security headers
const SECURITY_HEADERS = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=()'
};

// Main route handler
router.get('*', async (request: Request, env: Env) => {
  const url = new URL(request.url);
  const { pathname } = url;

  // Handle API routes
  if (pathname.startsWith('/api/')) {
    return handleAPIRoute(request, env);
  }

  // Handle static assets
  if (isStaticAsset(pathname)) {
    return handleStaticAsset(request, env);
  }

  // Handle SPA routing
  return handleSPARoute(request, env);
});

// Handle API routes with regional optimization
async function handleAPIRoute(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  
  // Determine optimal region based on user location
  const region = getOptimalRegion(request);
  const apiBaseUrl = env[`API_BASE_${region.toUpperCase()}`] || env.API_BASE_DEFAULT;
  
  // Proxy to regional API
  const apiUrl = `${apiBaseUrl}${url.pathname}${url.search}`;
  
  const apiRequest = new Request(apiUrl, {
    method: request.method,
    headers: request.headers,
    body: request.body
  });

  const response = await fetch(apiRequest);
  
  // Add CORS headers
  const corsHeaders = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization'
  };

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: { ...response.headers, ...corsHeaders }
  });
}

// Handle static assets with aggressive caching
async function handleStaticAsset(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  const cacheKey = new Request(url.toString(), request);
  const cache = caches.default;

  // Try to get from cache first
  let response = await cache.match(cacheKey);
  
  if (!response) {
    // Fetch from origin
    response = await fetch(request);
    
    if (response.ok) {
      // Clone response and add cache headers
      const cacheResponse = new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers: {
          ...response.headers,
          'Cache-Control': `public, max-age=${CACHE_CONFIG.static}`,
          ...SECURITY_HEADERS
        }
      });
      
      // Store in cache
      await cache.put(cacheKey, cacheResponse.clone());
      return cacheResponse;
    }
  }

  return response || new Response('Not Found', { status: 404 });
}

// Handle SPA routing
async function handleSPARoute(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  
  // Always serve index.html for SPA routes
  const indexRequest = new Request(`${url.origin}/index.html`, {
    headers: request.headers
  });

  const response = await fetch(indexRequest);
  
  if (response.ok) {
    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: {
        ...response.headers,
        'Cache-Control': `public, max-age=${CACHE_CONFIG.html}`,
        ...SECURITY_HEADERS
      }
    });
  }

  return new Response('Not Found', { status: 404 });
}

// Utility functions
function isStaticAsset(pathname: string): boolean {
  const staticExtensions = ['.js', '.css', '.png', '.jpg', '.jpeg', '.gif', '.svg', '.ico', '.woff', '.woff2'];
  return staticExtensions.some(ext => pathname.endsWith(ext));
}

function getOptimalRegion(request: Request): string {
  const cf = request.cf as CloudflareRequestInit['cf'];
  const country = cf?.country || 'US';
  
  // Regional mapping
  const regionMap: Record<string, string> = {
    'US': 'us_east',
    'CA': 'us_east',
    'GB': 'eu_west',
    'DE': 'eu_west',
    'FR': 'eu_west',
    'JP': 'asia_pacific',
    'AU': 'asia_pacific',
    'SG': 'asia_pacific'
  };
  
  return regionMap[country] || 'us_east';
}

// Environment interface
interface Env {
  API_BASE_DEFAULT: string;
  API_BASE_US_EAST: string;
  API_BASE_EU_WEST: string;
  API_BASE_ASIA_PACIFIC: string;
  [key: string]: string;
}

// Export handler
export default {
  fetch: router.handle
};
```

## Interview-Ready AWS/Azure Deployment Summary

**Cloud Deployment Architectures:**
1. **AWS Stack** - S3 + CloudFront + Route53 + Lambda@Edge for global distribution
2. **Azure Stack** - Static Web Apps + Azure CDN + Application Insights for monitoring
3. **Multi-Cloud Strategy** - Terraform-based infrastructure with DNS failover
4. **Edge Computing** - CloudFlare Workers for performance optimization

**Infrastructure as Code:**
- AWS CDK for type-safe infrastructure definitions
- Azure Bicep templates for Azure resource management
- Terraform for multi-cloud orchestration and state management
- GitHub Actions for automated deployment pipelines

**Enterprise Features:**
- SSL/TLS certificate management with auto-renewal
- WAF integration for security protection
- Global CDN distribution with edge caching strategies
- Health checks and automatic failover mechanisms
- Performance monitoring and alerting

**Key Interview Topics:** Cloud deployment strategies, CDN optimization, Infrastructure as Code best practices, multi-cloud architecture, edge computing, automated deployment pipelines, security and compliance in cloud environments, cost optimization strategies.