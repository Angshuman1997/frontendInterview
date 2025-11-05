# JWT Security and Authentication Best Practices

JSON Web Tokens (JWT) are widely used for authentication and authorization in modern web applications. This guide covers enterprise-grade JWT implementation, security best practices, and common vulnerabilities with their prevention strategies.

## JWT Security Fundamentals

### 1. Secure JWT Implementation

```typescript
// Enterprise JWT Management System
import { sign, verify, decode, JwtPayload } from 'jsonwebtoken';
import { randomBytes, createHash, timingSafeEqual } from 'crypto';

interface JWTConfig {
  accessTokenSecret: string;
  refreshTokenSecret: string;
  accessTokenExpiry: string;
  refreshTokenExpiry: string;
  issuer: string;
  audience: string;
  algorithm: 'HS256' | 'RS256' | 'ES256';
  keyRotationInterval: number;
}

interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

interface CustomPayload extends JwtPayload {
  userId: string;
  role: string;
  permissions: string[];
  sessionId: string;
  tokenVersion: number;
}

class SecureJWTManager {
  private config: JWTConfig;
  private blacklistedTokens: Set<string> = new Set();
  private activeRefreshTokens: Map<string, string> = new Map(); // userId -> refreshToken

  constructor(config: JWTConfig) {
    this.config = config;
    this.initializeCleanupScheduler();
  }

  // Generate cryptographically secure token pair
  async generateTokenPair(payload: Omit<CustomPayload, 'iat' | 'exp' | 'iss' | 'aud'>): Promise<TokenPair> {
    const sessionId = this.generateSecureId();
    const now = Math.floor(Date.now() / 1000);

    // Access token payload
    const accessPayload: CustomPayload = {
      ...payload,
      sessionId,
      iat: now,
      iss: this.config.issuer,
      aud: this.config.audience
    };

    // Refresh token payload (minimal information)
    const refreshPayload = {
      userId: payload.userId,
      sessionId,
      tokenType: 'refresh',
      iat: now,
      iss: this.config.issuer,
      aud: this.config.audience
    };

    try {
      const accessToken = sign(accessPayload, this.config.accessTokenSecret, {
        expiresIn: this.config.accessTokenExpiry,
        algorithm: this.config.algorithm
      });

      const refreshToken = sign(refreshPayload, this.config.refreshTokenSecret, {
        expiresIn: this.config.refreshTokenExpiry,
        algorithm: this.config.algorithm
      });

      // Store refresh token association
      this.activeRefreshTokens.set(payload.userId, refreshToken);

      return {
        accessToken,
        refreshToken,
        expiresIn: this.parseExpiryToSeconds(this.config.accessTokenExpiry)
      };
    } catch (error) {
      throw new Error(`Failed to generate tokens: ${error.message}`);
    }
  }

  // Verify and decode access token
  async verifyAccessToken(token: string): Promise<CustomPayload> {
    if (this.blacklistedTokens.has(token)) {
      throw new Error('Token has been revoked');
    }

    try {
      const decoded = verify(token, this.config.accessTokenSecret, {
        issuer: this.config.issuer,
        audience: this.config.audience,
        algorithms: [this.config.algorithm]
      }) as CustomPayload;

      // Additional security checks
      this.validateTokenStructure(decoded);
      await this.validateSession(decoded.sessionId);

      return decoded;
    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        throw new Error('Access token has expired');
      } else if (error.name === 'JsonWebTokenError') {
        throw new Error('Invalid access token');
      }
      throw error;
    }
  }

  // Verify refresh token and generate new access token
  async refreshAccessToken(refreshToken: string): Promise<TokenPair> {
    if (this.blacklistedTokens.has(refreshToken)) {
      throw new Error('Refresh token has been revoked');
    }

    try {
      const decoded = verify(refreshToken, this.config.refreshTokenSecret, {
        issuer: this.config.issuer,
        audience: this.config.audience,
        algorithms: [this.config.algorithm]
      }) as any;

      // Verify refresh token is still active
      const storedToken = this.activeRefreshTokens.get(decoded.userId);
      if (!storedToken || !timingSafeEqual(
        Buffer.from(storedToken), 
        Buffer.from(refreshToken)
      )) {
        throw new Error('Refresh token is not valid');
      }

      // Get updated user information
      const userPayload = await this.getUserPayload(decoded.userId);
      
      // Generate new token pair
      const newTokenPair = await this.generateTokenPair(userPayload);

      // Revoke old refresh token
      this.blacklistedTokens.add(refreshToken);
      
      return newTokenPair;
    } catch (error) {
      throw new Error(`Failed to refresh token: ${error.message}`);
    }
  }

  // Revoke all tokens for a user
  async revokeAllTokens(userId: string): Promise<void> {
    const refreshToken = this.activeRefreshTokens.get(userId);
    if (refreshToken) {
      this.blacklistedTokens.add(refreshToken);
      this.activeRefreshTokens.delete(userId);
    }

    // In production, you'd also want to:
    // 1. Increment token version in database
    // 2. Invalidate all sessions for the user
    // 3. Clear any cached user data
  }

  // Revoke specific token
  revokeToken(token: string): void {
    this.blacklistedTokens.add(token);
  }

  // Validate token structure
  private validateTokenStructure(payload: CustomPayload): void {
    const requiredFields = ['userId', 'role', 'sessionId', 'iat'];
    
    for (const field of requiredFields) {
      if (!(field in payload)) {
        throw new Error(`Token missing required field: ${field}`);
      }
    }

    // Validate role and permissions
    if (!payload.role || typeof payload.role !== 'string') {
      throw new Error('Invalid role in token');
    }

    if (!Array.isArray(payload.permissions)) {
      throw new Error('Invalid permissions in token');
    }
  }

  // Validate session (implement based on your session storage)
  private async validateSession(sessionId: string): Promise<void> {
    // This would typically check against your session store
    // For now, we'll do basic validation
    if (!sessionId || typeof sessionId !== 'string') {
      throw new Error('Invalid session ID');
    }
  }

  // Get updated user payload (implement based on your user store)
  private async getUserPayload(userId: string): Promise<Omit<CustomPayload, 'iat' | 'exp' | 'iss' | 'aud'>> {
    // This would typically fetch from your database
    // For demo purposes, returning mock data
    return {
      userId,
      role: 'user',
      permissions: ['read', 'write'],
      sessionId: this.generateSecureId(),
      tokenVersion: 1
    };
  }

  // Generate cryptographically secure ID
  private generateSecureId(): string {
    return randomBytes(32).toString('hex');
  }

  // Parse expiry string to seconds
  private parseExpiryToSeconds(expiry: string): number {
    const match = expiry.match(/^(\d+)([smhd])$/);
    if (!match) return 3600; // Default 1 hour

    const [, value, unit] = match;
    const num = parseInt(value, 10);

    switch (unit) {
      case 's': return num;
      case 'm': return num * 60;
      case 'h': return num * 3600;
      case 'd': return num * 86400;
      default: return 3600;
    }
  }

  // Cleanup expired blacklisted tokens
  private initializeCleanupScheduler(): void {
    setInterval(() => {
      this.cleanupExpiredTokens();
    }, 60 * 60 * 1000); // Clean up every hour
  }

  private cleanupExpiredTokens(): void {
    // In production, implement proper cleanup based on token expiration
    // For now, we'll keep it simple
    if (this.blacklistedTokens.size > 10000) {
      this.blacklistedTokens.clear();
    }
  }
}

// React Hook for JWT Authentication
import { createContext, useContext, useEffect, useState, useCallback } from 'react';

interface AuthContextType {
  user: CustomPayload | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
}

interface LoginCredentials {
  email: string;
  password: string;
  rememberMe?: boolean;
}

const AuthContext = createContext<AuthContextType | null>(null);

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

// Secure Token Storage
class SecureTokenStorage {
  private static readonly ACCESS_TOKEN_KEY = 'access_token';
  private static readonly REFRESH_TOKEN_KEY = 'refresh_token';
  private static readonly TOKEN_EXPIRY_KEY = 'token_expiry';

  // Store tokens securely
  static storeTokens(tokenPair: TokenPair): void {
    const expiryTime = Date.now() + (tokenPair.expiresIn * 1000);
    
    // Store access token in memory/sessionStorage for security
    sessionStorage.setItem(this.ACCESS_TOKEN_KEY, tokenPair.accessToken);
    sessionStorage.setItem(this.TOKEN_EXPIRY_KEY, expiryTime.toString());
    
    // Store refresh token in httpOnly cookie (handled by server)
    // OR in secure localStorage with encryption for SPA
    this.storeRefreshTokenSecurely(tokenPair.refreshToken);
  }

  // Get access token
  static getAccessToken(): string | null {
    const token = sessionStorage.getItem(this.ACCESS_TOKEN_KEY);
    const expiry = sessionStorage.getItem(this.TOKEN_EXPIRY_KEY);
    
    if (!token || !expiry) return null;
    
    if (Date.now() > parseInt(expiry, 10)) {
      this.clearTokens();
      return null;
    }
    
    return token;
  }

  // Get refresh token
  static getRefreshToken(): string | null {
    // In production, this would be retrieved from httpOnly cookie
    return localStorage.getItem(this.REFRESH_TOKEN_KEY);
  }

  // Clear all tokens
  static clearTokens(): void {
    sessionStorage.removeItem(this.ACCESS_TOKEN_KEY);
    sessionStorage.removeItem(this.TOKEN_EXPIRY_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
  }

  // Secure refresh token storage (simplified)
  private static storeRefreshTokenSecurely(refreshToken: string): void {
    // In production, prefer httpOnly cookies set by server
    // This is a fallback for client-only applications
    localStorage.setItem(this.REFRESH_TOKEN_KEY, refreshToken);
  }
}

// Auth Provider Component
export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<CustomPayload | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const jwtManager = useMemo(() => {
    const config: JWTConfig = {
      accessTokenSecret: process.env.REACT_APP_JWT_ACCESS_SECRET!,
      refreshTokenSecret: process.env.REACT_APP_JWT_REFRESH_SECRET!,
      accessTokenExpiry: '15m',
      refreshTokenExpiry: '7d',
      issuer: 'your-app.com',
      audience: 'your-app-users',
      algorithm: 'HS256',
      keyRotationInterval: 24 * 60 * 60 * 1000 // 24 hours
    };
    
    return new SecureJWTManager(config);
  }, []);

  // Initialize authentication state
  useEffect(() => {
    initializeAuth();
  }, []);

  const initializeAuth = async () => {
    setIsLoading(true);
    
    try {
      const accessToken = SecureTokenStorage.getAccessToken();
      
      if (accessToken) {
        const payload = await jwtManager.verifyAccessToken(accessToken);
        setUser(payload);
      } else {
        // Try to refresh with refresh token
        await refreshToken();
      }
    } catch (error) {
      console.warn('Auth initialization failed:', error);
      SecureTokenStorage.clearTokens();
    } finally {
      setIsLoading(false);
    }
  };

  const login = async (credentials: LoginCredentials) => {
    try {
      setIsLoading(true);
      
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(credentials)
      });

      if (!response.ok) {
        throw new Error('Login failed');
      }

      const { tokenPair, user: userData } = await response.json();
      
      SecureTokenStorage.storeTokens(tokenPair);
      setUser(userData);
    } catch (error) {
      throw new Error(`Login failed: ${error.message}`);
    } finally {
      setIsLoading(false);
    }
  };

  const logout = async () => {
    try {
      const refreshToken = SecureTokenStorage.getRefreshToken();
      
      if (refreshToken) {
        await fetch('/api/auth/logout', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ refreshToken })
        });
      }
    } catch (error) {
      console.warn('Logout API call failed:', error);
    } finally {
      SecureTokenStorage.clearTokens();
      setUser(null);
    }
  };

  const refreshToken = async () => {
    try {
      const refreshToken = SecureTokenStorage.getRefreshToken();
      
      if (!refreshToken) {
        throw new Error('No refresh token available');
      }

      const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken })
      });

      if (!response.ok) {
        throw new Error('Token refresh failed');
      }

      const { tokenPair, user: userData } = await response.json();
      
      SecureTokenStorage.storeTokens(tokenPair);
      setUser(userData);
    } catch (error) {
      SecureTokenStorage.clearTokens();
      setUser(null);
      throw error;
    }
  };

  // Auto-refresh token before expiration
  useEffect(() => {
    if (!user) return;

    const interval = setInterval(async () => {
      try {
        await refreshToken();
      } catch (error) {
        console.warn('Auto token refresh failed:', error);
      }
    }, 10 * 60 * 1000); // Refresh every 10 minutes

    return () => clearInterval(interval);
  }, [user]);

  const value: AuthContextType = {
    user,
    isAuthenticated: !!user,
    isLoading,
    login,
    logout,
    refreshToken
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};
```

