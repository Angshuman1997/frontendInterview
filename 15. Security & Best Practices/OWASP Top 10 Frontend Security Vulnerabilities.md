# OWASP Top 10 for Frontend Applications

The OWASP Top 10 represents the most critical security risks to web applications. This guide covers frontend-specific implementations and prevention strategies for each vulnerability, with enterprise-grade security patterns and real-world examples.

## 1. A01:2021 – Broken Access Control

### Understanding the Risk
Access control enforces policy such that users cannot act outside of their intended permissions. Frontend applications must implement robust client-side controls while never relying solely on them for security.

### Frontend Implementation

```typescript
// Role-Based Access Control (RBAC) System
interface Permission {
  resource: string;
  action: 'create' | 'read' | 'update' | 'delete';
  scope?: 'own' | 'team' | 'organization' | 'global';
}

interface Role {
  id: string;
  name: string;
  permissions: Permission[];
  hierarchy: number; // Higher number = more privileges
}

interface User {
  id: string;
  roles: Role[];
  permissions: Permission[];
  organizationId: string;
  teamId?: string;
}

class AccessControlManager {
  private currentUser: User | null = null;

  setUser(user: User): void {
    this.currentUser = user;
  }

  // Check if user has specific permission
  hasPermission(resource: string, action: string, scope: string = 'own'): boolean {
    if (!this.currentUser) return false;

    // Check direct permissions
    const hasDirectPermission = this.currentUser.permissions.some(
      p => p.resource === resource && 
           p.action === action && 
           this.checkScope(p.scope, scope)
    );

    if (hasDirectPermission) return true;

    // Check role-based permissions
    return this.currentUser.roles.some(role =>
      role.permissions.some(p =>
        p.resource === resource &&
        p.action === action &&
        this.checkScope(p.scope, scope)
      )
    );
  }

  // Check if user can access specific resource
  canAccess(resourceType: string, resourceId: string, action: string): boolean {
    if (!this.currentUser) return false;

    // Admin override
    if (this.hasRole('admin')) return true;

    // Resource-specific checks
    switch (resourceType) {
      case 'user':
        return this.canAccessUser(resourceId, action);
      case 'document':
        return this.canAccessDocument(resourceId, action);
      case 'project':
        return this.canAccessProject(resourceId, action);
      default:
        return this.hasPermission(resourceType, action);
    }
  }

  // Role checking
  hasRole(roleName: string): boolean {
    if (!this.currentUser) return false;
    return this.currentUser.roles.some(role => role.name === roleName);
  }

  // Get user's highest role hierarchy
  getHighestRoleHierarchy(): number {
    if (!this.currentUser) return 0;
    return Math.max(...this.currentUser.roles.map(role => role.hierarchy));
  }

  private checkScope(permissionScope: string = 'own', requiredScope: string): boolean {
    const scopeHierarchy = {
      'own': 1,
      'team': 2,
      'organization': 3,
      'global': 4
    };

    return scopeHierarchy[permissionScope] >= scopeHierarchy[requiredScope];
  }

  private canAccessUser(userId: string, action: string): boolean {
    // Users can read their own profile
    if (action === 'read' && userId === this.currentUser?.id) return true;
    
    // Check if user has user management permissions
    return this.hasPermission('user', action);
  }

  private canAccessDocument(documentId: string, action: string): boolean {
    // This would typically check document ownership/sharing
    // For demo, checking basic permissions
    return this.hasPermission('document', action);
  }

  private canAccessProject(projectId: string, action: string): boolean {
    // Check project membership and permissions
    return this.hasPermission('project', action);
  }
}

// React Hook for Access Control
export const useAccessControl = () => {
  const [accessControl] = useState(() => new AccessControlManager());
  
  return {
    hasPermission: accessControl.hasPermission.bind(accessControl),
    canAccess: accessControl.canAccess.bind(accessControl),
    hasRole: accessControl.hasRole.bind(accessControl),
    setUser: accessControl.setUser.bind(accessControl)
  };
};

// Protected Route Component
export const ProtectedRoute: React.FC<{
  children: React.ReactNode;
  requiredPermission?: { resource: string; action: string };
  requiredRole?: string;
  fallback?: React.ReactNode;
}> = ({ children, requiredPermission, requiredRole, fallback }) => {
  const { hasPermission, hasRole } = useAccessControl();
  
  const hasAccess = useMemo(() => {
    if (requiredRole && !hasRole(requiredRole)) return false;
    if (requiredPermission && !hasPermission(
      requiredPermission.resource, 
      requiredPermission.action
    )) return false;
    return true;
  }, [requiredPermission, requiredRole, hasPermission, hasRole]);

  if (!hasAccess) {
    return fallback || <div>Access Denied</div>;
  }

  return <>{children}</>;
};
```

