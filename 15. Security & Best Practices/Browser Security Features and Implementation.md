# Browser Security Features and Implementation

Modern browsers provide powerful security features that frontend developers must understand and implement correctly. This comprehensive guide covers Content Security Policy (CSP), Subresource Integrity (SRI), HTTP Strict Transport Security (HSTS), and other browser security mechanisms essential for enterprise-grade web applications.

## Content Security Policy (CSP) Implementation

### Advanced CSP Configuration and Management

```typescript
// Content Security Policy Management System
interface CSPDirective {
  name: string;
  sources: string[];
  allowUnsafe?: boolean;
  reportOnly?: boolean;
  violations?: CSPViolation[];
}

interface CSPPolicy {
  id: string;
  name: string;
  environment: 'development' | 'staging' | 'production';
  directives: Map<string, CSPDirective>;
  reportUri?: string;
  reportTo?: string;
  upgradeInsecureRequests?: boolean;
  blockAllMixedContent?: boolean;
  sandbox?: string[];
  frameAncestors?: string[];
  baseUri?: string[];
  formAction?: string[];
}

interface CSPViolation {
  timestamp: Date;
  blockedUri: string;
  documentUri: string;
  originalPolicy: string;
  referrer: string;
  violatedDirective: string;
  effectiveDirective: string;
  sourceFile?: string;
  lineNumber?: number;
  columnNumber?: number;
  userAgent: string;
  disposition: 'enforce' | 'report';
}

class CSPManager {
  private policies: Map<string, CSPPolicy> = new Map();
  private violations: CSPViolation[] = [];
  private nonceStore: Map<string, string> = new Map();

  constructor() {
    this.initializeDefaultPolicies();
    this.setupViolationReporting();
  }

  // Create comprehensive CSP policy
  createPolicy(config: {
    environment: 'development' | 'staging' | 'production';
    allowInline?: boolean;
    allowEval?: boolean;
    trustedDomains?: string[];
    reportingEnabled?: boolean;
  }): CSPPolicy {
    const policy: CSPPolicy = {
      id: this.generatePolicyId(),
      name: `CSP-${config.environment}`,
      environment: config.environment,
      directives: new Map(),
      reportUri: config.reportingEnabled ? '/api/csp-report' : undefined,
      upgradeInsecureRequests: config.environment === 'production',
      blockAllMixedContent: config.environment === 'production'
    };

    // Default source directive
    this.addDirective(policy, 'default-src', ["'self'"]);

    // Script source directive
    const scriptSources = ["'self'"];
    if (config.allowInline) {
      scriptSources.push("'unsafe-inline'");
    }
    if (config.allowEval) {
      scriptSources.push("'unsafe-eval'");
    }
    if (config.trustedDomains) {
      scriptSources.push(...config.trustedDomains);
    }
    
    // Add nonce support for inline scripts
    const nonce = this.generateNonce();
    scriptSources.push(`'nonce-${nonce}'`);
    
    this.addDirective(policy, 'script-src', scriptSources);

    // Style source directive
    const styleSources = ["'self'", "'unsafe-inline'"]; // CSS often requires unsafe-inline
    if (config.trustedDomains) {
      styleSources.push(...config.trustedDomains);
    }
    this.addDirective(policy, 'style-src', styleSources);

    // Image source directive
    this.addDirective(policy, 'img-src', ["'self'", 'data:', 'https:']);

    // Font source directive
    this.addDirective(policy, 'font-src', ["'self'", 'https:', 'data:']);

    // Connect source directive (for AJAX, WebSockets, EventSource)
    const connectSources = ["'self'"];
    if (config.trustedDomains) {
      connectSources.push(...config.trustedDomains);
    }
    this.addDirective(policy, 'connect-src', connectSources);

    // Media source directive
    this.addDirective(policy, 'media-src', ["'self'"]);

    // Object source directive (plugins)
    this.addDirective(policy, 'object-src', ["'none'"]);

    // Child source directive (frames, workers)
    this.addDirective(policy, 'child-src', ["'self'"]);

    // Frame ancestors directive (embedding protection)
    this.addDirective(policy, 'frame-ancestors', ["'self'"]);

    // Form action directive
    this.addDirective(policy, 'form-action', ["'self'"]);

    // Base URI directive
    this.addDirective(policy, 'base-uri', ["'self'"]);

    // Worker source directive
    this.addDirective(policy, 'worker-src', ["'self'"]);

    // Manifest source directive
    this.addDirective(policy, 'manifest-src', ["'self'"]);

    // Environment-specific adjustments
    if (config.environment === 'development') {
      this.adjustForDevelopment(policy);
    } else if (config.environment === 'production') {
      this.adjustForProduction(policy);
    }

    this.policies.set(policy.id, policy);
    return policy;
  }

  // Generate CSP header string
  generateCSPHeader(policyId: string, reportOnly: boolean = false): string {
    const policy = this.policies.get(policyId);
    if (!policy) {
      throw new Error(`Policy ${policyId} not found`);
    }

    const headerName = reportOnly ? 
      'Content-Security-Policy-Report-Only' : 
      'Content-Security-Policy';

    const directives: string[] = [];

    policy.directives.forEach((directive, name) => {
      const sources = directive.sources.join(' ');
      directives.push(`${name} ${sources}`);
    });

    if (policy.reportUri) {
      directives.push(`report-uri ${policy.reportUri}`);
    }

    if (policy.reportTo) {
      directives.push(`report-to ${policy.reportTo}`);
    }

    if (policy.upgradeInsecureRequests) {
      directives.push('upgrade-insecure-requests');
    }

    if (policy.blockAllMixedContent) {
      directives.push('block-all-mixed-content');
    }

    if (policy.sandbox && policy.sandbox.length > 0) {
      directives.push(`sandbox ${policy.sandbox.join(' ')}`);
    }

    return directives.join('; ');
  }

  // Nonce generation for inline scripts
  generateNonce(): string {
    const nonce = btoa(String.fromCharCode(...crypto.getRandomValues(new Uint8Array(16))));
    const requestId = Math.random().toString(36).substr(2, 9);
    this.nonceStore.set(requestId, nonce);
    return nonce;
  }

  // Get nonce for current request
  getCurrentNonce(requestId: string): string | null {
    return this.nonceStore.get(requestId) || null;
  }

  // Process CSP violation reports
  processViolationReport(report: any): void {
    const violation: CSPViolation = {
      timestamp: new Date(),
      blockedUri: report['blocked-uri'] || '',
      documentUri: report['document-uri'] || '',
      originalPolicy: report['original-policy'] || '',
      referrer: report.referrer || '',
      violatedDirective: report['violated-directive'] || '',
      effectiveDirective: report['effective-directive'] || '',
      sourceFile: report['source-file'],
      lineNumber: report['line-number'],
      columnNumber: report['column-number'],
      userAgent: report['user-agent'] || '',
      disposition: report.disposition || 'enforce'
    };

    this.violations.push(violation);
    this.analyzeViolation(violation);
    this.updatePolicyBasedOnViolations(violation);
  }

  // CSP violation analysis
  private analyzeViolation(violation: CSPViolation): void {
    // Determine if violation is legitimate or attack
    const isLegitimate = this.isLegitimateViolation(violation);
    
    if (!isLegitimate) {
      this.flagSecurityIncident(violation);
    } else {
      this.suggestPolicyUpdate(violation);
    }
  }

  private isLegitimateViolation(violation: CSPViolation): boolean {
    // Check if violation might be from legitimate sources
    const legitimateSources = [
      'chrome-extension://',
      'moz-extension://',
      'safari-extension://',
      'about:blank'
    ];

    return legitimateSources.some(source => 
      violation.blockedUri.startsWith(source)
    );
  }

  private flagSecurityIncident(violation: CSPViolation): void {
    console.warn('Potential security incident detected:', violation);
    
    // In production, this would integrate with security monitoring
    this.sendSecurityAlert({
      type: 'csp_violation',
      severity: 'medium',
      violation,
      timestamp: new Date()
    });
  }

  private suggestPolicyUpdate(violation: CSPViolation): void {
    console.info('CSP policy update suggested:', violation);
    
    // Analyze patterns and suggest policy improvements
    const suggestion = this.generatePolicySuggestion(violation);
    this.logPolicySuggestion(suggestion);
  }

  private generatePolicySuggestion(violation: CSPViolation): PolicySuggestion {
    return {
      directive: violation.violatedDirective,
      currentSources: [],
      suggestedSources: [violation.blockedUri],
      reason: 'Based on violation analysis',
      confidence: 'medium'
    };
  }

  // CSP testing and validation
  async testPolicy(policyId: string, testUrls: string[]): Promise<CSPTestResults> {
    const policy = this.policies.get(policyId);
    if (!policy) {
      throw new Error(`Policy ${policyId} not found`);
    }

    const results: CSPTestResults = {
      policyId,
      testDate: new Date(),
      testsPerformed: testUrls.length,
      violations: [],
      recommendations: []
    };

    for (const url of testUrls) {
      const testResult = await this.testPolicyAgainstUrl(policy, url);
      results.violations.push(...testResult.violations);
    }

    results.recommendations = this.generateRecommendations(results.violations);
    return results;
  }

  private async testPolicyAgainstUrl(policy: CSPPolicy, url: string): Promise<{ violations: CSPViolation[] }> {
    // Implementation would test policy against specific URL
    // This is a simplified version
    return { violations: [] };
  }

  private generateRecommendations(violations: CSPViolation[]): string[] {
    const recommendations: string[] = [];
    
    if (violations.length === 0) {
      recommendations.push('CSP policy appears to be working correctly');
    } else {
      recommendations.push(`Found ${violations.length} violations that may need attention`);
      recommendations.push('Consider using report-only mode initially');
      recommendations.push('Review and whitelist legitimate sources');
    }

    return recommendations;
  }

  // Helper methods
  private initializeDefaultPolicies(): void {
    // Create default policies for different environments
    this.createPolicy({ environment: 'development', allowInline: true, allowEval: true });
    this.createPolicy({ environment: 'production', allowInline: false, allowEval: false, reportingEnabled: true });
  }

  private setupViolationReporting(): void {
    // Set up violation reporting endpoint
    if (typeof window !== 'undefined') {
      document.addEventListener('securitypolicyviolation', (event) => {
        this.processViolationReport({
          'blocked-uri': event.blockedURI,
          'document-uri': event.documentURI,
          'original-policy': event.originalPolicy,
          'referrer': event.referrer,
          'violated-directive': event.violatedDirective,
          'effective-directive': event.effectiveDirective,
          'source-file': event.sourceFile,
          'line-number': event.lineNumber,
          'column-number': event.columnNumber,
          'disposition': event.disposition
        });
      });
    }
  }

  private addDirective(policy: CSPPolicy, name: string, sources: string[]): void {
    policy.directives.set(name, {
      name,
      sources,
      violations: []
    });
  }

  private adjustForDevelopment(policy: CSPPolicy): void {
    // Add development-specific sources
    const devSources = ['localhost:*', '127.0.0.1:*', '*.local'];
    
    policy.directives.forEach(directive => {
      if (['script-src', 'connect-src', 'img-src'].includes(directive.name)) {
        directive.sources.push(...devSources);
      }
    });
  }

  private adjustForProduction(policy: CSPPolicy): void {
    // Remove unsafe directives for production
    policy.directives.forEach(directive => {
      directive.sources = directive.sources.filter(source => 
        !source.includes('unsafe-inline') && !source.includes('unsafe-eval')
      );
    });
  }

  private generatePolicyId(): string {
    return `csp_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private updatePolicyBasedOnViolations(violation: CSPViolation): void {
    // Auto-adjust policy based on violation patterns (with caution)
    // This should be done carefully and with human oversight
  }

  private sendSecurityAlert(alert: any): void {
    // Send security alert to monitoring system
    console.error('Security Alert:', alert);
  }

  private logPolicySuggestion(suggestion: PolicySuggestion): void {
    console.log('Policy Suggestion:', suggestion);
  }
}

