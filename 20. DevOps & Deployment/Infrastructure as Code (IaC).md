# Infrastructure as Code (IaC) for Frontend Applications

Infrastructure as Code enables reproducible, version-controlled, and automated infrastructure management for frontend applications. This comprehensive guide covers Terraform, AWS CDK, Pulumi, and advanced deployment patterns for scalable frontend infrastructure with enterprise-grade practices.

## Terraform Infrastructure Management

### Multi-Environment Frontend Infrastructure

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
  }

  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "frontend/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Variables
variable "app_name" {
  description = "Application name"
  type        = string
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.app_name))
    error_message = "App name must contain only lowercase letters, numbers, and hyphens."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "domain_name" {
  description = "Primary domain name"
  type        = string
}

variable "enable_cdn" {
  description = "Enable CDN distribution"
  type        = bool
  default     = true
}

variable "enable_waf" {
  description = "Enable Web Application Firewall"
  type        = bool
  default     = false
}

variable "enable_monitoring" {
  description = "Enable comprehensive monitoring"
  type        = bool
  default     = true
}

variable "ssl_certificate_arn" {
  description = "ARN of SSL certificate"
  type        = string
}

variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    ManagedBy = "Terraform"
    Project   = "Frontend Infrastructure"
  }
}

# Local values for computed configurations
locals {
  name_prefix = "${var.app_name}-${var.environment}"
  
  environment_config = {
    dev = {
      instance_class          = "cache.t3.micro"
      cloudfront_price_class = "PriceClass_100"
      backup_retention       = 7
      enable_deletion_protection = false
    }
    staging = {
      instance_class          = "cache.t3.small"
      cloudfront_price_class = "PriceClass_200" 
      backup_retention       = 14
      enable_deletion_protection = false
    }
    prod = {
      instance_class          = "cache.t3.medium"
      cloudfront_price_class = "PriceClass_All"
      backup_retention       = 30
      enable_deletion_protection = true
    }
  }

  current_config = local.environment_config[var.environment]
  
  common_tags = merge(var.tags, {
    Environment = var.environment
    Application = var.app_name
  })
}

# Data sources
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

data "aws_route53_zone" "main" {
  name         = var.domain_name
  private_zone = false
}

# S3 Bucket for static assets
resource "aws_s3_bucket" "assets" {
  bucket = "${local.name_prefix}-assets-${random_string.bucket_suffix.result}"
  
  tags = local.common_tags
}

resource "random_string" "bucket_suffix" {
  length  = 8
  special = false
  upper   = false
}

resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id
  versioning_configuration {
    status = var.environment == "prod" ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_encryption" "assets" {
  bucket = aws_s3_bucket.assets.id

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
      bucket_key_enabled = true
    }
  }
}

resource "aws_s3_bucket_public_access_block" "assets" {
  bucket = aws_s3_bucket.assets.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  rule {
    id     = "cleanup_old_versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = local.current_config.backup_retention
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  rule {
    id     = "transition_to_ia"
    status = var.environment == "prod" ? "Enabled" : "Disabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}

# CloudFront Origin Access Control
resource "aws_cloudfront_origin_access_control" "assets" {
  name                              = "${local.name_prefix}-oac"
  description                       = "OAC for ${local.name_prefix} S3 bucket"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# S3 Bucket Policy for CloudFront
resource "aws_s3_bucket_policy" "assets" {
  bucket = aws_s3_bucket.assets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudFrontServicePrincipal"
        Effect = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.assets.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
          }
        }
      }
    ]
  })
}

# WAF Web ACL
resource "aws_wafv2_web_acl" "main" {
  count = var.enable_waf ? 1 : 0
  
  name  = "${local.name_prefix}-waf"
  scope = "CLOUDFRONT"

  default_action {
    allow {}
  }

  # Rate limiting rule
  rule {
    name     = "RateLimitRule"
    priority = 1

    override_action {
      none {}
    }

    statement {
      rate_based_statement {
        limit              = 10000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "${local.name_prefix}-rate-limit"
      sampled_requests_enabled   = true
    }

    action {
      block {}
    }
  }

  # AWS Managed Rules
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "${local.name_prefix}-common-rules"
      sampled_requests_enabled   = true
    }
  }

  # Known bad inputs rule
  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "${local.name_prefix}-bad-inputs"
      sampled_requests_enabled   = true
    }
  }

  tags = local.common_tags

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${local.name_prefix}-waf"
    sampled_requests_enabled   = true
  }
}

