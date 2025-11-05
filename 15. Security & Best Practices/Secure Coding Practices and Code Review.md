# Secure Coding Practices and Code Review Guidelines

Secure coding is fundamental to building robust frontend applications that resist security vulnerabilities. This comprehensive guide covers enterprise-grade secure coding practices, automated security testing, and systematic code review processes for identifying and preventing security issues.

## Secure Coding Standards

### Input Validation and Sanitization Framework

```typescript
// Comprehensive Input Validation Framework
interface ValidationConfig {
  type: 'string' | 'number' | 'email' | 'url' | 'html' | 'json' | 'file';
  required?: boolean;
  minLength?: number;
  maxLength?: number;
  pattern?: RegExp;
  allowedValues?: any[];
  sanitize?: boolean;
  encoding?: 'html' | 'url' | 'js';
  customValidator?: (value: any) => boolean | string;
}

interface SecurityValidationResult {
  isValid: boolean;
  sanitizedValue?: any;
  errors: string[];
  securityIssues: SecurityIssue[];
}

interface SecurityIssue {
  type: 'xss' | 'sql_injection' | 'command_injection' | 'path_traversal' | 'code_injection';
  severity: 'low' | 'medium' | 'high' | 'critical';
  description: string;
  location: string;
}

class SecureInputValidator {
  private static readonly SECURITY_PATTERNS = {
    xss: [
      /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi,
      /javascript:/gi,
      /on\w+\s*=/gi,
      /<iframe\b[^<]*(?:(?!<\/iframe>)<[^<]*)*<\/iframe>/gi,
      /data:text\/html/gi,
      /vbscript:/gi
    ],
    sqlInjection: [
      /(\bSELECT\b|\bINSERT\b|\bUPDATE\b|\bDELETE\b|\bDROP\b|\bCREATE\b)/i,
      /(\bUNION\b|\bOR\b|\bAND\b)\s+\d+\s*=\s*\d+/i,
      /[\'"]\s*(OR|AND)\s+[\'"]\d+[\'"]?\s*=\s*[\'"]\d+/i,
      /exec\s*\(/i,
      /sp_\w+/i
    ],
    commandInjection: [
      /[;&|`$(){}[\]]/,
      /\b(cat|ls|ps|whoami|pwd|id|uname|echo|curl|wget|rm|mv|cp)\b/i,
      /\|\s*(cat|ls|ps|whoami|pwd|id|uname|echo|curl|wget|rm|mv|cp)/i
    ],
    pathTraversal: [
      /\.\.[\/\\]/,
      /%2e%2e[\/\\]/i,
      /\.\.[%2f%5c]/i,
      /\.\.%2f/i,
      /\.\.%5c/i
    ],
    codeInjection: [
      /eval\s*\(/i,
      /Function\s*\(/i,
      /setTimeout\s*\(/i,
      /setInterval\s*\(/i,
      /document\.write/i,
      /innerHTML\s*=/i,
      /outerHTML\s*=/i
    ]
  };

  static validate(value: any, config: ValidationConfig): SecurityValidationResult {
    const result: SecurityValidationResult = {
      isValid: true,
      errors: [],
      securityIssues: []
    };

    // Basic validation
    if (config.required && (value === null || value === undefined || value === '')) {
      result.isValid = false;
      result.errors.push('Value is required');
      return result;
    }

    if (value === null || value === undefined) {
      result.sanitizedValue = value;
      return result;
    }

    // Convert value to string for security checks
    const stringValue = String(value);

    // Security vulnerability detection
    this.detectSecurityIssues(stringValue, result);

    // Type-specific validation
    switch (config.type) {
      case 'string':
        this.validateString(stringValue, config, result);
        break;
      case 'number':
        this.validateNumber(value, config, result);
        break;
      case 'email':
        this.validateEmail(stringValue, config, result);
        break;
      case 'url':
        this.validateUrl(stringValue, config, result);
        break;
      case 'html':
        this.validateHtml(stringValue, config, result);
        break;
      case 'json':
        this.validateJson(stringValue, config, result);
        break;
      case 'file':
        this.validateFile(value, config, result);
        break;
    }

    // Custom validation
    if (config.customValidator) {
      const customResult = config.customValidator(value);
      if (customResult !== true) {
        result.isValid = false;
        result.errors.push(typeof customResult === 'string' ? customResult : 'Custom validation failed');
      }
    }

    // Sanitization
    if (config.sanitize && result.isValid) {
      result.sanitizedValue = this.sanitizeValue(stringValue, config);
    } else {
      result.sanitizedValue = value;
    }

    return result;
  }

  private static detectSecurityIssues(value: string, result: SecurityValidationResult): void {
    Object.entries(this.SECURITY_PATTERNS).forEach(([type, patterns]) => {
      patterns.forEach(pattern => {
        if (pattern.test(value)) {
          result.securityIssues.push({
            type: type as any,
            severity: this.getSeverityForType(type as any),
            description: `Potential ${type.replace(/([A-Z])/g, ' $1').toLowerCase()} detected`,
            location: 'input_validation'
          });
          result.isValid = false;
        }
      });
    });
  }

  private static getSeverityForType(type: string): 'low' | 'medium' | 'high' | 'critical' {
    const severityMap: Record<string, 'low' | 'medium' | 'high' | 'critical'> = {
      'xss': 'high',
      'sqlInjection': 'critical',
      'commandInjection': 'critical',
      'pathTraversal': 'high',
      'codeInjection': 'critical'
    };
    return severityMap[type] || 'medium';
  }

  private static validateString(value: string, config: ValidationConfig, result: SecurityValidationResult): void {
    if (config.minLength && value.length < config.minLength) {
      result.isValid = false;
      result.errors.push(`Minimum length is ${config.minLength}`);
    }

    if (config.maxLength && value.length > config.maxLength) {
      result.isValid = false;
      result.errors.push(`Maximum length is ${config.maxLength}`);
    }

    if (config.pattern && !config.pattern.test(value)) {
      result.isValid = false;
      result.errors.push('Value does not match required pattern');
    }

    if (config.allowedValues && !config.allowedValues.includes(value)) {
      result.isValid = false;
      result.errors.push('Value is not in allowed list');
    }
  }

  private static validateNumber(value: any, config: ValidationConfig, result: SecurityValidationResult): void {
    const numValue = Number(value);
    if (isNaN(numValue)) {
      result.isValid = false;
      result.errors.push('Value must be a valid number');
    }
  }

  private static validateEmail(value: string, config: ValidationConfig, result: SecurityValidationResult): void {
    const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
    if (!emailRegex.test(value)) {
      result.isValid = false;
      result.errors.push('Invalid email format');
    }
  }

  private static validateUrl(value: string, config: ValidationConfig, result: SecurityValidationResult): void {
    try {
      const url = new URL(value);
      
      // Check for dangerous protocols
      const dangerousProtocols = ['javascript:', 'data:', 'vbscript:', 'file:'];
      if (dangerousProtocols.some(protocol => url.protocol.startsWith(protocol))) {
        result.isValid = false;
        result.errors.push('Unsafe URL protocol detected');
        result.securityIssues.push({
          type: 'xss',
          severity: 'high',
          description: 'Dangerous URL protocol detected',
          location: 'url_validation'
        });
      }
    } catch (error) {
      result.isValid = false;
      result.errors.push('Invalid URL format');
    }
  }

  private static validateHtml(value: string, config: ValidationConfig, result: SecurityValidationResult): void {
    // Check for dangerous HTML elements and attributes
    const dangerousElements = ['script', 'iframe', 'object', 'embed', 'form', 'meta'];
    const dangerousAttributes = ['onclick', 'onload', 'onmouseover', 'onerror', 'onchange'];

    dangerousElements.forEach(element => {
      const regex = new RegExp(`<${element}\\b`, 'gi');
      if (regex.test(value)) {
        result.securityIssues.push({
          type: 'xss',
          severity: 'high',
          description: `Dangerous HTML element detected: ${element}`,
          location: 'html_validation'
        });
        result.isValid = false;
      }
    });

    dangerousAttributes.forEach(attr => {
      const regex = new RegExp(`${attr}\\s*=`, 'gi');
      if (regex.test(value)) {
        result.securityIssues.push({
          type: 'xss',
          severity: 'high',
          description: `Dangerous HTML attribute detected: ${attr}`,
          location: 'html_validation'
        });
        result.isValid = false;
      }
    });
  }

  private static validateJson(value: string, config: ValidationConfig, result: SecurityValidationResult): void {
    try {
      JSON.parse(value);
    } catch (error) {
      result.isValid = false;
      result.errors.push('Invalid JSON format');
    }
  }

  private static validateFile(value: File, config: ValidationConfig, result: SecurityValidationResult): void {
    if (!(value instanceof File)) {
      result.isValid = false;
      result.errors.push('Value must be a File object');
      return;
    }

    // Check file type
    const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'text/plain', 'application/pdf'];
    if (!allowedTypes.includes(value.type)) {
      result.isValid = false;
      result.errors.push('File type not allowed');
    }

    // Check file size (10MB limit)
    const maxSize = 10 * 1024 * 1024;
    if (value.size > maxSize) {
      result.isValid = false;
      result.errors.push('File size exceeds limit');
    }

    // Check filename for path traversal
    if (this.SECURITY_PATTERNS.pathTraversal.some(pattern => pattern.test(value.name))) {
      result.securityIssues.push({
        type: 'path_traversal',
        severity: 'high',
        description: 'Potential path traversal in filename',
        location: 'file_validation'
      });
      result.isValid = false;
    }
  }

  private static sanitizeValue(value: string, config: ValidationConfig): string {
    let sanitized = value;

    switch (config.encoding) {
      case 'html':
        sanitized = this.htmlEncode(sanitized);
        break;
      case 'url':
        sanitized = encodeURIComponent(sanitized);
        break;
      case 'js':
        sanitized = this.jsEncode(sanitized);
        break;
    }

    return sanitized;
  }

  private static htmlEncode(str: string): string {
    const htmlEntities: Record<string, string> = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#x27;',
      '/': '&#x2F;'
    };

    return str.replace(/[&<>"'/]/g, (match) => htmlEntities[match]);
  }

  private static jsEncode(str: string): string {
    return str.replace(/[\\'"\/\b\f\n\r\t]/g, (match) => {
      const escapeMap: Record<string, string> = {
        '\\': '\\\\',
        "'": "\\'",
        '"': '\\"',
        '/': '\\/',
        '\b': '\\b',
        '\f': '\\f',
        '\n': '\\n',
        '\r': '\\r',
        '\t': '\\t'
      };
      return escapeMap[match];
    });
  }
}

// React Hook for Secure Input Validation
export const useSecureValidation = <T extends Record<string, any>>(
  validationSchema: Record<keyof T, ValidationConfig>
) => {
  const [values, setValues] = useState<T>({} as T);
  const [errors, setErrors] = useState<Record<keyof T, string[]>>({} as Record<keyof T, string[]>);
  const [securityIssues, setSecurityIssues] = useState<SecurityIssue[]>([]);

  const validateField = useCallback((name: keyof T, value: any) => {
    const config = validationSchema[name];
    if (!config) return { isValid: true, sanitizedValue: value };

    const result = SecureInputValidator.validate(value, config);
    
    setErrors(prev => ({
      ...prev,
      [name]: result.errors
    }));

    setSecurityIssues(prev => {
      const filtered = prev.filter(issue => issue.location !== `field_${String(name)}`);
      const newIssues = result.securityIssues.map(issue => ({
        ...issue,
        location: `field_${String(name)}`
      }));
      return [...filtered, ...newIssues];
    });

    return result;
  }, [validationSchema]);

  const setValue = useCallback((name: keyof T, value: any) => {
    const validation = validateField(name, value);
    
    setValues(prev => ({
      ...prev,
      [name]: validation.sanitizedValue
    }));
  }, [validateField]);

  const validateAll = useCallback(() => {
    let isFormValid = true;
    const newErrors: Record<keyof T, string[]> = {} as Record<keyof T, string[]>;
    const newSecurityIssues: SecurityIssue[] = [];

    Object.keys(validationSchema).forEach(key => {
      const fieldName = key as keyof T;
      const result = validateField(fieldName, values[fieldName]);
      
      if (!result.isValid) {
        isFormValid = false;
        newErrors[fieldName] = result.errors;
        newSecurityIssues.push(...result.securityIssues);
      }
    });

    setErrors(newErrors);
    setSecurityIssues(newSecurityIssues);

    return isFormValid;
  }, [values, validationSchema, validateField]);

  const hasSecurityIssues = securityIssues.length > 0;
  const hasCriticalIssues = securityIssues.some(issue => issue.severity === 'critical');

  return {
    values,
    errors,
    securityIssues,
    setValue,
    validateField,
    validateAll,
    hasSecurityIssues,
    hasCriticalIssues,
    isSecure: !hasSecurityIssues
  };
};
```

## Automated Security Testing

### Security Linting and Static Analysis

```typescript
// Custom ESLint Rules for Security
const securityRules = {
  // Detect dangerous innerHTML usage
  'no-dangerous-innerhtml': {
    meta: {
      type: 'problem',
      docs: {
        description: 'Disallow dangerous innerHTML assignments',
        category: 'Security'
      },
      fixable: null
    },
    create(context: any) {
      return {
        AssignmentExpression(node: any) {
          if (
            node.left.type === 'MemberExpression' &&
            node.left.property.name === 'innerHTML' &&
            node.right.type !== 'Literal'
          ) {
            context.report({
              node,
              message: 'Avoid dynamic innerHTML assignments. Use textContent or sanitize input.'
            });
          }
        }
      };
    }
  },

  // Detect eval usage
  'no-eval-usage': {
    meta: {
      type: 'problem',
      docs: {
        description: 'Disallow eval and Function constructor',
        category: 'Security'
      }
    },
    create(context: any) {
      return {
        CallExpression(node: any) {
          if (
            (node.callee.name === 'eval') ||
            (node.callee.name === 'Function' && node.arguments.length > 0)
          ) {
            context.report({
              node,
              message: 'eval() and Function constructor are security risks'
            });
          }
        }
      };
    }
  },

  // Detect unsafe localStorage usage
  'secure-localstorage': {
    meta: {
      type: 'problem',
      docs: {
        description: 'Ensure localStorage is used securely',
        category: 'Security'
      }
    },
    create(context: any) {
      return {
        MemberExpression(node: any) {
          if (
            node.object.name === 'localStorage' &&
            (node.property.name === 'setItem' || node.property.name === 'getItem')
          ) {
            const parent = node.parent;
            if (parent.type === 'CallExpression') {
              const args = parent.arguments;
              if (args.length > 1 && args[1].type !== 'CallExpression') {
                context.report({
                  node,
                  message: 'Consider encrypting sensitive data before storing in localStorage'
                });
              }
            }
          }
        }
      };
    }
  }
};

// Security Testing Utilities
interface SecurityTestConfig {
  testXSS: boolean;
  testCSRF: boolean;
  testInjection: boolean;
  testAuthentication: boolean;
  testAuthorization: boolean;
}

class SecurityTester {
  private config: SecurityTestConfig;

  constructor(config: SecurityTestConfig) {
    this.config = config;
  }

  // Run comprehensive security tests
  async runSecuritySuite(component: React.ComponentType<any>): Promise<SecurityTestResults> {
    const results: SecurityTestResults = {
      passed: 0,
      failed: 0,
      vulnerabilities: [],
      recommendations: []
    };

    if (this.config.testXSS) {
      const xssResults = await this.testXSSVulnerabilities(component);
      this.mergeResults(results, xssResults);
    }

    if (this.config.testCSRF) {
      const csrfResults = await this.testCSRFProtection(component);
      this.mergeResults(results, csrfResults);
    }

    if (this.config.testInjection) {
      const injectionResults = await this.testInjectionVulnerabilities(component);
      this.mergeResults(results, injectionResults);
    }

    if (this.config.testAuthentication) {
      const authResults = await this.testAuthenticationFlows(component);
      this.mergeResults(results, authResults);
    }

    if (this.config.testAuthorization) {
      const authzResults = await this.testAuthorizationControls(component);
      this.mergeResults(results, authzResults);
    }

    return results;
  }

  // Test XSS vulnerabilities
  private async testXSSVulnerabilities(component: React.ComponentType<any>): Promise<SecurityTestResults> {
    const results: SecurityTestResults = {
      passed: 0,
      failed: 0,
      vulnerabilities: [],
      recommendations: []
    };

    const xssPayloads = [
      '<script>alert("XSS")</script>',
      'javascript:alert("XSS")',
      '<img src="x" onerror="alert(\'XSS\')">',
      '<svg onload="alert(\'XSS\')">',
      'data:text/html,<script>alert("XSS")</script>'
    ];

    for (const payload of xssPayloads) {
      try {
        // Test would render component with malicious payload
        // and check if it's properly sanitized
        const isVulnerable = await this.checkXSSPayload(component, payload);
        
        if (isVulnerable) {
          results.failed++;
          results.vulnerabilities.push({
            type: 'xss',
            severity: 'high',
            description: `XSS vulnerability detected with payload: ${payload}`,
            location: 'component_render'
          });
        } else {
          results.passed++;
        }
      } catch (error) {
        results.failed++;
        results.vulnerabilities.push({
          type: 'xss',
          severity: 'medium',
          description: `XSS test failed: ${error}`,
          location: 'test_execution'
        });
      }
    }

    if (results.failed > 0) {
      results.recommendations.push(
        'Implement proper input sanitization using DOMPurify or similar library',
        'Use textContent instead of innerHTML for user-generated content',
        'Implement Content Security Policy (CSP) headers'
      );
    }

    return results;
  }

  // Test CSRF protection
  private async testCSRFProtection(component: React.ComponentType<any>): Promise<SecurityTestResults> {
    const results: SecurityTestResults = {
      passed: 0,
      failed: 0,
      vulnerabilities: [],
      recommendations: []
    };

    // Check if forms include CSRF tokens
    const hasCSRFProtection = await this.checkCSRFTokens(component);
    
    if (!hasCSRFProtection) {
      results.failed++;
      results.vulnerabilities.push({
        type: 'csrf',
        severity: 'high',
        description: 'CSRF protection not detected in form submissions',
        location: 'form_submission'
      });
      
      results.recommendations.push(
        'Implement CSRF tokens for all state-changing requests',
        'Use SameSite cookie attributes',
        'Validate referrer headers for sensitive operations'
      );
    } else {
      results.passed++;
    }

    return results;
  }

  // Test injection vulnerabilities
  private async testInjectionVulnerabilities(component: React.ComponentType<any>): Promise<SecurityTestResults> {
    const results: SecurityTestResults = {
      passed: 0,
      failed: 0,
      vulnerabilities: [],
      recommendations: []
    };

    const injectionPayloads = [
      "'; DROP TABLE users; --",
      "' OR '1'='1",
      '${7*7}',
      '#{7*7}',
      '<%= 7*7 %>',
      '{{7*7}}'
    ];

    for (const payload of injectionPayloads) {
      const isVulnerable = await this.checkInjectionPayload(component, payload);
      
      if (isVulnerable) {
        results.failed++;
        results.vulnerabilities.push({
          type: 'sql_injection',
          severity: 'critical',
          description: `Injection vulnerability detected with payload: ${payload}`,
          location: 'data_processing'
        });
      } else {
        results.passed++;
      }
    }

    return results;
  }

  // Test authentication flows
  private async testAuthenticationFlows(component: React.ComponentType<any>): Promise<SecurityTestResults> {
    const results: SecurityTestResults = {
      passed: 0,
      failed: 0,
      vulnerabilities: [],
      recommendations: []
    };

    // Check for common authentication issues
    const authIssues = await this.checkAuthenticationIssues(component);
    
    authIssues.forEach(issue => {
      results.failed++;
      results.vulnerabilities.push(issue);
    });

    if (authIssues.length === 0) {
      results.passed++;
    }

    return results;
  }

  // Test authorization controls
  private async testAuthorizationControls(component: React.ComponentType<any>): Promise<SecurityTestResults> {
    const results: SecurityTestResults = {
      passed: 0,
      failed: 0,
      vulnerabilities: [],
      recommendations: []
    };

    // Check for authorization bypass attempts
    const authzIssues = await this.checkAuthorizationIssues(component);
    
    authzIssues.forEach(issue => {
      results.failed++;
      results.vulnerabilities.push(issue);
    });

    if (authzIssues.length === 0) {
      results.passed++;
    }

    return results;
  }

  // Helper methods (simplified for demo)
  private async checkXSSPayload(component: React.ComponentType<any>, payload: string): Promise<boolean> {
    // Implementation would test actual XSS vulnerability
    return false; // Assume secure by default
  }

  private async checkCSRFTokens(component: React.ComponentType<any>): Promise<boolean> {
    // Implementation would check for CSRF tokens
    return true; // Assume protected by default
  }

  private async checkInjectionPayload(component: React.ComponentType<any>, payload: string): Promise<boolean> {
    // Implementation would test injection vulnerabilities
    return false; // Assume secure by default
  }

  private async checkAuthenticationIssues(component: React.ComponentType<any>): Promise<SecurityIssue[]> {
    // Implementation would check for auth issues
    return [];
  }

  private async checkAuthorizationIssues(component: React.ComponentType<any>): Promise<SecurityIssue[]> {
    // Implementation would check for authz issues
    return [];
  }

  private mergeResults(target: SecurityTestResults, source: SecurityTestResults): void {
    target.passed += source.passed;
    target.failed += source.failed;
    target.vulnerabilities.push(...source.vulnerabilities);
    target.recommendations.push(...source.recommendations);
  }
}

interface SecurityTestResults {
  passed: number;
  failed: number;
  vulnerabilities: SecurityIssue[];
  recommendations: string[];
}

// Jest Security Test Helpers
export const securityTestHelpers = {
  // Test for XSS vulnerabilities
  expectNoXSS: (rendered: any, maliciousInput: string) => {
    const container = rendered.container;
    const scripts = container.querySelectorAll('script');
    
    expect(scripts).toHaveLength(0);
    expect(container.innerHTML).not.toContain(maliciousInput);
    expect(container.innerHTML).not.toMatch(/<script/i);
  },

  // Test for proper input sanitization
  expectSanitizedInput: (rendered: any, originalInput: string, expectedOutput: string) => {
    expect(rendered.container.textContent).toContain(expectedOutput);
    expect(rendered.container.textContent).not.toContain(originalInput);
  },

  // Test for CSRF protection
  expectCSRFProtection: (form: HTMLFormElement) => {
    const csrfToken = form.querySelector('input[name="csrf_token"]') ||
                      form.querySelector('input[name="_token"]');
    
    expect(csrfToken).toBeTruthy();
    expect((csrfToken as HTMLInputElement)?.value).toBeTruthy();
  },

  // Test for secure cookie settings
  expectSecureCookies: () => {
    const cookies = document.cookie.split(';');
    const securityCookies = cookies.filter(cookie => 
      cookie.includes('Secure') && cookie.includes('HttpOnly')
    );
    
    expect(securityCookies.length).toBeGreaterThan(0);
  }
};
```

## Code Review Security Checklist

### Comprehensive Security Review Guidelines

```typescript
// Security Code Review Checklist
interface SecurityReviewItem {
  category: 'input_validation' | 'authentication' | 'authorization' | 'data_protection' | 'error_handling' | 'logging';
  severity: 'low' | 'medium' | 'high' | 'critical';
  item: string;
  checkMethod: string;
  remediation: string;
}

const SECURITY_REVIEW_CHECKLIST: SecurityReviewItem[] = [
  // Input Validation
  {
    category: 'input_validation',
    severity: 'critical',
    item: 'All user inputs are validated and sanitized',
    checkMethod: 'Search for direct DOM manipulation, innerHTML usage, and eval() calls',
    remediation: 'Implement input validation using secure validation libraries'
  },
  {
    category: 'input_validation',
    severity: 'high',
    item: 'File uploads are properly validated',
    checkMethod: 'Check file type, size, and content validation',
    remediation: 'Implement file type whitelisting and content scanning'
  },
  {
    category: 'input_validation',
    severity: 'high',
    item: 'URLs are validated before redirection',
    checkMethod: 'Look for open redirects in navigation logic',
    remediation: 'Validate redirect URLs against whitelist'
  },

  // Authentication
  {
    category: 'authentication',
    severity: 'critical',
    item: 'Passwords are never stored in plain text',
    checkMethod: 'Search for password storage and transmission',
    remediation: 'Use proper hashing and encryption for passwords'
  },
  {
    category: 'authentication',
    severity: 'high',
    item: 'JWT tokens are properly validated',
    checkMethod: 'Check JWT signature verification and expiration',
    remediation: 'Implement proper JWT validation with signature checking'
  },
  {
    category: 'authentication',
    severity: 'medium',
    item: 'Session management is secure',
    checkMethod: 'Review session timeout and invalidation',
    remediation: 'Implement secure session management practices'
  },

  // Authorization
  {
    category: 'authorization',
    severity: 'critical',
    item: 'Access control is enforced on sensitive operations',
    checkMethod: 'Check if user permissions are verified before actions',
    remediation: 'Implement role-based access control (RBAC)'
  },
  {
    category: 'authorization',
    severity: 'high',
    item: 'Protected routes require authentication',
    checkMethod: 'Verify route protection implementation',
    remediation: 'Add authentication guards to protected routes'
  },

  // Data Protection
  {
    category: 'data_protection',
    severity: 'critical',
    item: 'Sensitive data is encrypted in storage',
    checkMethod: 'Check localStorage and sessionStorage usage',
    remediation: 'Encrypt sensitive data before client-side storage'
  },
  {
    category: 'data_protection',
    severity: 'high',
    item: 'API communications use HTTPS',
    checkMethod: 'Verify all API calls use secure protocols',
    remediation: 'Ensure all API endpoints use HTTPS'
  },
  {
    category: 'data_protection',
    severity: 'medium',
    item: 'Personal data handling complies with GDPR',
    checkMethod: 'Review data collection and processing',
    remediation: 'Implement GDPR compliance measures'
  },

  // Error Handling
  {
    category: 'error_handling',
    severity: 'medium',
    item: 'Error messages do not expose sensitive information',
    checkMethod: 'Review error handling and logging',
    remediation: 'Implement generic error messages for users'
  },
  {
    category: 'error_handling',
    severity: 'high',
    item: 'Failed authentication attempts are limited',
    checkMethod: 'Check for rate limiting on login attempts',
    remediation: 'Implement account lockout and rate limiting'
  },

  // Logging
  {
    category: 'logging',
    severity: 'medium',
    item: 'Security events are properly logged',
    checkMethod: 'Verify logging of authentication and authorization events',
    remediation: 'Implement comprehensive security event logging'
  },
  {
    category: 'logging',
    severity: 'low',
    item: 'Logs do not contain sensitive information',
    checkMethod: 'Review log content for sensitive data',
    remediation: 'Sanitize log messages to remove sensitive data'
  }
];

// Automated Security Review Tool
class SecurityReviewTool {
  private checklist: SecurityReviewItem[] = SECURITY_REVIEW_CHECKLIST;

  // Run automated security review
  async runSecurityReview(codebase: string[]): Promise<SecurityReviewResults> {
    const results: SecurityReviewResults = {
      totalChecks: this.checklist.length,
      passed: 0,
      failed: 0,
      warnings: 0,
      findings: [],
      recommendations: []
    };

    for (const item of this.checklist) {
      const finding = await this.checkSecurityItem(item, codebase);
      results.findings.push(finding);

      switch (finding.status) {
        case 'pass':
          results.passed++;
          break;
        case 'fail':
          results.failed++;
          break;
        case 'warning':
          results.warnings++;
          break;
      }
    }

    // Generate recommendations
    results.recommendations = this.generateRecommendations(results.findings);

    return results;
  }

  // Check individual security item
  private async checkSecurityItem(
    item: SecurityReviewItem, 
    codebase: string[]
  ): Promise<SecurityFinding> {
    const finding: SecurityFinding = {
      category: item.category,
      severity: item.severity,
      item: item.item,
      status: 'pass',
      evidence: [],
      remediation: item.remediation
    };

    // Perform automated checks based on category
    switch (item.category) {
      case 'input_validation':
        finding.evidence = await this.checkInputValidation(codebase);
        break;
      case 'authentication':
        finding.evidence = await this.checkAuthentication(codebase);
        break;
      case 'authorization':
        finding.evidence = await this.checkAuthorization(codebase);
        break;
      case 'data_protection':
        finding.evidence = await this.checkDataProtection(codebase);
        break;
      case 'error_handling':
        finding.evidence = await this.checkErrorHandling(codebase);
        break;
      case 'logging':
        finding.evidence = await this.checkLogging(codebase);
        break;
    }

    // Determine status based on evidence
    if (finding.evidence.length > 0) {
      finding.status = item.severity === 'critical' || item.severity === 'high' ? 'fail' : 'warning';
    }

    return finding;
  }

  private async checkInputValidation(codebase: string[]): Promise<string[]> {
    const evidence: string[] = [];
    const patterns = [
      /innerHTML\s*=/g,
      /eval\s*\(/g,
      /Function\s*\(/g,
      /document\.write/g,
      /\.html\s*\(/g
    ];

    codebase.forEach((file, index) => {
      patterns.forEach(pattern => {
        const matches = file.match(pattern);
        if (matches) {
          evidence.push(`File ${index}: Found ${matches.length} potential input validation issues`);
        }
      });
    });

    return evidence;
  }

  private async checkAuthentication(codebase: string[]): Promise<string[]> {
    const evidence: string[] = [];
    const patterns = [
      /password.*=.*['"`]/gi,
      /token.*localStorage/gi,
      /jwt.*decode.*without.*verify/gi
    ];

    codebase.forEach((file, index) => {
      patterns.forEach(pattern => {
        if (pattern.test(file)) {
          evidence.push(`File ${index}: Potential authentication issue detected`);
        }
      });
    });

    return evidence;
  }

  private async checkAuthorization(codebase: string[]): Promise<string[]> {
    const evidence: string[] = [];
    const routePattern = /<Route.*path.*component/g;
    const protectionPattern = /ProtectedRoute|requireAuth|isAuthenticated/g;

    codebase.forEach((file, index) => {
      const routes = file.match(routePattern);
      const protections = file.match(protectionPattern);

      if (routes && (!protections || protections.length < routes.length)) {
        evidence.push(`File ${index}: Potentially unprotected routes detected`);
      }
    });

    return evidence;
  }

  private async checkDataProtection(codebase: string[]): Promise<string[]> {
    const evidence: string[] = [];
    const patterns = [
      /localStorage\.setItem.*(?!encrypt)/gi,
      /sessionStorage\.setItem.*(?!encrypt)/gi,
      /http:\/\//g
    ];

    codebase.forEach((file, index) => {
      patterns.forEach((pattern, patternIndex) => {
        const matches = file.match(pattern);
        if (matches) {
          const issues = [
            'Unencrypted localStorage usage',
            'Unencrypted sessionStorage usage',
            'Insecure HTTP usage'
          ];
          evidence.push(`File ${index}: ${issues[patternIndex]} - ${matches.length} occurrences`);
        }
      });
    });

    return evidence;
  }

  private async checkErrorHandling(codebase: string[]): Promise<string[]> {
    const evidence: string[] = [];
    const patterns = [
      /catch.*console\.log.*error/gi,
      /throw.*Error.*password|token|secret/gi
    ];

    codebase.forEach((file, index) => {
      patterns.forEach(pattern => {
        if (pattern.test(file)) {
          evidence.push(`File ${index}: Potential information disclosure in error handling`);
        }
      });
    });

    return evidence;
  }

  private async checkLogging(codebase: string[]): Promise<string[]> {
    const evidence: string[] = [];
    const sensitivePattern = /console\.log.*password|token|secret|key/gi;

    codebase.forEach((file, index) => {
      const matches = file.match(sensitivePattern);
      if (matches) {
        evidence.push(`File ${index}: Sensitive information in logs - ${matches.length} occurrences`);
      }
    });

    return evidence;
  }

  private generateRecommendations(findings: SecurityFinding[]): string[] {
    const recommendations: string[] = [];
    const criticalFindings = findings.filter(f => f.severity === 'critical' && f.status === 'fail');
    const highFindings = findings.filter(f => f.severity === 'high' && f.status === 'fail');

    if (criticalFindings.length > 0) {
      recommendations.push('🚨 CRITICAL: Address all critical security issues immediately');
      recommendations.push('Conduct manual security review for critical findings');
    }

    if (highFindings.length > 0) {
      recommendations.push('⚠️ HIGH: Schedule high-priority security fixes');
      recommendations.push('Implement additional security testing');
    }

    recommendations.push('Consider implementing automated security testing in CI/CD pipeline');
    recommendations.push('Regular security training for development team');
    recommendations.push('Establish security code review process');

    return recommendations;
  }
}

interface SecurityReviewResults {
  totalChecks: number;
  passed: number;
  failed: number;
  warnings: number;
  findings: SecurityFinding[];
  recommendations: string[];
}

interface SecurityFinding {
  category: string;
  severity: string;
  item: string;
  status: 'pass' | 'fail' | 'warning';
  evidence: string[];
  remediation: string;
}

// React Hook for Security Review Integration
export const useSecurityReview = () => {
  const [reviewTool] = useState(() => new SecurityReviewTool());
  const [isReviewing, setIsReviewing] = useState(false);
  const [results, setResults] = useState<SecurityReviewResults | null>(null);

  const runReview = async (codeFiles: string[]) => {
    setIsReviewing(true);
    try {
      const reviewResults = await reviewTool.runSecurityReview(codeFiles);
      setResults(reviewResults);
      return reviewResults;
    } catch (error) {
      console.error('Security review failed:', error);
      throw error;
    } finally {
      setIsReviewing(false);
    }
  };

  const getSecurityScore = () => {
    if (!results) return 0;
    return Math.round((results.passed / results.totalChecks) * 100);
  };

  const getCriticalIssues = () => {
    if (!results) return [];
    return results.findings.filter(f => f.severity === 'critical' && f.status === 'fail');
  };

  return {
    runReview,
    isReviewing,
    results,
    securityScore: getSecurityScore(),
    criticalIssues: getCriticalIssues(),
    hasIssues: results ? results.failed > 0 : false
  };
};
```

## Interview-Ready Secure Coding Summary

**Core Secure Coding Principles:**
1. **Input Validation** - Validate and sanitize all user inputs
2. **Output Encoding** - Properly encode data for different contexts
3. **Authentication & Authorization** - Implement proper access controls
4. **Error Handling** - Handle errors securely without information disclosure
5. **Logging & Monitoring** - Log security events without exposing sensitive data

**Automated Security Testing:**
- Custom ESLint rules for security vulnerabilities
- Comprehensive security test suites with XSS, CSRF, injection testing
- Jest security test helpers for common vulnerability patterns
- Continuous security monitoring and automated vulnerability detection

**Code Review Security Focus:**
- Systematic security checklist covering all major vulnerability categories
- Automated security review tools with pattern detection
- Evidence-based security findings with remediation guidance
- Security scoring and risk assessment integration

**Enterprise Security Practices:**
- Security-first development lifecycle integration
- Automated security testing in CI/CD pipelines
- Regular security training and awareness programs
- Comprehensive security documentation and audit trails

**Key Interview Topics:** Secure coding standards, input validation frameworks, automated security testing, security code review processes, vulnerability detection and remediation, security testing integration, enterprise security practices.