// React Hook for CSP Management
export const useCSP = (environment: 'development' | 'staging' | 'production') => {
  const [cspManager] = useState(() => new CSPManager());
  const [currentPolicy, setCurrentPolicy] = useState<CSPPolicy | null>(null);
  const [violations, setViolations] = useState<CSPViolation[]>([]);

  useEffect(() => {
    const policy = cspManager.createPolicy({ 
      environment,
      allowInline: environment === 'development',
      allowEval: environment === 'development',
      reportingEnabled: environment !== 'development'
    });
    
    setCurrentPolicy(policy);
  }, [environment, cspManager]);

  const generateNonce = useCallback(() => {
    return cspManager.generateNonce();
  }, [cspManager]);

  const getCSPHeader = useCallback((reportOnly: boolean = false) => {
    if (!currentPolicy) return '';
    return cspManager.generateCSPHeader(currentPolicy.id, reportOnly);
  }, [currentPolicy, cspManager]);

  return {
    currentPolicy,
    violations,
    generateNonce,
    getCSPHeader,
    processViolation: cspManager.processViolationReport.bind(cspManager)
  };
};

// Supporting interfaces
interface PolicySuggestion {
  directive: string;
  currentSources: string[];
  suggestedSources: string[];
  reason: string;
  confidence: 'low' | 'medium' | 'high';
}