# CloudFront Response Headers Policy
resource "aws_cloudfront_response_headers_policy" "security_headers" {
  name = "${local.name_prefix}-security-headers"

  security_headers_config {
    strict_transport_security {
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }

    content_type_options {
      override = true
    }

    frame_options {
      frame_option = "DENY"
      override     = true
    }

    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }
  }

  custom_headers_config {
    items {
      header   = "Permissions-Policy"
      value    = "camera=(), microphone=(), geolocation=()"
      override = true
    }

    items {
      header   = "X-Robots-Tag"
      value    = var.environment == "prod" ? "index, follow" : "noindex, nofollow"
      override = true
    }
  }
}

# CloudFront Cache Policy
resource "aws_cloudfront_cache_policy" "app_cache" {
  name        = "${local.name_prefix}-app-cache"
  comment     = "Cache policy for ${local.name_prefix} application"
  default_ttl = 86400   # 1 day
  max_ttl     = 31536000 # 1 year
  min_ttl     = 1

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true

    query_strings_config {
      query_string_behavior = "none"
    }

    headers_config {
      header_behavior = "whitelist"
      headers {
        items = ["Accept", "Accept-Language"]
      }
    }

    cookies_config {
      cookie_behavior = "none"
    }
  }
}

resource "aws_cloudfront_cache_policy" "static_cache" {
  name        = "${local.name_prefix}-static-cache"
  comment     = "Cache policy for static assets"
  default_ttl = 2592000  # 30 days
  max_ttl     = 31536000 # 1 year
  min_ttl     = 86400    # 1 day

  parameters_in_cache_key_and_forwarded_to_origin {
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true

    query_strings_config {
      query_string_behavior = "none"
    }

    headers_config {
      header_behavior = "none"
    }

    cookies_config {
      cookie_behavior = "none"
    }
  }
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  aliases             = [var.domain_name]
  comment             = "${local.name_prefix} distribution"
  default_root_object = "index.html"
  enabled             = true
  http_version        = "http2and3"
  is_ipv6_enabled     = true
  price_class         = local.current_config.cloudfront_price_class
  web_acl_id          = var.enable_waf ? aws_wafv2_web_acl.main[0].arn : null

  # Default behavior for SPA
  default_cache_behavior {
    allowed_methods              = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods               = ["GET", "HEAD"]
    target_origin_id             = "S3-${aws_s3_bucket.assets.bucket}"
    compress                     = true
    viewer_protocol_policy       = "redirect-to-https"
    cache_policy_id              = aws_cloudfront_cache_policy.app_cache.id
    response_headers_policy_id   = aws_cloudfront_response_headers_policy.security_headers.id

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.spa_routing.arn
    }
  }

  # Static assets cache behavior
  ordered_cache_behavior {
    path_pattern               = "/static/*"
    allowed_methods            = ["GET", "HEAD", "OPTIONS"]
    cached_methods             = ["GET", "HEAD"]
    target_origin_id           = "S3-${aws_s3_bucket.assets.bucket}"
    compress                   = true
    viewer_protocol_policy     = "redirect-to-https"
    cache_policy_id            = aws_cloudfront_cache_policy.static_cache.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security_headers.id
  }

  # API proxy behavior
  ordered_cache_behavior {
    path_pattern           = "/api/*"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "API-${var.app_name}"
    compress               = true
    viewer_protocol_policy = "https-only"
    cache_policy_id        = aws_cloudfront_cache_policy.CACHING_DISABLED.id

    origin_request_policy_id = aws_cloudfront_origin_request_policy.api_policy.id
  }

  # S3 Origin
  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "S3-${aws_s3_bucket.assets.bucket}"
    origin_access_control_id = aws_cloudfront_origin_access_control.assets.id
  }

  # API Origin (placeholder - replace with actual API gateway)
  origin {
    domain_name = "api.${var.domain_name}"
    origin_id   = "API-${var.app_name}"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Error pages
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
    error_caching_min_ttl = 300
  }

  custom_error_response {
    error_code         = 403
    response_code      = 200
    response_page_path = "/index.html"
    error_caching_min_ttl = 300
  }

  # Geo restrictions
  restrictions {
    geo_restriction {
      restriction_type = var.environment == "prod" ? "none" : "whitelist"
      locations        = var.environment == "prod" ? [] : ["US", "CA", "GB"]
    }
  }

  # SSL Certificate
  viewer_certificate {
    acm_certificate_arn      = var.ssl_certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = local.common_tags
}