## 2. A02:2021 – Cryptographic Failures

### Secure Client-Side Encryption

```typescript
// Client-Side Encryption Utilities
class ClientCrypto {
  // Generate secure random key
  static async generateKey(): Promise<CryptoKey> {
    return await window.crypto.subtle.generateKey(
      {
        name: 'AES-GCM',
        length: 256,
      },
      true, // extractable
      ['encrypt', 'decrypt']
    );
  }

  // Encrypt data using Web Crypto API
  static async encrypt(data: string, key: CryptoKey): Promise<{ 
    encrypted: ArrayBuffer; 
    iv: Uint8Array 
  }> {
    const encoder = new TextEncoder();
    const dataBuffer = encoder.encode(data);
    
    const iv = window.crypto.getRandomValues(new Uint8Array(12));
    
    const encrypted = await window.crypto.subtle.encrypt(
      {
        name: 'AES-GCM',
        iv: iv,
      },
      key,
      dataBuffer
    );

    return { encrypted, iv };
  }

  // Decrypt data
  static async decrypt(
    encryptedData: ArrayBuffer, 
    key: CryptoKey, 
    iv: Uint8Array
  ): Promise<string> {
    const decrypted = await window.crypto.subtle.decrypt(
      {
        name: 'AES-GCM',
        iv: iv,
      },
      key,
      encryptedData
    );

    const decoder = new TextDecoder();
    return decoder.decode(decrypted);
  }

  // Secure hash generation
  static async hash(data: string): Promise<string> {
    const encoder = new TextEncoder();
    const dataBuffer = encoder.encode(data);
    
    const hashBuffer = await window.crypto.subtle.digest('SHA-256', dataBuffer);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    
    return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  }

  // Password-based key derivation
  static async deriveKey(password: string, salt: Uint8Array): Promise<CryptoKey> {
    const encoder = new TextEncoder();
    const keyMaterial = await window.crypto.subtle.importKey(
      'raw',
      encoder.encode(password),
      { name: 'PBKDF2' },
      false,
      ['deriveBits', 'deriveKey']
    );

    return await window.crypto.subtle.deriveKey(
      {
        name: 'PBKDF2',
        salt: salt,
        iterations: 100000,
        hash: 'SHA-256',
      },
      keyMaterial,
      { name: 'AES-GCM', length: 256 },
      true,
      ['encrypt', 'decrypt']
    );
  }

  // Secure random number generation
  static generateSecureRandom(length: number): Uint8Array {
    return window.crypto.getRandomValues(new Uint8Array(length));
  }
}

// Secure Local Storage with Encryption
class SecureStorage {
  private static key: CryptoKey | null = null;

  static async initialize(password?: string): Promise<void> {
    if (password) {
      const salt = this.getOrCreateSalt();
      this.key = await ClientCrypto.deriveKey(password, salt);
    } else {
      this.key = await ClientCrypto.generateKey();
    }
  }

  static async setItem(key: string, value: any): Promise<void> {
    if (!this.key) throw new Error('SecureStorage not initialized');

    const serialized = JSON.stringify(value);
    const { encrypted, iv } = await ClientCrypto.encrypt(serialized, this.key);
    
    const storageData = {
      encrypted: Array.from(new Uint8Array(encrypted)),
      iv: Array.from(iv)
    };

    localStorage.setItem(key, JSON.stringify(storageData));
  }

  static async getItem<T>(key: string): Promise<T | null> {
    if (!this.key) throw new Error('SecureStorage not initialized');

    const storedData = localStorage.getItem(key);
    if (!storedData) return null;

    try {
      const { encrypted, iv } = JSON.parse(storedData);
      const encryptedBuffer = new Uint8Array(encrypted).buffer;
      const ivArray = new Uint8Array(iv);

      const decrypted = await ClientCrypto.decrypt(encryptedBuffer, this.key, ivArray);
      return JSON.parse(decrypted);
    } catch (error) {
      console.error('Failed to decrypt stored data:', error);
      return null;
    }
  }

  static removeItem(key: string): void {
    localStorage.removeItem(key);
  }

  private static getOrCreateSalt(): Uint8Array {
    const stored = localStorage.getItem('_salt');
    if (stored) {
      return new Uint8Array(JSON.parse(stored));
    }

    const salt = ClientCrypto.generateSecureRandom(16);
    localStorage.setItem('_salt', JSON.stringify(Array.from(salt)));
    return salt;
  }
}
```