## JWT Security Best Practices

### 1. Token Storage Security

```typescript
// Secure Token Storage Strategies
class TokenSecurityManager {
  // Method 1: Memory Storage (Most Secure)
  private static inMemoryToken: string | null = null;
  
  static storeInMemory(token: string): void {
    this.inMemoryToken = token;
  }
  
  static getFromMemory(): string | null {
    return this.inMemoryToken;
  }
  
  static clearMemory(): void {
    this.inMemoryToken = null;
  }

  // Method 2: Encrypted Local Storage
  static storeEncrypted(token: string, encryptionKey: string): void {
    try {
      const encrypted = this.encrypt(token, encryptionKey);
      localStorage.setItem('auth_token_enc', encrypted);
    } catch (error) {
      console.error('Failed to encrypt token:', error);
    }
  }
  
  static getDecrypted(encryptionKey: string): string | null {
    try {
      const encrypted = localStorage.getItem('auth_token_enc');
      if (!encrypted) return null;
      
      return this.decrypt(encrypted, encryptionKey);
    } catch (error) {
      console.error('Failed to decrypt token:', error);
      return null;
    }
  }

  // Simple encryption (use a proper crypto library in production)
  private static encrypt(text: string, key: string): string {
    // In production, use Web Crypto API or a proper encryption library
    return btoa(text + key);
  }
  
  private static decrypt(encrypted: string, key: string): string {
    // In production, use Web Crypto API or a proper decryption library
    const decrypted = atob(encrypted);
    return decrypted.replace(key, '');
  }

  // Method 3: HttpOnly Cookie (Server-Side Handling)
  static setHttpOnlyCookie(token: string): void {
    // This would be handled by your backend
    fetch('/api/auth/set-cookie', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ token }),
      credentials: 'include'
    });
  }
}

// Automatic Token Rotation
class TokenRotationManager {
  private rotationInterval: number;
  private rotationTimer: NodeJS.Timeout | null = null;
  
  constructor(intervalMinutes: number = 30) {
    this.rotationInterval = intervalMinutes * 60 * 1000;
  }
  
  startRotation(refreshCallback: () => Promise<void>): void {
    this.rotationTimer = setInterval(async () => {
      try {
        await refreshCallback();
        console.log('Token rotated successfully');
      } catch (error) {
        console.error('Token rotation failed:', error);
        this.stopRotation();
      }
    }, this.rotationInterval);
  }
  
  stopRotation(): void {
    if (this.rotationTimer) {
      clearInterval(this.rotationTimer);
      this.rotationTimer = null;
    }
  }
}
```

