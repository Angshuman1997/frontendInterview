# How do you implement authentication and authorization in a web application

Authentication and authorization are fundamental security concepts in web applications. Authentication verifies who a user is, while authorization determines what they can access. This guide covers modern implementation patterns, security best practices, and common architectural approaches.

## Core Concepts

### Authentication vs Authorization

**Authentication (AuthN):** "Who are you?"
- Verifies user identity
- Mechanisms: passwords, biometrics, tokens, certificates
- Result: authenticated user identity

**Authorization (AuthZ):** "What can you do?"
- Determines user permissions
- Mechanisms: roles, permissions, policies, ACLs
- Result: access granted or denied

### Common Authentication Flows

**1. Session-Based Authentication:**
```typescript
// Traditional session-based flow
// 1. User submits credentials
// 2. Server validates credentials
// 3. Server creates session and stores session ID in cookie
// 4. Client sends cookie with each request
// 5. Server validates session on each request

// Express.js session example
import session from 'express-session';
import bcrypt from 'bcrypt';

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
  }
}));

// Login endpoint
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;
  
  try {
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    const isValidPassword = await bcrypt.compare(password, user.passwordHash);
    if (!isValidPassword) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Store user info in session
    req.session.userId = user.id;
    req.session.user = {
      id: user.id,
      email: user.email,
      roles: user.roles
    };

    res.json({ 
      user: { 
        id: user.id, 
        email: user.email, 
        roles: user.roles 
      } 
    });
  } catch (error) {
    res.status(500).json({ error: 'Authentication failed' });
  }
});

// Authentication middleware
const requireAuth = (req, res, next) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  next();
};

// Protected route
app.get('/api/protected', requireAuth, (req, res) => {
  res.json({ message: 'Protected data', user: req.session.user });
});
```

**2. Token-Based Authentication (JWT):**
```typescript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';

interface JWTPayload {
  userId: string;
  email: string;
  roles: string[];
  iat: number;
  exp: number;
}

// JWT configuration
const JWT_SECRET = process.env.JWT_SECRET!;
const JWT_EXPIRES_IN = '24h';
const REFRESH_TOKEN_EXPIRES_IN = '7d';

// Generate JWT tokens
const generateTokens = (user: User) => {
  const payload = {
    userId: user.id,
    email: user.email,
    roles: user.roles
  };

  const accessToken = jwt.sign(payload, JWT_SECRET, {
    expiresIn: JWT_EXPIRES_IN,
    issuer: 'your-app',
    audience: 'your-app-users'
  });

  const refreshToken = jwt.sign(
    { userId: user.id }, 
    JWT_SECRET, 
    { expiresIn: REFRESH_TOKEN_EXPIRES_IN }
  );

  return { accessToken, refreshToken };
};

// Login with JWT
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;

  try {
    const user = await User.findOne({ email });
    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    const { accessToken, refreshToken } = generateTokens(user);

    // Store refresh token securely
    await RefreshToken.create({
      token: refreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
    });

    // Set refresh token as httpOnly cookie
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.json({
      accessToken,
      user: {
        id: user.id,
        email: user.email,
        roles: user.roles
      }
    });
  } catch (error) {
    res.status(500).json({ error: 'Authentication failed' });
  }
});

// JWT verification middleware
const verifyToken = (req, res, next) => {
  const authHeader = req.headers.authorization;
  const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, JWT_SECRET, (err, decoded) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid token' });
    }

    req.user = decoded as JWTPayload;
    next();
  });
};

// Token refresh endpoint
app.post('/api/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;

  if (!refreshToken) {
    return res.status(401).json({ error: 'Refresh token required' });
  }

  try {
    const storedToken = await RefreshToken.findOne({ 
      token: refreshToken,
      expiresAt: { $gt: new Date() }
    });

    if (!storedToken) {
      return res.status(403).json({ error: 'Invalid refresh token' });
    }

    const decoded = jwt.verify(refreshToken, JWT_SECRET) as { userId: string };
    const user = await User.findById(decoded.userId);

    if (!user) {
      return res.status(403).json({ error: 'User not found' });
    }

    const { accessToken, refreshToken: newRefreshToken } = generateTokens(user);

    // Replace old refresh token
    await RefreshToken.deleteOne({ token: refreshToken });
    await RefreshToken.create({
      token: newRefreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
    });

    res.cookie('refreshToken', newRefreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.json({ accessToken });
  } catch (error) {
    res.status(403).json({ error: 'Token refresh failed' });
  }
});
```

## Authorization Patterns