## 3. A03:2021 – Injection

### Input Validation and Sanitization

```typescript
// Comprehensive Input Validation
interface ValidationRule {
  type: 'required' | 'email' | 'url' | 'length' | 'pattern' | 'custom';
  message: string;
  params?: any;
  validator?: (value: any) => boolean;
}

class InputValidator {
  private static readonly EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  private static readonly URL_REGEX = /^https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)$/;
  private static readonly SQL_INJECTION_PATTERNS = [
    /(\bSELECT\b|\bINSERT\b|\bUPDATE\b|\bDELETE\b|\bDROP\b|\bCREATE\b)/i,
    /(\bUNION\b|\bOR\b|\bAND\b)\s+\d+\s*=\s*\d+/i,
    /[\'"]\s*(OR|AND)\s+[\'"]\d+[\'"]?\s*=\s*[\'"]\d+/i
  ];

  static validate(value: any, rules: ValidationRule[]): { isValid: boolean; errors: string[] } {
    const errors: string[] = [];

    for (const rule of rules) {
      const isValid = this.validateRule(value, rule);
      if (!isValid) {
        errors.push(rule.message);
      }
    }

    return {
      isValid: errors.length === 0,
      errors
    };
  }

  private static validateRule(value: any, rule: ValidationRule): boolean {
    switch (rule.type) {
      case 'required':
        return value !== null && value !== undefined && value !== '';
      
      case 'email':
        return typeof value === 'string' && this.EMAIL_REGEX.test(value);
      
      case 'url':
        return typeof value === 'string' && this.URL_REGEX.test(value);
      
      case 'length':
        const length = typeof value === 'string' ? value.length : 0;
        const { min = 0, max = Infinity } = rule.params || {};
        return length >= min && length <= max;
      
      case 'pattern':
        return typeof value === 'string' && rule.params.test(value);
      
      case 'custom':
        return rule.validator ? rule.validator(value) : true;
      
      default:
        return true;
    }
  }

  // SQL Injection Detection
  static detectSQLInjection(input: string): boolean {
    return this.SQL_INJECTION_PATTERNS.some(pattern => pattern.test(input));
  }

  // XSS Detection
  static detectXSS(input: string): boolean {
    const xssPatterns = [
      /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi,
      /javascript:/gi,
      /on\w+\s*=/gi,
      /<iframe\b[^<]*(?:(?!<\/iframe>)<[^<]*)*<\/iframe>/gi
    ];

    return xssPatterns.some(pattern => pattern.test(input));
  }

  // Command Injection Detection
  static detectCommandInjection(input: string): boolean {
    const commandPatterns = [
      /[;&|`$(){}[\]]/,
      /\b(cat|ls|ps|whoami|pwd|id|uname|echo|curl|wget)\b/i
    ];

    return commandPatterns.some(pattern => pattern.test(input));
  }
}