# CloudFront Function for SPA routing
resource "aws_cloudfront_function" "spa_routing" {
  name    = "${local.name_prefix}-spa-routing"
  runtime = "cloudfront-js-1.0"
  comment = "SPA routing function for ${local.name_prefix}"
  publish = true
  code    = file("${path.module}/cloudfront-functions/spa-routing.js")
}

# Origin Request Policy for API
resource "aws_cloudfront_origin_request_policy" "api_policy" {
  name    = "${local.name_prefix}-api-policy"
  comment = "Origin request policy for API routes"

  cookies_config {
    cookie_behavior = "all"
  }

  headers_config {
    header_behavior = "whitelist"
    headers {
      items = [
        "Accept",
        "Accept-Language", 
        "Authorization",
        "Content-Type",
        "Origin",
        "Referer",
        "User-Agent",
        "X-Forwarded-For"
      ]
    }
  }

  query_strings_config {
    query_string_behavior = "all"
  }
}

# Route53 DNS Record
resource "aws_route53_record" "main" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}

# Route53 IPv6 Record
resource "aws_route53_record" "main_ipv6" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "AAAA"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}

# CloudWatch Log Group for monitoring
resource "aws_cloudwatch_log_group" "cloudfront_logs" {
  count             = var.enable_monitoring ? 1 : 0
  name              = "/aws/cloudfront/${local.name_prefix}"
  retention_in_days = local.current_config.backup_retention

  tags = local.common_tags
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "high_error_rate" {
  count = var.enable_monitoring ? 1 : 0
  
  alarm_name          = "${local.name_prefix}-high-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "ErrorRate"
  namespace           = "AWS/CloudFront"
  period              = "300"
  statistic           = "Average"
  threshold           = "5"
  alarm_description   = "This metric monitors CloudFront error rate"
  alarm_actions       = [aws_sns_topic.alerts[0].arn]

  dimensions = {
    DistributionId = aws_cloudfront_distribution.main.id
  }

  tags = local.common_tags
}

# SNS Topic for alerts
resource "aws_sns_topic" "alerts" {
  count = var.enable_monitoring ? 1 : 0
  name  = "${local.name_prefix}-alerts"

  tags = local.common_tags
}

# Output values
output "cloudfront_distribution_id" {
  description = "CloudFront Distribution ID"
  value       = aws_cloudfront_distribution.main.id
}

output "cloudfront_domain_name" {
  description = "CloudFront Distribution Domain Name"
  value       = aws_cloudfront_distribution.main.domain_name
}

output "s3_bucket_name" {
  description = "S3 Bucket Name"
  value       = aws_s3_bucket.assets.bucket
}

output "domain_name" {
  description = "Application Domain Name"
  value       = var.domain_name
}

output "waf_web_acl_arn" {
  description = "WAF Web ACL ARN"
  value       = var.enable_waf ? aws_wafv2_web_acl.main[0].arn : null
}
```

### Terraform Modules for Reusability

```hcl
# modules/frontend-stack/variables.tf
variable "app_name" {
  description = "Application name"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "domain_name" {
  description = "Domain name for the application"
  type        = string
}

variable "api_domain" {
  description = "API domain name"
  type        = string
  default     = ""
}

variable "enable_monitoring" {
  description = "Enable monitoring and alerting"
  type        = bool
  default     = true
}

variable "monitoring_config" {
  description = "Monitoring configuration"
  type = object({
    datadog_api_key = optional(string)
    slack_webhook   = optional(string)
    pagerduty_key   = optional(string)
  })
  default = {}
}

variable "performance_config" {
  description = "Performance optimization configuration"
  type = object({
    enable_brotli     = optional(bool, true)
    enable_gzip       = optional(bool, true)
    cache_ttl_static  = optional(number, 31536000)  # 1 year
    cache_ttl_html    = optional(number, 86400)     # 1 day
    cache_ttl_api     = optional(number, 0)         # No cache
  })
  default = {}
}

variable "security_config" {
  description = "Security configuration"
  type = object({
    enable_waf           = optional(bool, false)
    enable_rate_limiting = optional(bool, true)
    allowed_countries    = optional(list(string), [])
    blocked_countries    = optional(list(string), [])
    hsts_max_age        = optional(number, 31536000)
  })
  default = {}
}

# modules/frontend-stack/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

locals {
  name_prefix = "${var.app_name}-${var.environment}"
  
  common_tags = {
    Application = var.app_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    Module      = "frontend-stack"
  }
}

# Call the main infrastructure module
module "infrastructure" {
  source = "../infrastructure"
  
  app_name              = var.app_name
  environment          = var.environment
  domain_name          = var.domain_name
  enable_monitoring    = var.enable_monitoring
  enable_waf           = var.security_config.enable_waf
  
  tags = local.common_tags
}

# Monitoring module
module "monitoring" {
  count  = var.enable_monitoring ? 1 : 0
  source = "../monitoring"
  
  app_name                = var.app_name
  environment            = var.environment
  cloudfront_distribution_id = module.infrastructure.cloudfront_distribution_id
  
  datadog_api_key = var.monitoring_config.datadog_api_key
  slack_webhook   = var.monitoring_config.slack_webhook
  pagerduty_key   = var.monitoring_config.pagerduty_key
  
  tags = local.common_tags
}

# Security module
module "security" {
  source = "../security"
  
  app_name    = var.app_name
  environment = var.environment
  
  enable_waf           = var.security_config.enable_waf
  enable_rate_limiting = var.security_config.enable_rate_limiting
  allowed_countries    = var.security_config.allowed_countries
  blocked_countries    = var.security_config.blocked_countries
  
  cloudfront_distribution_arn = module.infrastructure.cloudfront_distribution_arn
  
  tags = local.common_tags
}

# Output important values
output "application_url" {
  description = "Application URL"
  value       = "https://${var.domain_name}"
}

output "cloudfront_distribution_id" {
  description = "CloudFront Distribution ID"
  value       = module.infrastructure.cloudfront_distribution_id
}

output "s3_bucket_name" {
  description = "S3 Bucket Name"
  value       = module.infrastructure.s3_bucket_name
}

output "monitoring_dashboard_url" {
  description = "Monitoring dashboard URL"
  value       = var.enable_monitoring ? module.monitoring[0].dashboard_url : null
}
```

### Environment-Specific Configurations

```hcl
# environments/dev/terraform.tfvars
app_name     = "my-frontend-app"
environment  = "dev"
domain_name  = "dev.myapp.com"
api_domain   = "api-dev.myapp.com"

enable_monitoring = true

monitoring_config = {
  datadog_api_key = "dev-datadog-key"
  slack_webhook   = "https://hooks.slack.com/dev-webhook"
}

performance_config = {
  cache_ttl_static = 86400    # 1 day for dev
  cache_ttl_html   = 300      # 5 minutes for dev
}

security_config = {
  enable_waf        = false
  allowed_countries = ["US", "CA"]
  hsts_max_age     = 86400
}

# environments/prod/terraform.tfvars
app_name     = "my-frontend-app"
environment  = "prod"
domain_name  = "myapp.com"
api_domain   = "api.myapp.com"

enable_monitoring = true

monitoring_config = {
  datadog_api_key = "prod-datadog-key"
  slack_webhook   = "https://hooks.slack.com/prod-webhook"
  pagerduty_key   = "prod-pagerduty-key"
}

performance_config = {
  enable_brotli    = true
  enable_gzip      = true
  cache_ttl_static = 31536000  # 1 year
  cache_ttl_html   = 86400     # 1 day
}

security_config = {
  enable_waf           = true
  enable_rate_limiting = true
  hsts_max_age        = 31536000
}
```

## AWS CDK TypeScript Implementation

### Advanced CDK Stack with Constructs

```typescript
// infrastructure/lib/frontend-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';
import * as route53 from 'aws-cdk-lib/aws-route53';
import * as targets from 'aws-cdk-lib/aws-route53-targets';
import * as certificatemanager from 'aws-cdk-lib/aws-certificatemanager';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as cr from 'aws-cdk-lib/custom-resources';
import { Construct } from 'constructs';

export interface FrontendStackProps extends cdk.StackProps {
  readonly appName: string;
  readonly environment: 'dev' | 'staging' | 'prod';
  readonly domainName: string;
  readonly certificateArn: string;
  readonly enableWaf?: boolean;
  readonly enableMonitoring?: boolean;
  readonly apiDomain?: string;
  readonly monitoringConfig?: {
    datadogApiKey?: string;
    slackWebhook?: string;
    alertEmail?: string;
  };
}

export class FrontendStack extends cdk.Stack {
  public readonly distribution: cloudfront.Distribution;
  public readonly bucket: s3.Bucket;
  public readonly deploymentRole: iam.Role;

  constructor(scope: Construct, id: string, props: FrontendStackProps) {
    super(scope, id, props);

    const namePrefix = `${props.appName}-${props.environment}`;

    // S3 Bucket for static assets
    this.bucket = new s3.Bucket(this, 'AssetsBucket', {
      bucketName: `${namePrefix}-assets-${this.account}`,
      removalPolicy: props.environment === 'prod' 
        ? cdk.RemovalPolicy.RETAIN 
        : cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: props.environment !== 'prod',
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      publicReadAccess: false,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      
      lifecycleRules: [
        {
          id: 'DeleteOldVersions',
          abortIncompleteMultipartUploadAfter: cdk.Duration.days(7),
          noncurrentVersionExpiration: cdk.Duration.days(30),
        },
        {
          id: 'TransitionToIA',
          enabled: props.environment === 'prod',
          transitions: [
            {
              storageClass: s3.StorageClass.INFREQUENT_ACCESS,
              transitionAfter: cdk.Duration.days(30),
            },
            {
              storageClass: s3.StorageClass.GLACIER,
              transitionAfter: cdk.Duration.days(90),
            },
          ],
        },
      ],
    });

    // WAF Web ACL
    const webAcl = props.enableWaf ? this.createWebACL(namePrefix) : undefined;

    // CloudFront Distribution
    this.distribution = this.createCloudFrontDistribution(
      props,
      namePrefix,
      webAcl
    );

    // Route53 DNS Records
    this.createDNSRecords(props);

    // IAM Role for deployment
    this.deploymentRole = this.createDeploymentRole(namePrefix);

    // Monitoring setup
    if (props.enableMonitoring) {
      this.setupMonitoring(props, namePrefix);
    }

    // Custom resource for initial deployment
    this.createInitialDeployment(namePrefix);

    // Outputs
    this.createOutputs(props);
  }

  private createWebACL(namePrefix: string): wafv2.CfnWebACL {
    return new wafv2.CfnWebACL(this, 'WebACL', {
      name: `${namePrefix}-waf`,
      scope: 'CLOUDFRONT',
      defaultAction: { allow: {} },
      
      rules: [
        {
          name: 'RateLimitRule',
          priority: 1,
          action: { block: {} },
          statement: {
            rateBasedStatement: {
              limit: 10000,
              aggregateKeyType: 'IP',
            },
          },
          visibilityConfig: {
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: `${namePrefix}-rate-limit`,
          },
        },
        {
          name: 'AWSManagedRulesCommonRuleSet',
          priority: 2,
          overrideAction: { none: {} },
          statement: {
            managedRuleGroupStatement: {
              vendorName: 'AWS',
              name: 'AWSManagedRulesCommonRuleSet',
            },
          },
          visibilityConfig: {
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: `${namePrefix}-common-rules`,
          },
        },
      ],
      
      visibilityConfig: {
        sampledRequestsEnabled: true,
        cloudWatchMetricsEnabled: true,
        metricName: `${namePrefix}-waf`,
      },
    });
  }

  private createCloudFrontDistribution(
    props: FrontendStackProps,
    namePrefix: string,
    webAcl?: wafv2.CfnWebACL
  ): cloudfront.Distribution {
    const certificate = certificatemanager.Certificate.fromCertificateArn(
      this,
      'Certificate',
      props.certificateArn
    );

    // Origin Access Identity
    const originAccessIdentity = new cloudfront.OriginAccessIdentity(
      this,
      'OriginAccessIdentity',
      {
        comment: `OAI for ${namePrefix}`,
      }
    );

    this.bucket.grantRead(originAccessIdentity);

    // Response Headers Policy
    const responseHeadersPolicy = new cloudfront.ResponseHeadersPolicy(
      this,
      'ResponseHeadersPolicy',
      {
        responseHeadersPolicyName: `${namePrefix}-security-headers`,
        securityHeadersBehavior: {
          strictTransportSecurity: {
            accessControlMaxAge: cdk.Duration.seconds(31536000),
            includeSubdomains: true,
            preload: true,
            override: true,
          },
          contentTypeOptions: { override: true },
          frameOptions: {
            frameOption: cloudfront.HeadersFrameOption.DENY,
            override: true,
          },
          referrerPolicy: {
            referrerPolicy: cloudfront.HeadersReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN,
            override: true,
          },
        },
        customHeadersBehavior: {
          customHeaders: [
            {
              header: 'Permissions-Policy',
              value: 'camera=(), microphone=(), geolocation=()',
              override: true,
            },
          ],
        },
      }
    );

    // Cache Policies
    const appCachePolicy = new cloudfront.CachePolicy(this, 'AppCachePolicy', {
      cachePolicyName: `${namePrefix}-app-cache`,
      defaultTtl: cdk.Duration.hours(24),
      maxTtl: cdk.Duration.days(365),
      minTtl: cdk.Duration.minutes(1),
      enableAcceptEncodingBrotli: true,
      enableAcceptEncodingGzip: true,
    });

    const staticCachePolicy = new cloudfront.CachePolicy(this, 'StaticCachePolicy', {
      cachePolicyName: `${namePrefix}-static-cache`,
      defaultTtl: cdk.Duration.days(30),
      maxTtl: cdk.Duration.days(365),
      minTtl: cdk.Duration.days(1),
      enableAcceptEncodingBrotli: true,
      enableAcceptEncodingGzip: true,
    });

    // CloudFront Function for SPA routing
    const spaRoutingFunction = new cloudfront.Function(this, 'SPARoutingFunction', {
      functionName: `${namePrefix}-spa-routing`,
      code: cloudfront.FunctionCode.fromFile({
        filePath: 'lambda/spa-routing.js',
      }),
    });

    const distribution = new cloudfront.Distribution(this, 'Distribution', {
      domainNames: [props.domainName],
      certificate,
      defaultRootObject: 'index.html',
      httpVersion: cloudfront.HttpVersion.HTTP2_AND_3,
      priceClass: props.environment === 'prod' 
        ? cloudfront.PriceClass.PRICE_CLASS_ALL
        : cloudfront.PriceClass.PRICE_CLASS_100,
      webAclId: webAcl?.attrArn,

      defaultBehavior: {
        origin: new origins.S3Origin(this.bucket, {
          originAccessIdentity,
        }),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cachePolicy: appCachePolicy,
        responseHeadersPolicy,
        compress: true,
        functionAssociations: [
          {
            function: spaRoutingFunction,
            eventType: cloudfront.FunctionEventType.VIEWER_REQUEST,
          },
        ],
      },

      additionalBehaviors: {
        '/static/*': {
          origin: new origins.S3Origin(this.bucket, {
            originAccessIdentity,
          }),
          viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
          cachePolicy: staticCachePolicy,
          responseHeadersPolicy,
          compress: true,
        },
        ...(props.apiDomain && {
          '/api/*': {
            origin: new origins.HttpOrigin(props.apiDomain),
            viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.HTTPS_ONLY,
            cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
            originRequestPolicy: cloudfront.OriginRequestPolicy.CORS_S3_ORIGIN,
            allowedMethods: cloudfront.AllowedMethods.ALLOW_ALL,
          },
        }),
      },

      errorResponses: [
        {
          httpStatus: 404,
          responseHttpStatus: 200,
          responsePagePath: '/index.html',
          ttl: cdk.Duration.minutes(5),
        },
        {
          httpStatus: 403,
          responseHttpStatus: 200,
          responsePagePath: '/index.html',
          ttl: cdk.Duration.minutes(5),
        },
      ],

      geoRestriction: props.environment === 'prod'
        ? cloudfront.GeoRestriction.allowlist('US', 'CA', 'GB', 'DE', 'FR')
        : undefined,

      enableLogging: true,
      logBucket: new s3.Bucket(this, 'LogsBucket', {
        bucketName: `${namePrefix}-logs-${this.account}`,
        lifecycleRules: [
          {
            expiration: cdk.Duration.days(90),
          },
        ],
      }),
      logFilePrefix: 'cloudfront-logs/',
    });

    return distribution;
  }

  private createDNSRecords(props: FrontendStackProps): void {
    const hostedZone = route53.HostedZone.fromLookup(this, 'HostedZone', {
      domainName: this.extractRootDomain(props.domainName),
    });

    new route53.ARecord(this, 'AliasRecord', {
      zone: hostedZone,
      recordName: props.domainName,
      target: route53.RecordTarget.fromAlias(
        new targets.CloudFrontTarget(this.distribution)
      ),
    });

    new route53.AaaaRecord(this, 'AliasRecordIPv6', {
      zone: hostedZone,
      recordName: props.domainName,
      target: route53.RecordTarget.fromAlias(
        new targets.CloudFrontTarget(this.distribution)
      ),
    });
  }

  private createDeploymentRole(namePrefix: string): iam.Role {
    const role = new iam.Role(this, 'DeploymentRole', {
      roleName: `${namePrefix}-deployment-role`,
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'),
      ],
    });

    // Grant S3 permissions
    this.bucket.grantReadWrite(role);

    // Grant CloudFront invalidation permissions
    role.addToPolicy(
      new iam.PolicyStatement({
        effect: iam.Effect.ALLOW,
        actions: ['cloudfront:CreateInvalidation'],
        resources: [this.distribution.distributionArn],
      })
    );

    return role;
  }

  private setupMonitoring(props: FrontendStackProps, namePrefix: string): void {
    // CloudWatch Log Group
    const logGroup = new logs.LogGroup(this, 'LogGroup', {
      logGroupName: `/aws/cloudfront/${namePrefix}`,
      retention: logs.RetentionDays.ONE_MONTH,
    });

    // SNS Topic for alerts
    const alertTopic = new sns.Topic(this, 'AlertTopic', {
      topicName: `${namePrefix}-alerts`,
    });

    if (props.monitoringConfig?.alertEmail) {
      alertTopic.addSubscription(
        new subscriptions.EmailSubscription(props.monitoringConfig.alertEmail)
      );
    }

    // CloudWatch Alarms
    new cloudwatch.Alarm(this, 'HighErrorRateAlarm', {
      alarmName: `${namePrefix}-high-error-rate`,
      metric: new cloudwatch.Metric({
        namespace: 'AWS/CloudFront',
        metricName: 'ErrorRate',
        dimensionsMap: {
          DistributionId: this.distribution.distributionId,
        },
        statistic: 'Average',
        period: cdk.Duration.minutes(5),
      }),
      threshold: 5,
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
    });

    new cloudwatch.Alarm(this, 'HighLatencyAlarm', {
      alarmName: `${namePrefix}-high-latency`,
      metric: new cloudwatch.Metric({
        namespace: 'AWS/CloudFront',
        metricName: 'OriginLatency',
        dimensionsMap: {
          DistributionId: this.distribution.distributionId,
        },
        statistic: 'Average',
        period: cdk.Duration.minutes(5),
      }),
      threshold: 5000, // 5 seconds
      evaluationPeriods: 3,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
    });

    // Custom metrics and dashboard
    this.createCustomDashboard(namePrefix);
  }

  private createCustomDashboard(namePrefix: string): void {
    const dashboard = new cloudwatch.Dashboard(this, 'Dashboard', {
      dashboardName: `${namePrefix}-frontend-dashboard`,
    });

    // Add widgets for key metrics
    dashboard.addWidgets(
      new cloudwatch.GraphWidget({
        title: 'CloudFront Requests',
        left: [
          new cloudwatch.Metric({
            namespace: 'AWS/CloudFront',
            metricName: 'Requests',
            dimensionsMap: {
              DistributionId: this.distribution.distributionId,
            },
            statistic: 'Sum',
          }),
        ],
      }),
      new cloudwatch.GraphWidget({
        title: 'Error Rate',
        left: [
          new cloudwatch.Metric({
            namespace: 'AWS/CloudFront',
            metricName: 'ErrorRate',
            dimensionsMap: {
              DistributionId: this.distribution.distributionId,
            },
            statistic: 'Average',
          }),
        ],
      }),
    );
  }

  private createInitialDeployment(namePrefix: string): void {
    // Lambda function for initial deployment
    const deploymentFunction = new lambda.Function(this, 'DeploymentFunction', {
      functionName: `${namePrefix}-initial-deployment`,
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: lambda.Code.fromInline(`
        const AWS = require('aws-sdk');
        const s3 = new AWS.S3();
        
        exports.handler = async (event) => {
          const bucketName = '${this.bucket.bucketName}';
          
          // Upload a simple index.html file
          const indexHtml = \`
            <!DOCTYPE html>
            <html>
            <head>
              <title>${namePrefix}</title>
            </head>
            <body>
              <h1>Welcome to ${namePrefix}</h1>
              <p>Your application is ready for deployment!</p>
            </body>
            </html>
          \`;
          
          await s3.putObject({
            Bucket: bucketName,
            Key: 'index.html',
            Body: indexHtml,
            ContentType: 'text/html',
          }).promise();
          
          return { statusCode: 200, body: 'Deployment complete' };
        };
      `),
      role: this.deploymentRole,
      timeout: cdk.Duration.minutes(5),
    });

    // Custom resource to trigger initial deployment
    new cr.AwsCustomResource(this, 'InitialDeployment', {
      onCreate: {
        service: 'Lambda',
        action: 'invoke',
        parameters: {
          FunctionName: deploymentFunction.functionName,
        },
        physicalResourceId: cr.PhysicalResourceId.of('initial-deployment'),
      },
      policy: cr.AwsCustomResourcePolicy.fromSdkCalls({
        resources: cr.AwsCustomResourcePolicy.ANY_RESOURCE,
      }),
    });
  }

  private createOutputs(props: FrontendStackProps): void {
    new cdk.CfnOutput(this, 'ApplicationURL', {
      value: `https://${props.domainName}`,
      description: 'Application URL',
    });

    new cdk.CfnOutput(this, 'CloudFrontDistributionId', {
      value: this.distribution.distributionId,
      description: 'CloudFront Distribution ID',
    });

    new cdk.CfnOutput(this, 'S3BucketName', {
      value: this.bucket.bucketName,
      description: 'S3 Bucket Name',
    });

    new cdk.CfnOutput(this, 'DeploymentRoleArn', {
      value: this.deploymentRole.roleArn,
      description: 'Deployment Role ARN',
    });
  }

  private extractRootDomain(domainName: string): string {
    const parts = domainName.split('.');
    return parts.slice(-2).join('.');
  }
}
```

## Interview-Ready Infrastructure as Code Summary

**Core IaC Concepts:**
1. **Terraform** - HCL-based infrastructure definition with state management, modules, and workspaces
2. **AWS CDK** - TypeScript-based infrastructure with constructs, high-level abstractions, and compile-time safety
3. **Multi-Environment Strategy** - Environment-specific configurations with shared modules and DRY principles
4. **GitOps Integration** - Version-controlled infrastructure with automated deployment pipelines

**Enterprise Patterns:**
- Modular architecture with reusable components and composition patterns
- Environment-specific configuration management with validation
- State management and backend configuration for team collaboration
- Resource tagging and naming conventions for governance

**Advanced Features:**
- Custom resource creation with Lambda-backed resources
- Monitoring and alerting infrastructure as code
- Security controls with WAF, IAM policies, and encryption
- Performance optimization with CDN, caching, and compression

**Key Interview Topics:** Infrastructure as Code principles, Terraform vs CDK trade-offs, state management strategies, module design patterns, environment promotion workflows, drift detection and remediation, cost optimization in IaC, security scanning and compliance automation.