### 1. Role-Based Access Control (RBAC)

```typescript
// User and role models
interface User {
  id: string;
  email: string;
  roles: Role[];
}

interface Role {
  id: string;
  name: string;
  permissions: Permission[];
}

interface Permission {
  id: string;
  resource: string;
  action: string; // create, read, update, delete
}

// Authorization middleware
const authorize = (requiredPermissions: string[]) => {
  return (req, res, next) => {
    const user = req.user;
    
    if (!user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const userPermissions = user.roles.flatMap(role => 
      role.permissions.map(permission => 
        `${permission.resource}:${permission.action}`
      )
    );

    const hasPermission = requiredPermissions.every(permission =>
      userPermissions.includes(permission)
    );

    if (!hasPermission) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
};

// Usage examples
app.get('/api/users', 
  verifyToken, 
  authorize(['users:read']), 
  getUsersController
);

app.post('/api/users', 
  verifyToken, 
  authorize(['users:create']), 
  createUserController
);

app.delete('/api/users/:id', 
  verifyToken, 
  authorize(['users:delete']), 
  deleteUserController
);

// Advanced authorization with resource ownership
const authorizeResource = (resource: string, action: string) => {
  return async (req, res, next) => {
    const user = req.user;
    const resourceId = req.params.id;

    // Check global permissions first
    const hasGlobalPermission = user.roles.some(role =>
      role.permissions.some(permission =>
        permission.resource === resource && permission.action === action
      )
    );

    if (hasGlobalPermission) {
      return next();
    }

    // Check resource ownership
    if (action === 'read' || action === 'update' || action === 'delete') {
      const resourceOwner = await getResourceOwner(resource, resourceId);
      
      if (resourceOwner === user.userId) {
        return next();
      }
    }

    res.status(403).json({ error: 'Insufficient permissions' });
  };
};

// Example: Users can only edit their own profiles
app.put('/api/users/:id', 
  verifyToken, 
  authorizeResource('users', 'update'), 
  updateUserController
);
```

### 2. Attribute-Based Access Control (ABAC)

```typescript
// ABAC policy engine
interface PolicyContext {
  user: User;
  resource: any;
  action: string;
  environment: {
    time: Date;
    ipAddress: string;
    userAgent: string;
  };
}

interface PolicyRule {
  name: string;
  effect: 'allow' | 'deny';
  conditions: PolicyCondition[];
}

interface PolicyCondition {
  attribute: string;
  operator: 'equals' | 'contains' | 'greaterThan' | 'lessThan' | 'in';
  value: any;
}

class PolicyEngine {
  private rules: PolicyRule[] = [];

  addRule(rule: PolicyRule) {
    this.rules.push(rule);
  }

  evaluate(context: PolicyContext): boolean {
    for (const rule of this.rules) {
      if (this.evaluateRule(rule, context)) {
        return rule.effect === 'allow';
      }
    }
    
    return false; // Default deny
  }

  private evaluateRule(rule: PolicyRule, context: PolicyContext): boolean {
    return rule.conditions.every(condition => 
      this.evaluateCondition(condition, context)
    );
  }

  private evaluateCondition(condition: PolicyCondition, context: PolicyContext): boolean {
    const value = this.getAttributeValue(condition.attribute, context);
    
    switch (condition.operator) {
      case 'equals':
        return value === condition.value;
      case 'contains':
        return Array.isArray(value) ? value.includes(condition.value) : false;
      case 'greaterThan':
        return value > condition.value;
      case 'lessThan':
        return value < condition.value;
      case 'in':
        return Array.isArray(condition.value) ? condition.value.includes(value) : false;
      default:
        return false;
    }
  }

  private getAttributeValue(attribute: string, context: PolicyContext): any {
    const parts = attribute.split('.');
    let value: any = context;
    
    for (const part of parts) {
      value = value?.[part];
    }
    
    return value;
  }
}

// Example ABAC policies
const policyEngine = new PolicyEngine();

// Allow managers to access reports during business hours
policyEngine.addRule({
  name: 'managers-business-hours',
  effect: 'allow',
  conditions: [
    { attribute: 'user.roles', operator: 'contains', value: 'manager' },
    { attribute: 'action', operator: 'equals', value: 'read' },
    { attribute: 'resource.type', operator: 'equals', value: 'report' },
    { attribute: 'environment.time', operator: 'greaterThan', value: '09:00' },
    { attribute: 'environment.time', operator: 'lessThan', value: '17:00' }
  ]
});

// ABAC middleware
const abacAuthorize = (req, res, next) => {
  const context: PolicyContext = {
    user: req.user,
    resource: req.body,
    action: getActionFromRequest(req),
    environment: {
      time: new Date(),
      ipAddress: req.ip,
      userAgent: req.get('User-Agent') || ''
    }
  };

  const isAllowed = policyEngine.evaluate(context);
  
  if (!isAllowed) {
    return res.status(403).json({ error: 'Access denied by policy' });
  }

  next();
};
```