### 2. JWT Validation and Middleware

```typescript
// Express.js JWT Middleware
import { Request, Response, NextFunction } from 'express';

interface AuthenticatedRequest extends Request {
  user?: CustomPayload;
}

export const jwtAuthMiddleware = (jwtManager: SecureJWTManager) => {
  return async (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    try {
      const authHeader = req.headers.authorization;
      
      if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'Access token required' });
      }
      
      const token = authHeader.substring(7);
      const payload = await jwtManager.verifyAccessToken(token);
      
      req.user = payload;
      next();
    } catch (error) {
      return res.status(401).json({ error: error.message });
    }
  };
};

// Role-based authorization middleware
export const requireRole = (requiredRole: string) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }
    
    if (req.user.role !== requiredRole) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    
    next();
  };
};

// Permission-based authorization middleware
export const requirePermission = (requiredPermission: string) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }
    
    if (!req.user.permissions.includes(requiredPermission)) {
      return res.status(403).json({ error: 'Missing required permission' });
    }
    
    next();
  };
};
```

## Common JWT Vulnerabilities and Prevention

### 1. Algorithm Confusion Attack Prevention

```typescript
// Prevent algorithm confusion attacks
class SecureJWTValidator {
  private allowedAlgorithms: string[];
  
  constructor(allowedAlgorithms: string[]) {
    this.allowedAlgorithms = allowedAlgorithms;
  }
  
  validateToken(token: string, secret: string): CustomPayload {
    // Decode header without verification
    const decoded = decode(token, { complete: true });
    
    if (!decoded || !decoded.header) {
      throw new Error('Invalid token structure');
    }
    
    // Check if algorithm is in allowed list
    if (!this.allowedAlgorithms.includes(decoded.header.alg)) {
      throw new Error(`Algorithm ${decoded.header.alg} not allowed`);
    }
    
    // Verify with specific algorithm
    return verify(token, secret, {
      algorithms: this.allowedAlgorithms as any[]
    }) as CustomPayload;
  }
}
```