interface CSPTestResults {
  policyId: string;
  testDate: Date;
  testsPerformed: number;
  violations: CSPViolation[];
  recommendations: string[];
}
```

## Subresource Integrity (SRI) Implementation

### SRI Hash Generation and Validation

```typescript
// Subresource Integrity Management
interface SRIResource {
  url: string;
  type: 'script' | 'stylesheet' | 'module';
  integrity: string;
  crossorigin?: 'anonymous' | 'use-credentials';
  fallback?: string;
  lastValidated: Date;
  hashAlgorithm: 'sha256' | 'sha384' | 'sha512';
}

interface SRIPolicy {
  enforceMode: boolean;
  requiredAlgorithms: string[];
  allowedSources: string[];
  fallbackStrategy: 'block' | 'fallback' | 'report';
}

class SRIManager {
  private resources: Map<string, SRIResource> = new Map();
  private policy: SRIPolicy;
  private hashCache: Map<string, string> = new Map();

  constructor(policy: SRIPolicy) {
    this.policy = policy;
  }

  // Generate SRI hash for resource
  async generateHash(
    content: string | ArrayBuffer,
    algorithm: 'sha256' | 'sha384' | 'sha512' = 'sha384'
  ): Promise<string> {
    let data: ArrayBuffer;
    
    if (typeof content === 'string') {
      data = new TextEncoder().encode(content);
    } else {
      data = content;
    }

    const hashBuffer = await crypto.subtle.digest(algorithm.toUpperCase(), data);
    const hashBase64 = btoa(String.fromCharCode(...new Uint8Array(hashBuffer)));
    
    return `${algorithm}-${hashBase64}`;
  }