// React Hook for Form Validation
export const useFormValidation = <T extends Record<string, any>>(
  initialValues: T,
  validationSchema: Record<keyof T, ValidationRule[]>
) => {
  const [values, setValues] = useState<T>(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string[]>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});

  const validateField = useCallback((name: keyof T, value: any) => {
    const rules = validationSchema[name];
    if (!rules) return { isValid: true, errors: [] };

    const result = InputValidator.validate(value, rules);
    
    // Additional security checks
    if (typeof value === 'string') {
      if (InputValidator.detectSQLInjection(value)) {
        result.errors.push('Potential SQL injection detected');
        result.isValid = false;
      }
      
      if (InputValidator.detectXSS(value)) {
        result.errors.push('Potential XSS attack detected');
        result.isValid = false;
      }
      
      if (InputValidator.detectCommandInjection(value)) {
        result.errors.push('Potential command injection detected');
        result.isValid = false;
      }
    }

    return result;
  }, [validationSchema]);

  const setValue = useCallback((name: keyof T, value: any) => {
    setValues(prev => ({ ...prev, [name]: value }));
    
    if (touched[name]) {
      const validation = validateField(name, value);
      setErrors(prev => ({
        ...prev,
        [name]: validation.isValid ? [] : validation.errors
      }));
    }
  }, [touched, validateField]);

  const setFieldTouched = useCallback((name: keyof T) => {
    setTouched(prev => ({ ...prev, [name]: true }));
    
    const validation = validateField(name, values[name]);
    setErrors(prev => ({
      ...prev,
      [name]: validation.isValid ? [] : validation.errors
    }));
  }, [values, validateField]);

  const validateAll = useCallback(() => {
    const newErrors: Partial<Record<keyof T, string[]>> = {};
    let isFormValid = true;

    Object.keys(validationSchema).forEach(key => {
      const fieldName = key as keyof T;
      const validation = validateField(fieldName, values[fieldName]);
      
      if (!validation.isValid) {
        newErrors[fieldName] = validation.errors;
        isFormValid = false;
      }
    });

    setErrors(newErrors);
    setTouched(Object.keys(validationSchema).reduce((acc, key) => ({
      ...acc,
      [key]: true
    }), {}));

    return isFormValid;
  }, [values, validationSchema, validateField]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  }, [initialValues]);

  return {
    values,
    errors,
    touched,
    setValue,
    setFieldTouched,
    validateAll,
    reset,
    isValid: Object.keys(errors).length === 0
  };
};
```

## 4. A04:2021 – Insecure Design

### Secure Design Patterns

```typescript
// Secure Component Design Pattern
interface SecureComponentProps {
  children: React.ReactNode;
  securityLevel: 'public' | 'authenticated' | 'admin';
  auditLog?: boolean;
  rateLimited?: boolean;
  encryptSensitiveData?: boolean;
}

const SecureComponent: React.FC<SecureComponentProps> = ({
  children,
  securityLevel,
  auditLog = false,
  rateLimited = false,
  encryptSensitiveData = false
}) => {
  const { user, hasRole } = useAuth();
  const [isAuthorized, setIsAuthorized] = useState(false);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const checkAuthorization = async () => {
      setIsLoading(true);
      
      try {
        switch (securityLevel) {
          case 'public':
            setIsAuthorized(true);
            break;
          case 'authenticated':
            setIsAuthorized(!!user);
            break;
          case 'admin':
            setIsAuthorized(hasRole('admin'));
            break;
          default:
            setIsAuthorized(false);
        }

        // Audit logging
        if (auditLog && user) {
          await logSecurityEvent({
            userId: user.id,
            action: 'component_access',
            component: 'SecureComponent',
            securityLevel,
            timestamp: new Date().toISOString()
          });
        }
      } catch (error) {
        console.error('Authorization check failed:', error);
        setIsAuthorized(false);
      } finally {
        setIsLoading(false);
      }
    };

    checkAuthorization();
  }, [user, securityLevel, hasRole, auditLog]);

  // Rate limiting check
  useEffect(() => {
    if (rateLimited && isAuthorized) {
      // Implement rate limiting logic
      const checkRateLimit = async () => {
        const isAllowed = await checkUserRateLimit(user?.id);
        if (!isAllowed) {
          setIsAuthorized(false);
        }
      };
      
      checkRateLimit();
    }
  }, [rateLimited, isAuthorized, user]);

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!isAuthorized) {
    return <div>Access Denied</div>;
  }

  return <>{children}</>;
};

