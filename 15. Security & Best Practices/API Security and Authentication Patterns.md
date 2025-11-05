# API Security and Authentication Patterns

Modern frontend applications heavily rely on API interactions, making API security a critical concern. This guide covers comprehensive strategies for securing API communications, implementing robust authentication flows, and handling security vulnerabilities specific to frontend-backend interactions.

## API Security Fundamentals

### Secure API Client Implementation

```typescript
// Secure API Client with Built-in Security Features
interface APIClientConfig {
  baseURL: string;
  timeout: number;
  retryAttempts: number;
  apiKey?: string;
  csrfProtection: boolean;
  requestEncryption: boolean;
  rateLimiting: boolean;
}

interface SecurityHeaders {
  'X-Request-ID': string;
  'X-Client-Version': string;
  'X-Timestamp': string;
  'X-Signature'?: string;
  'Authorization'?: string;
  'X-CSRF-Token'?: string;
}

class SecureAPIClient {
  private config: APIClientConfig;
  private requestQueue: Map<string, AbortController> = new Map();
  private rateLimiter: RateLimiter;
  private csrfToken: string | null = null;

  constructor(config: APIClientConfig) {
    this.config = config;
    this.rateLimiter = new RateLimiter(100, 60000); // 100 requests per minute
  }

  // Main request method with security features
  async request<T>(
    endpoint: string,
    options: RequestInit & {
      requireAuth?: boolean;
      skipCSRF?: boolean;
      priority?: 'high' | 'normal' | 'low';
      encrypt?: boolean;
    } = {}
  ): Promise<T> {
    const {
      requireAuth = true,
      skipCSRF = false,
      priority = 'normal',
      encrypt = false,
      ...fetchOptions
    } = options;

    // Rate limiting check
    if (this.config.rateLimiting && !this.rateLimiter.isAllowed()) {
      throw new Error('Rate limit exceeded');
    }

    // Generate unique request ID
    const requestId = this.generateRequestId();
    
    // Create abort controller for request cancellation
    const abortController = new AbortController();
    this.requestQueue.set(requestId, abortController);

    try {
      // Build secure headers
      const headers = await this.buildSecureHeaders(requireAuth, !skipCSRF);
      
      // Prepare request body
      let body = fetchOptions.body;
      if (encrypt && body) {
        body = await this.encryptRequestBody(body);
        headers['Content-Type'] = 'application/octet-stream';
        headers['X-Encrypted'] = 'true';
      }

      // Make the request
      const response = await fetch(`${this.config.baseURL}${endpoint}`, {
        ...fetchOptions,
        headers: { ...headers, ...fetchOptions.headers },
        body,
        signal: abortController.signal,
        timeout: this.config.timeout
      });

      // Validate response
      await this.validateResponse(response);

      // Parse response
      const data = await this.parseResponse<T>(response);
      
      // Log successful request for audit
      this.logAPIRequest(requestId, endpoint, 'success');
      
      return data;

    } catch (error) {
      this.logAPIRequest(requestId, endpoint, 'error', error);
      throw this.handleAPIError(error);
    } finally {
      this.requestQueue.delete(requestId);
    }
  }

  // Specialized HTTP methods
  async get<T>(endpoint: string, options?: RequestInit): Promise<T> {
    return this.request<T>(endpoint, { ...options, method: 'GET' });
  }

  async post<T>(endpoint: string, data?: any, options?: RequestInit): Promise<T> {
    return this.request<T>(endpoint, {
      ...options,
      method: 'POST',
      body: data ? JSON.stringify(data) : undefined,
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers
      }
    });
  }

  async put<T>(endpoint: string, data?: any, options?: RequestInit): Promise<T> {
    return this.request<T>(endpoint, {
      ...options,
      method: 'PUT',
      body: data ? JSON.stringify(data) : undefined,
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers
      }
    });
  }

  async delete<T>(endpoint: string, options?: RequestInit): Promise<T> {
    return this.request<T>(endpoint, { ...options, method: 'DELETE' });
  }

  // Cancel all pending requests
  cancelAllRequests(): void {
    this.requestQueue.forEach(controller => controller.abort());
    this.requestQueue.clear();
  }

  // Cancel specific request
  cancelRequest(requestId: string): void {
    const controller = this.requestQueue.get(requestId);
    if (controller) {
      controller.abort();
      this.requestQueue.delete(requestId);
    }
  }

  // Private helper methods
  private async buildSecureHeaders(requireAuth: boolean, includeCSRF: boolean): Promise<SecurityHeaders> {
    const headers: SecurityHeaders = {
      'X-Request-ID': this.generateRequestId(),
      'X-Client-Version': process.env.REACT_APP_VERSION || '1.0.0',
      'X-Timestamp': new Date().toISOString()
    };

    // Add authentication header
    if (requireAuth) {
      const token = await this.getAuthToken();
      if (token) {
        headers['Authorization'] = `Bearer ${token}`;
      } else {
        throw new Error('Authentication required but no token available');
      }
    }

    // Add CSRF protection
    if (includeCSRF && this.config.csrfProtection) {
      const csrfToken = await this.getCSRFToken();
      if (csrfToken) {
        headers['X-CSRF-Token'] = csrfToken;
      }
    }

    // Add request signature for integrity
    const signature = await this.signRequest(headers);
    headers['X-Signature'] = signature;

    return headers;
  }

  private async getAuthToken(): Promise<string | null> {
    // Get token from secure storage
    return await SecureStorage.getItem('auth_token');
  }

  private async getCSRFToken(): Promise<string | null> {
    if (!this.csrfToken) {
      try {
        const response = await fetch(`${this.config.baseURL}/api/csrf-token`);
        const data = await response.json();
        this.csrfToken = data.token;
      } catch (error) {
        console.error('Failed to get CSRF token:', error);
      }
    }
    return this.csrfToken;
  }

  private async signRequest(headers: SecurityHeaders): Promise<string> {
    // Create signature for request integrity
    const signatureData = `${headers['X-Request-ID']}:${headers['X-Timestamp']}`;
    return await ClientCrypto.hash(signatureData);
  }

  private async encryptRequestBody(body: BodyInit): Promise<string> {
    if (typeof body === 'string') {
      const key = await ClientCrypto.generateKey();
      const { encrypted, iv } = await ClientCrypto.encrypt(body, key);
      return JSON.stringify({
        encrypted: Array.from(new Uint8Array(encrypted)),
        iv: Array.from(iv)
      });
    }
    return body as string;
  }

  private async validateResponse(response: Response): Promise<void> {
    // Check for security headers
    const securityHeaders = [
      'X-Content-Type-Options',
      'X-Frame-Options',
      'X-XSS-Protection',
      'Strict-Transport-Security'
    ];

    securityHeaders.forEach(header => {
      if (!response.headers.get(header)) {
        console.warn(`Missing security header: ${header}`);
      }
    });

    // Validate response status
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      throw new APIError(response.status, errorData.message || 'Request failed', errorData);
    }
  }

  private async parseResponse<T>(response: Response): Promise<T> {
    const contentType = response.headers.get('Content-Type');
    
    if (contentType?.includes('application/json')) {
      return await response.json();
    } else if (contentType?.includes('text/')) {
      return await response.text() as unknown as T;
    } else {
      return await response.blob() as unknown as T;
    }
  }

  private generateRequestId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  private handleAPIError(error: any): APIError {
    if (error instanceof APIError) {
      return error;
    }

    if (error.name === 'AbortError') {
      return new APIError(499, 'Request cancelled');
    }

    if (error.name === 'TimeoutError') {
      return new APIError(408, 'Request timeout');
    }

    return new APIError(500, 'Unknown error occurred', error);
  }

  private logAPIRequest(requestId: string, endpoint: string, status: 'success' | 'error', error?: any): void {
    const logData = {
      requestId,
      endpoint,
      status,
      timestamp: new Date().toISOString(),
      error: error?.message
    };

    // In production, send to logging service
    console.log('API Request Log:', logData);
  }
}

// Custom Error Class
class APIError extends Error {
  constructor(
    public status: number,
    message: string,
    public data?: any
  ) {
    super(message);
    this.name = 'APIError';
  }
}

// Rate Limiter Implementation
class RateLimiter {
  private requests: number[] = [];
  
  constructor(private limit: number, private windowMs: number) {}

  isAllowed(): boolean {
    const now = Date.now();
    // Remove old requests outside the window
    this.requests = this.requests.filter(time => now - time < this.windowMs);
    
    if (this.requests.length >= this.limit) {
      return false;
    }
    
    this.requests.push(now);
    return true;
  }

  getRemaining(): number {
    const now = Date.now();
    this.requests = this.requests.filter(time => now - time < this.windowMs);
    return Math.max(0, this.limit - this.requests.length);
  }
}
```