  // Register resource with SRI
  async registerResource(
    url: string,
    type: 'script' | 'stylesheet' | 'module',
    options: {
      fetchAndHash?: boolean;
      crossorigin?: 'anonymous' | 'use-credentials';
      fallbackUrl?: string;
      algorithm?: 'sha256' | 'sha384' | 'sha512';
    } = {}
  ): Promise<SRIResource> {
    const algorithm = options.algorithm || 'sha384';
    let integrity = '';

    if (options.fetchAndHash) {
      integrity = await this.fetchAndGenerateHash(url, algorithm);
    }

    const resource: SRIResource = {
      url,
      type,
      integrity,
      crossorigin: options.crossorigin,
      fallback: options.fallbackUrl,
      lastValidated: new Date(),
      hashAlgorithm: algorithm
    };

    this.resources.set(url, resource);
    return resource;
  }

  // Fetch resource and generate hash
  private async fetchAndGenerateHash(
    url: string,
    algorithm: 'sha256' | 'sha384' | 'sha512'
  ): Promise<string> {
    // Check cache first
    const cacheKey = `${url}:${algorithm}`;
    if (this.hashCache.has(cacheKey)) {
      return this.hashCache.get(cacheKey)!;
    }

    try {
      const response = await fetch(url);
      if (!response.ok) {
        throw new Error(`Failed to fetch resource: ${response.status}`);
      }

      const content = await response.arrayBuffer();
      const hash = await this.generateHash(content, algorithm);
      
      // Cache the hash
      this.hashCache.set(cacheKey, hash);
      
      return hash;
    } catch (error) {
      console.error(`Failed to generate hash for ${url}:`, error);
      throw error;
    }
  }

  // Generate script tag with SRI
  generateScriptTag(url: string, options: {
    async?: boolean;
    defer?: boolean;
    module?: boolean;
    nonce?: string;
  } = {}): string {
    const resource = this.resources.get(url);
    if (!resource || resource.type !== 'script') {
      throw new Error(`Script resource ${url} not registered or invalid type`);
    }

    const attributes: string[] = [`src="${url}"`];
    
    if (resource.integrity) {
      attributes.push(`integrity="${resource.integrity}"`);
    }
    
    if (resource.crossorigin) {
      attributes.push(`crossorigin="${resource.crossorigin}"`);
    }
    
    if (options.async) {
      attributes.push('async');
    }
    
    if (options.defer) {
      attributes.push('defer');
    }
    
    if (options.module) {
      attributes.push('type="module"');
    }
    
    if (options.nonce) {
      attributes.push(`nonce="${options.nonce}"`);
    }

    return `<script ${attributes.join(' ')}></script>`;
  }

  // Generate link tag for stylesheets with SRI
  generateLinkTag(url: string, options: {
    media?: string;
    title?: string;
  } = {}): string {
    const resource = this.resources.get(url);
    if (!resource || resource.type !== 'stylesheet') {
      throw new Error(`Stylesheet resource ${url} not registered or invalid type`);
    }

    const attributes: string[] = [
      'rel="stylesheet"',
      `href="${url}"`
    ];
    
    if (resource.integrity) {
      attributes.push(`integrity="${resource.integrity}"`);
    }
    
    if (resource.crossorigin) {
      attributes.push(`crossorigin="${resource.crossorigin}"`);
    }
    
    if (options.media) {
      attributes.push(`media="${options.media}"`);
    }
    
    if (options.title) {
      attributes.push(`title="${options.title}"`);
    }

    return `<link ${attributes.join(' ')}>`;
  }

  // Validate resource integrity
  async validateResource(url: string): Promise<ValidationResult> {
    const resource = this.resources.get(url);
    if (!resource) {
      return {
        valid: false,
        reason: 'Resource not registered',
        resource: null
      };
    }

    try {
      const currentHash = await this.fetchAndGenerateHash(url, resource.hashAlgorithm);
      const valid = currentHash === resource.integrity;
      
      if (!valid) {
        this.handleIntegrityFailure(resource, currentHash);
      }

      return {
        valid,
        reason: valid ? 'Hash matches' : 'Hash mismatch',
        resource,
        currentHash,
        expectedHash: resource.integrity
      };
    } catch (error) {
      return {
        valid: false,
        reason: `Validation failed: ${error}`,
        resource
      };
    }
  }