## OAuth 2.0 and Social Authentication

### OAuth 2.0 Implementation

```typescript
import { OAuth2Client } from 'google-auth-library';
import axios from 'axios';

// Google OAuth setup
const googleClient = new OAuth2Client(
  process.env.GOOGLE_CLIENT_ID,
  process.env.GOOGLE_CLIENT_SECRET,
  process.env.GOOGLE_REDIRECT_URI
);

// OAuth flow initiation
app.get('/api/auth/google', (req, res) => {
  const authUrl = googleClient.generateAuthUrl({
    access_type: 'offline',
    scope: ['profile', 'email'],
    state: generateStateToken() // CSRF protection
  });
  
  res.redirect(authUrl);
});

// OAuth callback handler
app.get('/api/auth/google/callback', async (req, res) => {
  const { code, state } = req.query;
  
  try {
    // Verify state token to prevent CSRF
    if (!verifyStateToken(state)) {
      return res.status(400).json({ error: 'Invalid state parameter' });
    }

    // Exchange code for tokens
    const { tokens } = await googleClient.getToken(code);
    googleClient.setCredentials(tokens);

    // Get user info from Google
    const ticket = await googleClient.verifyIdToken({
      idToken: tokens.id_token!,
      audience: process.env.GOOGLE_CLIENT_ID
    });

    const payload = ticket.getPayload();
    if (!payload) {
      throw new Error('Invalid token payload');
    }

    // Find or create user
    let user = await User.findOne({ googleId: payload.sub });
    
    if (!user) {
      user = await User.create({
        googleId: payload.sub,
        email: payload.email,
        name: payload.name,
        avatar: payload.picture,
        provider: 'google'
      });
    }

    // Generate JWT tokens
    const { accessToken, refreshToken } = generateTokens(user);

    // Set secure cookie and redirect
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.redirect(`${process.env.CLIENT_URL}/auth/success?token=${accessToken}`);
  } catch (error) {
    res.redirect(`${process.env.CLIENT_URL}/auth/error`);
  }
});

// GitHub OAuth implementation
app.get('/api/auth/github', (req, res) => {
  const githubAuthUrl = `https://github.com/login/oauth/authorize?` +
    `client_id=${process.env.GITHUB_CLIENT_ID}&` +
    `scope=user:email&` +
    `state=${generateStateToken()}`;
    
  res.redirect(githubAuthUrl);
});

app.get('/api/auth/github/callback', async (req, res) => {
  const { code, state } = req.query;
  
  try {
    if (!verifyStateToken(state)) {
      return res.status(400).json({ error: 'Invalid state parameter' });
    }

    // Exchange code for access token
    const tokenResponse = await axios.post('https://github.com/login/oauth/access_token', {
      client_id: process.env.GITHUB_CLIENT_ID,
      client_secret: process.env.GITHUB_CLIENT_SECRET,
      code
    }, {
      headers: { Accept: 'application/json' }
    });

    const { access_token } = tokenResponse.data;

    // Get user info from GitHub
    const userResponse = await axios.get('https://api.github.com/user', {
      headers: { Authorization: `Bearer ${access_token}` }
    });

    const githubUser = userResponse.data;

    // Handle user creation/login similar to Google OAuth
    let user = await User.findOne({ githubId: githubUser.id });
    
    if (!user) {
      user = await User.create({
        githubId: githubUser.id,
        email: githubUser.email,
        name: githubUser.name,
        username: githubUser.login,
        avatar: githubUser.avatar_url,
        provider: 'github'
      });
    }

    const { accessToken: jwt, refreshToken } = generateTokens(user);

    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.redirect(`${process.env.CLIENT_URL}/auth/success?token=${jwt}`);
  } catch (error) {
    res.redirect(`${process.env.CLIENT_URL}/auth/error`);
  }
});
```

## Frontend Integration Patterns

### React Authentication Context

```typescript
// Auth context for React applications
interface AuthContextType {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  register: (userData: RegisterData) => Promise<void>;
  loading: boolean;
  error: string | null;
}

const AuthContext = createContext<AuthContextType | null>(null);

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