## Advanced Authentication Patterns

### Multi-Factor Authentication (MFA) Implementation

```typescript
// MFA Authentication Flow
interface MFAConfig {
  methods: ('sms' | 'email' | 'totp' | 'backup')[];
  required: boolean;
  gracePeriod: number; // minutes
}

interface MFAChallenge {
  id: string;
  method: string;
  expiresAt: string;
  attempts: number;
  maxAttempts: number;
}

class MFAManager {
  private challenges = new Map<string, MFAChallenge>();
  
  // Initiate MFA challenge
  async initiateMFA(userId: string, method: string): Promise<MFAChallenge> {
    const challengeId = this.generateChallengeId();
    const expiresAt = new Date(Date.now() + 5 * 60 * 1000).toISOString(); // 5 minutes
    
    const challenge: MFAChallenge = {
      id: challengeId,
      method,
      expiresAt,
      attempts: 0,
      maxAttempts: 3
    };

    this.challenges.set(challengeId, challenge);

    try {
      switch (method) {
        case 'sms':
          await this.sendSMSChallenge(userId, challengeId);
          break;
        case 'email':
          await this.sendEmailChallenge(userId, challengeId);
          break;
        case 'totp':
          // TOTP doesn't need server-side challenge
          break;
        default:
          throw new Error(`Unsupported MFA method: ${method}`);
      }

      return challenge;
    } catch (error) {
      this.challenges.delete(challengeId);
      throw error;
    }
  }

  // Verify MFA challenge
  async verifyMFA(challengeId: string, code: string, method: string): Promise<boolean> {
    const challenge = this.challenges.get(challengeId);
    
    if (!challenge) {
      throw new Error('Invalid or expired challenge');
    }

    if (new Date() > new Date(challenge.expiresAt)) {
      this.challenges.delete(challengeId);
      throw new Error('Challenge expired');
    }

    if (challenge.attempts >= challenge.maxAttempts) {
      this.challenges.delete(challengeId);
      throw new Error('Maximum attempts exceeded');
    }

    challenge.attempts++;

    let isValid = false;
    
    try {
      switch (method) {
        case 'sms':
        case 'email':
          isValid = await this.verifyCodeChallenge(challengeId, code);
          break;
        case 'totp':
          isValid = await this.verifyTOTPCode(code);
          break;
        default:
          throw new Error(`Unsupported MFA method: ${method}`);
      }

      if (isValid) {
        this.challenges.delete(challengeId);
        return true;
      }

      return false;
    } catch (error) {
      console.error('MFA verification error:', error);
      return false;
    }
  }

  // Generate backup codes
  generateBackupCodes(count: number = 10): string[] {
    const codes: string[] = [];
    
    for (let i = 0; i < count; i++) {
      // Generate 8-character alphanumeric codes
      const code = Array.from({ length: 8 }, () => 
        '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'[Math.floor(Math.random() * 36)]
      ).join('');
      codes.push(code);
    }
    
    return codes;
  }

  private generateChallengeId(): string {
    return Array.from({ length: 16 }, () => 
      '0123456789abcdef'[Math.floor(Math.random() * 16)]
    ).join('');
  }

  private async sendSMSChallenge(userId: string, challengeId: string): Promise<void> {
    // Implementation would integrate with SMS service
    console.log(`SMS challenge sent for user ${userId}, challenge ${challengeId}`);
  }

  private async sendEmailChallenge(userId: string, challengeId: string): Promise<void> {
    // Implementation would integrate with email service
    console.log(`Email challenge sent for user ${userId}, challenge ${challengeId}`);
  }

  private async verifyCodeChallenge(challengeId: string, code: string): Promise<boolean> {
    // Implementation would verify against stored challenge code
    return code.length === 6 && /^\d+$/.test(code);
  }

  private async verifyTOTPCode(code: string): Promise<boolean> {
    // Implementation would verify TOTP code against user's secret
    return code.length === 6 && /^\d+$/.test(code);
  }
}

// React Hook for MFA
export const useMFA = () => {
  const [mfaManager] = useState(() => new MFAManager());
  const [currentChallenge, setCurrentChallenge] = useState<MFAChallenge | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const initiateMFA = async (userId: string, method: string) => {
    setIsLoading(true);
    try {
      const challenge = await mfaManager.initiateMFA(userId, method);
      setCurrentChallenge(challenge);
      return challenge;
    } catch (error) {
      console.error('Failed to initiate MFA:', error);
      throw error;
    } finally {
      setIsLoading(false);
    }
  };

  const verifyMFA = async (code: string) => {
    if (!currentChallenge) {
      throw new Error('No active MFA challenge');
    }

    setIsLoading(true);
    try {
      const isValid = await mfaManager.verifyMFA(
        currentChallenge.id,
        code,
        currentChallenge.method
      );
      
      if (isValid) {
        setCurrentChallenge(null);
      }
      
      return isValid;
    } catch (error) {
      console.error('MFA verification failed:', error);
      throw error;
    } finally {
      setIsLoading(false);
    }
  };

  const cancelMFA = () => {
    setCurrentChallenge(null);
  };

  return {
    currentChallenge,
    isLoading,
    initiateMFA,
    verifyMFA,
    cancelMFA,
    generateBackupCodes: mfaManager.generateBackupCodes.bind(mfaManager)
  };
};
```