  // Handle integrity check failure
  private handleIntegrityFailure(resource: SRIResource, actualHash: string): void {
    console.error(`SRI failure for ${resource.url}:`, {
      expected: resource.integrity,
      actual: actualHash
    });

    // Log security incident
    this.logSecurityIncident({
      type: 'sri_failure',
      resource: resource.url,
      expectedHash: resource.integrity,
      actualHash,
      timestamp: new Date()
    });

    // Apply fallback strategy
    switch (this.policy.fallbackStrategy) {
      case 'block':
        this.blockResource(resource);
        break;
      case 'fallback':
        this.loadFallbackResource(resource);
        break;
      case 'report':
        this.reportIntegrityFailure(resource);
        break;
    }
  }

  private blockResource(resource: SRIResource): void {
    console.warn(`Blocking resource due to SRI failure: ${resource.url}`);
  }

  private loadFallbackResource(resource: SRIResource): void {
    if (resource.fallback) {
      console.warn(`Loading fallback resource: ${resource.fallback}`);
      // Implementation would load fallback resource
    }
  }

  private reportIntegrityFailure(resource: SRIResource): void {
    // Report to security monitoring system
    console.warn(`Reporting SRI failure for: ${resource.url}`);
  }

  // Batch validation of all resources
  async validateAllResources(): Promise<ValidationReport> {
    const results: ValidationResult[] = [];
    const urls = Array.from(this.resources.keys());

    for (const url of urls) {
      const result = await this.validateResource(url);
      results.push(result);
    }

    const failed = results.filter(r => !r.valid);
    
    return {
      totalResources: results.length,
      validResources: results.length - failed.length,
      failedResources: failed.length,
      results,
      summary: failed.length === 0 ? 'All resources valid' : `${failed.length} resources failed validation`
    };
  }

  // Update resource hash
  async updateResourceHash(url: string): Promise<void> {
    const resource = this.resources.get(url);
    if (!resource) {
      throw new Error(`Resource ${url} not found`);
    }

    const newHash = await this.fetchAndGenerateHash(url, resource.hashAlgorithm);
    resource.integrity = newHash;
    resource.lastValidated = new Date();

    console.log(`Updated hash for ${url}: ${newHash}`);
  }

  private logSecurityIncident(incident: any): void {
    // Log to security monitoring system
    console.error('Security Incident:', incident);
  }
}

// React Hook for SRI Management
export const useSRI = () => {
  const [sriManager] = useState(() => new SRIManager({
    enforceMode: true,
    requiredAlgorithms: ['sha384'],
    allowedSources: ['self', 'https://cdn.jsdelivr.net', 'https://unpkg.com'],
    fallbackStrategy: 'report'
  }));

  const registerScript = useCallback(async (url: string, options: {
    fetchAndHash?: boolean;
    crossorigin?: 'anonymous' | 'use-credentials';
    fallbackUrl?: string;
  } = {}) => {
    return await sriManager.registerResource(url, 'script', options);
  }, [sriManager]);

  const registerStylesheet = useCallback(async (url: string, options: {
    fetchAndHash?: boolean;
    crossorigin?: 'anonymous' | 'use-credentials';
    fallbackUrl?: string;
  } = {}) => {
    return await sriManager.registerResource(url, 'stylesheet', options);
  }, [sriManager]);

  const generateScriptTag = useCallback((url: string, options?: any) => {
    return sriManager.generateScriptTag(url, options);
  }, [sriManager]);

  const generateLinkTag = useCallback((url: string, options?: any) => {
    return sriManager.generateLinkTag(url, options);
  }, [sriManager]);

  const validateAll = useCallback(async () => {
    return await sriManager.validateAllResources();
  }, [sriManager]);

  return {
    registerScript,
    registerStylesheet,
    generateScriptTag,
    generateLinkTag,
    validateAll,
    generateHash: sriManager.generateHash.bind(sriManager)
  };
};

// Supporting interfaces
interface ValidationResult {
  valid: boolean;
  reason: string;
  resource: SRIResource | null;
  currentHash?: string;
  expectedHash?: string;
}

interface ValidationReport {
  totalResources: number;
  validResources: number;
  failedResources: number;
  results: ValidationResult[];
  summary: string;
}
```

## HTTP Strict Transport Security (HSTS) and Other Security Headers

### Comprehensive Security Headers Management

```typescript
// Security Headers Management System
interface SecurityHeaders {
  hsts?: HSTSConfig;
  csp?: string;
  xFrameOptions?: 'DENY' | 'SAMEORIGIN' | string;
  xContentTypeOptions?: 'nosniff';
  xXSSProtection?: '1; mode=block' | '0';
  referrerPolicy?: ReferrerPolicyValue;
  featurePolicy?: FeaturePolicyConfig;
  permissionsPolicy?: PermissionsPolicyConfig;
  expectCT?: ExpectCTConfig;
  crossOriginEmbedderPolicy?: 'require-corp' | 'unsafe-none';
  crossOriginOpenerPolicy?: 'same-origin' | 'same-origin-allow-popups' | 'unsafe-none';
  crossOriginResourcePolicy?: 'same-site' | 'same-origin' | 'cross-origin';
}