// Auth provider implementation
export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  // Check for existing session on mount
  useEffect(() => {
    const initializeAuth = async () => {
      try {
        const token = localStorage.getItem('accessToken');
        if (token) {
          const user = await validateToken(token);
          setUser(user);
        }
      } catch (error) {
        localStorage.removeItem('accessToken');
      } finally {
        setLoading(false);
      }
    };

    initializeAuth();
  }, []);

  const login = async (email: string, password: string) => {
    try {
      setLoading(true);
      setError(null);

      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password }),
        credentials: 'include'
      });

      if (!response.ok) {
        throw new Error('Login failed');
      }

      const { accessToken, user } = await response.json();
      
      localStorage.setItem('accessToken', accessToken);
      setUser(user);
    } catch (error) {
      setError(error.message);
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const logout = async () => {
    try {
      await fetch('/api/auth/logout', {
        method: 'POST',
        credentials: 'include'
      });
    } catch (error) {
      console.error('Logout error:', error);
    } finally {
      localStorage.removeItem('accessToken');
      setUser(null);
    }
  };

  const register = async (userData: RegisterData) => {
    try {
      setLoading(true);
      setError(null);

      const response = await fetch('/api/auth/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData)
      });

      if (!response.ok) {
        throw new Error('Registration failed');
      }

      const { accessToken, user } = await response.json();
      
      localStorage.setItem('accessToken', accessToken);
      setUser(user);
    } catch (error) {
      setError(error.message);
      throw error;
    } finally {
      setLoading(false);
    }
  };

  return (
    <AuthContext.Provider value={{
      user,
      login,
      logout,
      register,
      loading,
      error
    }}>
      {children}
    </AuthContext.Provider>
  );
};

// Protected route component
export const ProtectedRoute: React.FC<{
  children: React.ReactNode;
  requiredRoles?: string[];
}> = ({ children, requiredRoles = [] }) => {
  const { user, loading } = useAuth();
  const location = useLocation();

  if (loading) {
    return <LoadingSpinner />;
  }

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (requiredRoles.length > 0) {
    const hasRequiredRole = requiredRoles.some(role => 
      user.roles.includes(role)
    );
    
    if (!hasRequiredRole) {
      return <Navigate to="/unauthorized" replace />;
    }
  }

  return <>{children}</>;
};

// Axios interceptor for automatic token handling
const setupAxiosInterceptors = () => {
  // Request interceptor to add auth token
  axios.interceptors.request.use(
    (config) => {
      const token = localStorage.getItem('accessToken');
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    },
    (error) => Promise.reject(error)
  );

  // Response interceptor to handle token refresh
  axios.interceptors.response.use(
    (response) => response,
    async (error) => {
      const originalRequest = error.config;

      if (error.response?.status === 401 && !originalRequest._retry) {
        originalRequest._retry = true;

        try {
          const response = await fetch('/api/auth/refresh', {
            method: 'POST',
            credentials: 'include'
          });

          if (response.ok) {
            const { accessToken } = await response.json();
            localStorage.setItem('accessToken', accessToken);
            originalRequest.headers.Authorization = `Bearer ${accessToken}`;
            return axios(originalRequest);
          }
        } catch (refreshError) {
          localStorage.removeItem('accessToken');
          window.location.href = '/login';
        }
      }

      return Promise.reject(error);
    }
  );
};
```

## Security Best Practices

### 1. Password Security

```typescript
import bcrypt from 'bcrypt';
import zxcvbn from 'zxcvbn';

// Password hashing
const SALT_ROUNDS = 12;

const hashPassword = async (password: string): Promise<string> => {
  return bcrypt.hash(password, SALT_ROUNDS);
};

const verifyPassword = async (password: string, hash: string): Promise<boolean> => {
  return bcrypt.compare(password, hash);
};

// Password strength validation
const validatePasswordStrength = (password: string) => {
  const result = zxcvbn(password);
  
  return {
    score: result.score, // 0-4 (weak to strong)
    feedback: result.feedback,
    isStrong: result.score >= 3,
    crackTimeDisplay: result.crack_times_display.offline_slow_hashing_1e4_per_second
  };
};