## OAuth 2.0 and OpenID Connect Implementation

### Secure OAuth Flow

```typescript
// OAuth 2.0 Implementation with PKCE
interface OAuthConfig {
  clientId: string;
  redirectUri: string;
  scope: string[];
  responseType: 'code';
  authorizationEndpoint: string;
  tokenEndpoint: string;
  userInfoEndpoint: string;
  usePKCE: boolean;
}

interface PKCEChallenge {
  codeVerifier: string;
  codeChallenge: string;
  codeChallengeMethod: 'S256';
}

interface TokenResponse {
  accessToken: string;
  refreshToken?: string;
  idToken?: string;
  tokenType: string;
  expiresIn: number;
  scope: string;
}

class OAuthClient {
  private config: OAuthConfig;
  private currentPKCE: PKCEChallenge | null = null;

  constructor(config: OAuthConfig) {
    this.config = config;
  }

  // Initiate OAuth authorization flow
  async authorize(): Promise<void> {
    const state = this.generateRandomString(32);
    const nonce = this.generateRandomString(32);
    
    // Store state and nonce for validation
    sessionStorage.setItem('oauth_state', state);
    sessionStorage.setItem('oauth_nonce', nonce);

    const params = new URLSearchParams({
      client_id: this.config.clientId,
      redirect_uri: this.config.redirectUri,
      response_type: this.config.responseType,
      scope: this.config.scope.join(' '),
      state,
      nonce
    });

    // Add PKCE challenge if enabled
    if (this.config.usePKCE) {
      this.currentPKCE = await this.generatePKCEChallenge();
      params.append('code_challenge', this.currentPKCE.codeChallenge);
      params.append('code_challenge_method', this.currentPKCE.codeChallengeMethod);
      
      // Store code verifier securely
      await SecureStorage.setItem('oauth_code_verifier', this.currentPKCE.codeVerifier);
    }

    // Redirect to authorization server
    const authUrl = `${this.config.authorizationEndpoint}?${params.toString()}`;
    window.location.href = authUrl;
  }

  // Handle authorization callback
  async handleCallback(callbackUrl: string): Promise<TokenResponse> {
    const url = new URL(callbackUrl);
    const code = url.searchParams.get('code');
    const state = url.searchParams.get('state');
    const error = url.searchParams.get('error');

    // Check for errors
    if (error) {
      throw new Error(`OAuth error: ${error}`);
    }

    // Validate state parameter
    const storedState = sessionStorage.getItem('oauth_state');
    if (!state || state !== storedState) {
      throw new Error('Invalid state parameter');
    }

    if (!code) {
      throw new Error('Authorization code not found');
    }

    // Exchange code for tokens
    return await this.exchangeCodeForTokens(code);
  }

  // Exchange authorization code for tokens
  private async exchangeCodeForTokens(code: string): Promise<TokenResponse> {
    const body = new URLSearchParams({
      grant_type: 'authorization_code',
      client_id: this.config.clientId,
      redirect_uri: this.config.redirectUri,
      code
    });

    // Add PKCE code verifier if used
    if (this.config.usePKCE) {
      const codeVerifier = await SecureStorage.getItem<string>('oauth_code_verifier');
      if (codeVerifier) {
        body.append('code_verifier', codeVerifier);
      }
    }

    const response = await fetch(this.config.tokenEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Accept': 'application/json'
      },
      body
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Token exchange failed: ${errorData.error_description || errorData.error}`);
    }

    const tokenData = await response.json();
    
    // Validate ID token if present
    if (tokenData.id_token) {
      await this.validateIdToken(tokenData.id_token);
    }

    // Clean up stored values
    sessionStorage.removeItem('oauth_state');
    sessionStorage.removeItem('oauth_nonce');
    await SecureStorage.removeItem('oauth_code_verifier');

    return {
      accessToken: tokenData.access_token,
      refreshToken: tokenData.refresh_token,
      idToken: tokenData.id_token,
      tokenType: tokenData.token_type || 'Bearer',
      expiresIn: tokenData.expires_in,
      scope: tokenData.scope
    };
  }

  // Refresh access token
  async refreshToken(refreshToken: string): Promise<TokenResponse> {
    const body = new URLSearchParams({
      grant_type: 'refresh_token',
      client_id: this.config.clientId,
      refresh_token: refreshToken
    });

    const response = await fetch(this.config.tokenEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Accept': 'application/json'
      },
      body
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Token refresh failed: ${errorData.error_description || errorData.error}`);
    }

    const tokenData = await response.json();

    return {
      accessToken: tokenData.access_token,
      refreshToken: tokenData.refresh_token || refreshToken, // Keep old refresh token if new one not provided
      idToken: tokenData.id_token,
      tokenType: tokenData.token_type || 'Bearer',
      expiresIn: tokenData.expires_in,
      scope: tokenData.scope
    };
  }

  // Get user info using access token
  async getUserInfo(accessToken: string): Promise<any> {
    const response = await fetch(this.config.userInfoEndpoint, {
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Accept': 'application/json'
      }
    });

    if (!response.ok) {
      throw new Error('Failed to get user info');
    }

    return await response.json();
  }

  // Generate PKCE challenge
  private async generatePKCEChallenge(): Promise<PKCEChallenge> {
    const codeVerifier = this.generateRandomString(128);
    const encoder = new TextEncoder();
    const data = encoder.encode(codeVerifier);
    const digest = await window.crypto.subtle.digest('SHA-256', data);
    const codeChallenge = btoa(String.fromCharCode(...new Uint8Array(digest)))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=/g, '');

    return {
      codeVerifier,
      codeChallenge,
      codeChallengeMethod: 'S256'
    };
  }

  // Validate ID token (basic validation)
  private async validateIdToken(idToken: string): Promise<void> {
    const parts = idToken.split('.');
    if (parts.length !== 3) {
      throw new Error('Invalid ID token format');
    }

    const payload = JSON.parse(atob(parts[1]));
    const storedNonce = sessionStorage.getItem('oauth_nonce');

    // Validate nonce
    if (payload.nonce !== storedNonce) {
      throw new Error('Invalid nonce in ID token');
    }

    // Validate audience
    if (payload.aud !== this.config.clientId) {
      throw new Error('Invalid audience in ID token');
    }

    // Validate expiration
    if (payload.exp < Date.now() / 1000) {
      throw new Error('ID token has expired');
    }

    // Additional validations would include signature verification
    // which requires the provider's public keys
  }

  private generateRandomString(length: number): string {
    const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~';
    let result = '';
    const randomValues = new Uint8Array(length);
    window.crypto.getRandomValues(randomValues);
    
    for (let i = 0; i < length; i++) {
      result += charset[randomValues[i] % charset.length];
    }
    
    return result;
  }
}

// React Hook for OAuth
export const useOAuth = (config: OAuthConfig) => {
  const [oauthClient] = useState(() => new OAuthClient(config));
  const [isLoading, setIsLoading] = useState(false);
  const [tokens, setTokens] = useState<TokenResponse | null>(null);

  const login = async () => {
    setIsLoading(true);
    try {
      await oauthClient.authorize();
    } catch (error) {
      console.error('OAuth login failed:', error);
      setIsLoading(false);
      throw error;
    }
  };

  const handleCallback = async (callbackUrl: string) => {
    setIsLoading(true);
    try {
      const tokenResponse = await oauthClient.handleCallback(callbackUrl);
      setTokens(tokenResponse);
      
      // Store tokens securely
      await SecureStorage.setItem('access_token', tokenResponse.accessToken);
      if (tokenResponse.refreshToken) {
        await SecureStorage.setItem('refresh_token', tokenResponse.refreshToken);
      }
      
      return tokenResponse;
    } catch (error) {
      console.error('OAuth callback handling failed:', error);
      throw error;
    } finally {
      setIsLoading(false);
    }
  };

  const refreshTokens = async () => {
    if (!tokens?.refreshToken) {
      throw new Error('No refresh token available');
    }

    setIsLoading(true);
    try {
      const newTokens = await oauthClient.refreshToken(tokens.refreshToken);
      setTokens(newTokens);
      
      // Update stored tokens
      await SecureStorage.setItem('access_token', newTokens.accessToken);
      if (newTokens.refreshToken) {
        await SecureStorage.setItem('refresh_token', newTokens.refreshToken);
      }
      
      return newTokens;
    } catch (error) {
      console.error('Token refresh failed:', error);
      throw error;
    } finally {
      setIsLoading(false);
    }
  };

  const logout = async () => {
    setTokens(null);
    await SecureStorage.removeItem('access_token');
    await SecureStorage.removeItem('refresh_token');
  };

  return {
    isLoading,
    tokens,
    login,
    handleCallback,
    refreshTokens,
    logout,
    getUserInfo: (accessToken: string) => oauthClient.getUserInfo(accessToken)
  };
};
```

## Interview-Ready API Security Summary

**Core API Security Principles:**
1. **Authentication & Authorization** - Implement robust token-based auth with MFA
2. **Request Signing** - Sign requests for integrity verification
3. **Rate Limiting** - Prevent abuse with client-side rate limiting
4. **Encryption** - Encrypt sensitive request/response data
5. **CSRF Protection** - Include CSRF tokens in state-changing requests

**Advanced Authentication Patterns:**
- OAuth 2.0 with PKCE for secure authorization
- Multi-factor authentication with multiple methods
- JWT token management with refresh strategies
- Secure token storage and rotation

**API Client Security Features:**
- Request/response encryption
- Automatic retry with exponential backoff
- Request cancellation and timeout handling
- Comprehensive error handling and logging
- Security header validation

**Key Interview Topics:** OAuth 2.0/OIDC flows, PKCE implementation, MFA strategies, API security best practices, token management, request signing, rate limiting, secure storage patterns.