interface HSTSConfig {
  maxAge: number; // seconds
  includeSubDomains?: boolean;
  preload?: boolean;
}

interface ExpectCTConfig {
  enforce?: boolean;
  maxAge: number;
  reportUri?: string;
}

interface FeaturePolicyConfig {
  accelerometer?: FeaturePolicyValue;
  ambient_light_sensor?: FeaturePolicyValue;
  autoplay?: FeaturePolicyValue;
  battery?: FeaturePolicyValue;
  camera?: FeaturePolicyValue;
  clipboard_read?: FeaturePolicyValue;
  clipboard_write?: FeaturePolicyValue;
  display_capture?: FeaturePolicyValue;
  document_domain?: FeaturePolicyValue;
  encrypted_media?: FeaturePolicyValue;
  execution_while_not_rendered?: FeaturePolicyValue;
  execution_while_out_of_viewport?: FeaturePolicyValue;
  fullscreen?: FeaturePolicyValue;
  geolocation?: FeaturePolicyValue;
  gyroscope?: FeaturePolicyValue;
  magnetometer?: FeaturePolicyValue;
  microphone?: FeaturePolicyValue;
  midi?: FeaturePolicyValue;
  navigation_override?: FeaturePolicyValue;
  payment?: FeaturePolicyValue;
  picture_in_picture?: FeaturePolicyValue;
  publickey_credentials_get?: FeaturePolicyValue;
  speaker_selection?: FeaturePolicyValue;
  sync_xhr?: FeaturePolicyValue;
  usb?: FeaturePolicyValue;
  wake_lock?: FeaturePolicyValue;
  web_share?: FeaturePolicyValue;
}

interface PermissionsPolicyConfig {
  accelerometer?: PermissionsPolicyValue;
  'ambient-light-sensor'?: PermissionsPolicyValue;
  autoplay?: PermissionsPolicyValue;
  battery?: PermissionsPolicyValue;
  camera?: PermissionsPolicyValue;
  'clipboard-read'?: PermissionsPolicyValue;
  'clipboard-write'?: PermissionsPolicyValue;
  'display-capture'?: PermissionsPolicyValue;
  'document-domain'?: PermissionsPolicyValue;
  'encrypted-media'?: PermissionsPolicyValue;
  'execution-while-not-rendered'?: PermissionsPolicyValue;
  'execution-while-out-of-viewport'?: PermissionsPolicyValue;
  fullscreen?: PermissionsPolicyValue;
  geolocation?: PermissionsPolicyValue;
  gyroscope?: PermissionsPolicyValue;
  magnetometer?: PermissionsPolicyValue;
  microphone?: PermissionsPolicyValue;
  midi?: PermissionsPolicyValue;
  'navigation-override'?: PermissionsPolicyValue;
  payment?: PermissionsPolicyValue;
  'picture-in-picture'?: PermissionsPolicyValue;
  'publickey-credentials-get'?: PermissionsPolicyValue;
  'speaker-selection'?: PermissionsPolicyValue;
  'sync-xhr'?: PermissionsPolicyValue;
  usb?: PermissionsPolicyValue;
  'wake-lock'?: PermissionsPolicyValue;
  'web-share'?: PermissionsPolicyValue;
}

type FeaturePolicyValue = '*' | 'self' | 'none' | string[];
type PermissionsPolicyValue = '*' | 'self' | 'none' | string[];
type ReferrerPolicyValue = 
  | 'no-referrer'
  | 'no-referrer-when-downgrade'
  | 'origin'
  | 'origin-when-cross-origin'
  | 'same-origin'
  | 'strict-origin'
  | 'strict-origin-when-cross-origin'
  | 'unsafe-url';

class SecurityHeadersManager {
  private defaultHeaders: SecurityHeaders;
  private environment: 'development' | 'staging' | 'production';

  constructor(environment: 'development' | 'staging' | 'production') {
    this.environment = environment;
    this.defaultHeaders = this.createDefaultHeaders();
  }

  // Create environment-appropriate default headers
  private createDefaultHeaders(): SecurityHeaders {
    const baseHeaders: SecurityHeaders = {
      xContentTypeOptions: 'nosniff',
      xFrameOptions: 'DENY',
      xXSSProtection: '1; mode=block',
      referrerPolicy: 'strict-origin-when-cross-origin',
      crossOriginOpenerPolicy: 'same-origin',
      crossOriginResourcePolicy: 'same-origin'
    };

    if (this.environment === 'production') {
      baseHeaders.hsts = {
        maxAge: 31536000, // 1 year
        includeSubDomains: true,
        preload: true
      };

      baseHeaders.expectCT = {
        enforce: true,
        maxAge: 86400, // 24 hours
        reportUri: '/api/expect-ct-report'
      };

      baseHeaders.crossOriginEmbedderPolicy = 'require-corp';
    }

    // Feature Policy / Permissions Policy
    baseHeaders.permissionsPolicy = {
      camera: 'none',
      microphone: 'none',
      geolocation: 'self',
      payment: 'self',
      usb: 'none',
      'ambient-light-sensor': 'none',
      accelerometer: 'none',
      gyroscope: 'none',
      magnetometer: 'none'
    };

    return baseHeaders;
  }