// Secure Data Handling Hook
export const useSecureData = <T>(
  data: T,
  options: {
    encrypt?: boolean;
    auditAccess?: boolean;
    maskSensitive?: boolean;
  } = {}
) => {
  const [secureData, setSecureData] = useState<T | null>(null);
  const [isProcessing, setIsProcessing] = useState(true);
  const { user } = useAuth();

  useEffect(() => {
    const processData = async () => {
      setIsProcessing(true);
      
      try {
        let processedData = data;

        // Audit data access
        if (options.auditAccess && user) {
          await logSecurityEvent({
            userId: user.id,
            action: 'data_access',
            dataType: typeof data,
            timestamp: new Date().toISOString()
          });
        }

        // Mask sensitive data
        if (options.maskSensitive) {
          processedData = maskSensitiveFields(data);
        }

        // Encrypt if requested
        if (options.encrypt) {
          // In a real implementation, you'd encrypt the data
          // For demo, we'll just mark it as encrypted
          processedData = { ...processedData, _encrypted: true } as T;
        }

        setSecureData(processedData);
      } catch (error) {
        console.error('Secure data processing failed:', error);
        setSecureData(null);
      } finally {
        setIsProcessing(false);
      }
    };

    processData();
  }, [data, options, user]);

  return {
    data: secureData,
    isProcessing,
    isSecure: !!options.encrypt || !!options.maskSensitive
  };
};

// Helper functions
const logSecurityEvent = async (event: any) => {
  try {
    await fetch('/api/security/audit', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(event)
    });
  } catch (error) {
    console.error('Failed to log security event:', error);
  }
};

const checkUserRateLimit = async (userId?: string): Promise<boolean> => {
  if (!userId) return false;
  
  try {
    const response = await fetch(`/api/security/rate-limit/${userId}`);
    const result = await response.json();
    return result.allowed;
  } catch (error) {
    console.error('Rate limit check failed:', error);
    return false;
  }
};

const maskSensitiveFields = <T>(data: T): T => {
  if (typeof data !== 'object' || data === null) return data;
  
  const sensitiveFields = ['password', 'ssn', 'creditCard', 'email'];
  const masked = { ...data };
  
  Object.keys(masked).forEach(key => {
    if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
      (masked as any)[key] = '***MASKED***';
    }
  });
  
  return masked;
};
```

## Interview-Ready OWASP Summary

**Top Frontend Security Priorities:**
1. **Access Control** - Implement RBAC with proper permission checking
2. **Cryptographic Protection** - Use Web Crypto API for client-side encryption
3. **Injection Prevention** - Validate and sanitize all user inputs
4. **Secure Design** - Build security into component architecture

**Common Frontend Vulnerabilities:**
- Client-side access control bypass
- Sensitive data exposure in browser storage
- XSS through unvalidated inputs
- Insecure direct object references
- Weak encryption implementations

**Enterprise Security Patterns:**
- Multi-layered access control validation
- Comprehensive input validation with security checks
- Secure component design with audit logging
- Encrypted client-side data storage
- Rate limiting and abuse prevention

**Key Interview Topics:** OWASP Top 10 awareness, secure coding practices, input validation strategies, access control implementation, client-side encryption, security design patterns, vulnerability assessment and prevention.