// Registration with password validation
app.post('/api/auth/register', async (req, res) => {
  const { email, password, name } = req.body;

  try {
    // Validate password strength
    const passwordAnalysis = validatePasswordStrength(password);
    if (!passwordAnalysis.isStrong) {
      return res.status(400).json({
        error: 'Password is too weak',
        feedback: passwordAnalysis.feedback
      });
    }

    // Check if user already exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(409).json({ error: 'User already exists' });
    }

    // Hash password and create user
    const passwordHash = await hashPassword(password);
    const user = await User.create({
      email,
      name,
      passwordHash
    });

    const { accessToken, refreshToken } = generateTokens(user);

    res.status(201).json({
      accessToken,
      user: {
        id: user.id,
        email: user.email,
        name: user.name
      }
    });
  } catch (error) {
    res.status(500).json({ error: 'Registration failed' });
  }
});
```

### 2. Rate Limiting and Brute Force Protection

```typescript
import rateLimit from 'express-rate-limit';
import slowDown from 'express-slow-down';

// General rate limiting
const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});

// Strict rate limiting for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // limit each IP to 5 requests per windowMs
  message: 'Too many authentication attempts',
  skipSuccessfulRequests: true,
});

// Progressive delay for repeated requests
const speedLimiter = slowDown({
  windowMs: 15 * 60 * 1000, // 15 minutes
  delayAfter: 2, // allow 2 requests per 15 minutes at full speed
  delayMs: 500 // slow down subsequent requests by 500ms per request
});

// Account lockout mechanism
class AccountLockout {
  private attempts = new Map<string, { count: number; lastAttempt: Date }>();
  private readonly MAX_ATTEMPTS = 5;
  private readonly LOCKOUT_DURATION = 30 * 60 * 1000; // 30 minutes

  async recordFailedAttempt(identifier: string): Promise<void> {
    const now = new Date();
    const attempts = this.attempts.get(identifier) || { count: 0, lastAttempt: now };

    // Reset count if lockout period has passed
    if (now.getTime() - attempts.lastAttempt.getTime() > this.LOCKOUT_DURATION) {
      attempts.count = 0;
    }

    attempts.count++;
    attempts.lastAttempt = now;
    this.attempts.set(identifier, attempts);

    // Store in database for persistence
    await FailedAttempt.upsert({
      identifier,
      count: attempts.count,
      lastAttempt: now
    });
  }

  async isLocked(identifier: string): Promise<boolean> {
    const attempts = this.attempts.get(identifier);
    if (!attempts) return false;

    const now = new Date();
    const timeSinceLastAttempt = now.getTime() - attempts.lastAttempt.getTime();

    return attempts.count >= this.MAX_ATTEMPTS && timeSinceLastAttempt < this.LOCKOUT_DURATION;
  }

  async clearAttempts(identifier: string): Promise<void> {
    this.attempts.delete(identifier);
    await FailedAttempt.destroy({ where: { identifier } });
  }
}

const accountLockout = new AccountLockout();

// Apply middleware to auth routes
app.use('/api/auth', generalLimiter);
app.use('/api/auth/login', authLimiter, speedLimiter);

// Enhanced login with lockout protection
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;
  const identifier = `${email}:${req.ip}`;

  try {
    // Check if account is locked
    if (await accountLockout.isLocked(identifier)) {
      return res.status(429).json({ 
        error: 'Account temporarily locked due to too many failed attempts' 
      });
    }

    const user = await User.findOne({ email });
    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
      await accountLockout.recordFailedAttempt(identifier);
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Clear failed attempts on successful login
    await accountLockout.clearAttempts(identifier);

    const { accessToken, refreshToken } = generateTokens(user);
    
    res.json({
      accessToken,
      user: {
        id: user.id,
        email: user.email,
        roles: user.roles
      }
    });
  } catch (error) {
    res.status(500).json({ error: 'Authentication failed' });
  }
});
```

## Interview-Ready Summary

**Authentication & Authorization implementation requires:**

1. **Authentication Methods** - Session-based, JWT tokens, OAuth 2.0, multi-factor authentication
2. **Authorization Patterns** - RBAC (role-based), ABAC (attribute-based), resource ownership
3. **Security Measures** - Password hashing, rate limiting, account lockout, CSRF protection
4. **Token Management** - Secure storage, refresh mechanisms, proper expiration
5. **Frontend Integration** - Auth context, protected routes, automatic token refresh

**Key security principles:**
- **Defense in depth** - Multiple security layers
- **Principle of least privilege** - Minimal required permissions
- **Secure by default** - Safe default configurations
- **Regular security audits** - Continuous vulnerability assessment

**Common vulnerabilities to prevent:** CSRF attacks, XSS injection, JWT token theft, session fixation, brute force attacks, privilege escalation.

**Best practices:** Use HTTPS everywhere, implement proper CORS, validate all inputs, log security events, regular security testing, and keep dependencies updated.