  // Generate all security headers
  generateHeaders(customHeaders: Partial<SecurityHeaders> = {}): Record<string, string> {
    const headers = { ...this.defaultHeaders, ...customHeaders };
    const headerMap: Record<string, string> = {};

    // HSTS
    if (headers.hsts) {
      headerMap['Strict-Transport-Security'] = this.formatHSTS(headers.hsts);
    }

    // CSP
    if (headers.csp) {
      headerMap['Content-Security-Policy'] = headers.csp;
    }

    // X-Frame-Options
    if (headers.xFrameOptions) {
      headerMap['X-Frame-Options'] = headers.xFrameOptions;
    }

    // X-Content-Type-Options
    if (headers.xContentTypeOptions) {
      headerMap['X-Content-Type-Options'] = headers.xContentTypeOptions;
    }

    // X-XSS-Protection
    if (headers.xXSSProtection) {
      headerMap['X-XSS-Protection'] = headers.xXSSProtection;
    }

    // Referrer Policy
    if (headers.referrerPolicy) {
      headerMap['Referrer-Policy'] = headers.referrerPolicy;
    }

    // Expect-CT
    if (headers.expectCT) {
      headerMap['Expect-CT'] = this.formatExpectCT(headers.expectCT);
    }

    // Cross-Origin-Embedder-Policy
    if (headers.crossOriginEmbedderPolicy) {
      headerMap['Cross-Origin-Embedder-Policy'] = headers.crossOriginEmbedderPolicy;
    }

    // Cross-Origin-Opener-Policy
    if (headers.crossOriginOpenerPolicy) {
      headerMap['Cross-Origin-Opener-Policy'] = headers.crossOriginOpenerPolicy;
    }

    // Cross-Origin-Resource-Policy
    if (headers.crossOriginResourcePolicy) {
      headerMap['Cross-Origin-Resource-Policy'] = headers.crossOriginResourcePolicy;
    }

    // Permissions Policy
    if (headers.permissionsPolicy) {
      headerMap['Permissions-Policy'] = this.formatPermissionsPolicy(headers.permissionsPolicy);
    }

    return headerMap;
  }

  // Format HSTS header
  private formatHSTS(config: HSTSConfig): string {
    let directive = `max-age=${config.maxAge}`;
    
    if (config.includeSubDomains) {
      directive += '; includeSubDomains';
    }
    
    if (config.preload) {
      directive += '; preload';
    }
    
    return directive;
  }

  // Format Expect-CT header
  private formatExpectCT(config: ExpectCTConfig): string {
    let directive = `max-age=${config.maxAge}`;
    
    if (config.enforce) {
      directive += ', enforce';
    }
    
    if (config.reportUri) {
      directive += `, report-uri="${config.reportUri}"`;
    }
    
    return directive;
  }

  // Format Permissions Policy header
  private formatPermissionsPolicy(config: PermissionsPolicyConfig): string {
    const directives: string[] = [];
    
    Object.entries(config).forEach(([feature, value]) => {
      if (value === 'none') {
        directives.push(`${feature}=()`);
      } else if (value === 'self') {
        directives.push(`${feature}=(self)`);
      } else if (value === '*') {
        directives.push(`${feature}=*`);
      } else if (Array.isArray(value)) {
        const sources = value.map(source => 
          source === 'self' ? 'self' : `"${source}"`
        ).join(' ');
        directives.push(`${feature}=(${sources})`);
      }
    });
    
    return directives.join(', ');
  }

  // HSTS preload submission helper
  generateHSTSPreloadSubmission(domain: string): HSTSPreloadSubmission {
    return {
      domain,
      requirements: [
        'Serve a valid certificate',
        'Redirect from HTTP to HTTPS on the same host',
        'Serve all subdomains over HTTPS',
        'Serve an HSTS header on the base domain for HTTPS requests',
        'HSTS header must have max-age of at least 31536000 seconds (1 year)',
        'HSTS header must include includeSubDomains directive',
        'HSTS header must include preload directive'
      ],
      submissionUrl: 'https://hstspreload.org/',
      verificationSteps: [
        'Check HSTS header configuration',
        'Verify HTTPS redirect',
        'Test subdomain HTTPS availability',
        'Submit domain to HSTS preload list'
      ]
    };
  }