### 2. Token Hijacking Prevention

```typescript
// Token binding and fingerprinting
class TokenSecurityEnhancer {
  // Generate browser fingerprint
  static generateFingerprint(): string {
    const components = [
      navigator.userAgent,
      navigator.language,
      screen.width,
      screen.height,
      new Date().getTimezoneOffset(),
      navigator.platform
    ];
    
    return createHash('sha256')
      .update(components.join('|'))
      .digest('hex');
  }
  
  // Bind token to IP and fingerprint
  static createBoundToken(payload: CustomPayload, request: Request): string {
    const boundPayload = {
      ...payload,
      fingerprint: this.generateFingerprint(),
      ipAddress: this.getClientIP(request)
    };
    
    return sign(boundPayload, process.env.JWT_SECRET!);
  }
  
  // Validate token binding
  static validateBinding(token: string, request: Request): boolean {
    try {
      const decoded = decode(token) as any;
      
      if (!decoded.fingerprint || !decoded.ipAddress) {
        return false;
      }
      
      const currentFingerprint = this.generateFingerprint();
      const currentIP = this.getClientIP(request);
      
      return decoded.fingerprint === currentFingerprint &&
             decoded.ipAddress === currentIP;
    } catch {
      return false;
    }
  }
  
  private static getClientIP(request: Request): string {
    return request.ip || 
           request.connection.remoteAddress || 
           request.headers['x-forwarded-for'] as string ||
           '0.0.0.0';
  }
}
```

## Interview-Ready JWT Security Summary

**JWT Security Best Practices:**
1. **Secure Storage** - Use httpOnly cookies or in-memory storage, avoid localStorage
2. **Short Expiration** - Keep access tokens short-lived (15-30 minutes)
3. **Proper Algorithms** - Use RS256 for production, prevent algorithm confusion
4. **Token Rotation** - Implement refresh token rotation and revocation
5. **Validation** - Always validate issuer, audience, and expiration

**Common Vulnerabilities:**
1. **Algorithm Confusion** - Attackers change algorithm to bypass verification
2. **Token Storage** - XSS attacks stealing tokens from localStorage
3. **No Expiration** - Long-lived tokens increase security risk
4. **Insufficient Validation** - Missing issuer/audience checks
5. **Token Leakage** - Tokens exposed in URLs or logs

**Enterprise Features:**
- Token binding to prevent hijacking
- Automatic token rotation
- Centralized token revocation
- Role and permission-based authorization
- Comprehensive audit logging

**Key Interview Topics:** JWT structure and validation, secure token storage strategies, refresh token implementation, role-based access control, common JWT vulnerabilities and prevention, token revocation strategies, production security considerations.