  // Security headers analysis
  analyzeHeaders(headers: Record<string, string>): SecurityAnalysis {
    const analysis: SecurityAnalysis = {
      score: 0,
      maxScore: 100,
      findings: [],
      recommendations: []
    };

    // Check HSTS
    if (headers['Strict-Transport-Security']) {
      analysis.score += 15;
      if (headers['Strict-Transport-Security'].includes('includeSubDomains')) {
        analysis.score += 5;
      }
      if (headers['Strict-Transport-Security'].includes('preload')) {
        analysis.score += 5;
      }
    } else {
      analysis.findings.push({
        severity: 'high',
        header: 'Strict-Transport-Security',
        issue: 'Missing HSTS header',
        recommendation: 'Add HSTS header with appropriate max-age'
      });
    }

    // Check CSP
    if (headers['Content-Security-Policy']) {
      analysis.score += 20;
    } else {
      analysis.findings.push({
        severity: 'high',
        header: 'Content-Security-Policy',
        issue: 'Missing CSP header',
        recommendation: 'Implement Content Security Policy'
      });
    }

    // Check X-Frame-Options
    if (headers['X-Frame-Options']) {
      analysis.score += 10;
    } else {
      analysis.findings.push({
        severity: 'medium',
        header: 'X-Frame-Options',
        issue: 'Missing X-Frame-Options header',
        recommendation: 'Add X-Frame-Options header to prevent clickjacking'
      });
    }

    // Check X-Content-Type-Options
    if (headers['X-Content-Type-Options'] === 'nosniff') {
      analysis.score += 10;
    } else {
      analysis.findings.push({
        severity: 'medium',
        header: 'X-Content-Type-Options',
        issue: 'Missing or incorrect X-Content-Type-Options header',
        recommendation: 'Set X-Content-Type-Options to nosniff'
      });
    }

    // Check Referrer Policy
    if (headers['Referrer-Policy']) {
      analysis.score += 10;
    } else {
      analysis.findings.push({
        severity: 'low',
        header: 'Referrer-Policy',
        issue: 'Missing Referrer-Policy header',
        recommendation: 'Add Referrer-Policy header to control referrer information'
      });
    }

    // Check Permissions Policy
    if (headers['Permissions-Policy']) {
      analysis.score += 15;
    } else {
      analysis.findings.push({
        severity: 'medium',
        header: 'Permissions-Policy',
        issue: 'Missing Permissions-Policy header',
        recommendation: 'Add Permissions-Policy to control browser features'
      });
    }

    // Generate overall recommendations
    if (analysis.score < 60) {
      analysis.recommendations.push('Critical: Implement basic security headers immediately');
    } else if (analysis.score < 80) {
      analysis.recommendations.push('Good: Consider implementing additional security headers');
    } else {
      analysis.recommendations.push('Excellent: Security headers are well configured');
    }

    return analysis;
  }
}

// React Hook for Security Headers
export const useSecurityHeaders = (environment: 'development' | 'staging' | 'production') => {
  const [headersManager] = useState(() => new SecurityHeadersManager(environment));

  const generateHeaders = useCallback((customHeaders?: Partial<SecurityHeaders>) => {
    return headersManager.generateHeaders(customHeaders);
  }, [headersManager]);

  const analyzeHeaders = useCallback((headers: Record<string, string>) => {
    return headersManager.analyzeHeaders(headers);
  }, [headersManager]);

  const getHSTSPreloadInfo = useCallback((domain: string) => {
    return headersManager.generateHSTSPreloadSubmission(domain);
  }, [headersManager]);

  return {
    generateHeaders,
    analyzeHeaders,
    getHSTSPreloadInfo
  };
};

// Supporting interfaces
interface HSTSPreloadSubmission {
  domain: string;
  requirements: string[];
  submissionUrl: string;
  verificationSteps: string[];
}

interface SecurityAnalysis {
  score: number;
  maxScore: number;
  findings: SecurityFinding[];
  recommendations: string[];
}

interface SecurityFinding {
  severity: 'low' | 'medium' | 'high' | 'critical';
  header: string;
  issue: string;
  recommendation: string;
}
```

## Interview-Ready Browser Security Summary

**Core Browser Security Features:**
1. **Content Security Policy (CSP)** - Comprehensive XSS prevention with directive management
2. **Subresource Integrity (SRI)** - Resource tampering protection with hash validation
3. **HTTP Strict Transport Security (HSTS)** - Transport layer security enforcement
4. **Security Headers** - Complete header suite for defense in depth

**Advanced Implementation Patterns:**
- Dynamic CSP policy generation with nonce support
- Automated SRI hash generation and validation
- Environment-specific security header configuration
- Comprehensive violation reporting and analysis

**Enterprise Security Management:**
- Centralized security policy management
- Automated compliance checking and reporting
- Security incident detection and response
- Continuous monitoring and policy updates

**Browser Security Best Practices:**
- Multi-layered security approach with defense in depth
- Automated security testing and validation
- Regular security header analysis and optimization
- Integration with security monitoring and alerting systems

**Key Interview Topics:** CSP implementation strategies, SRI hash management, HSTS preload requirements, security header configuration, browser security feature support, violation handling and analysis, security policy automation, compliance and monitoring integration.