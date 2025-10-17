# How do you handle errors in React

This file covers comprehensive error handling strategies in React applications.
Essential for building robust, user-friendly applications that gracefully handle failures.

---

## Types of errors in React

### JavaScript Errors
* Runtime errors in component logic
* Type errors and null reference errors
* API call failures
* Third-party library errors

### React-specific Errors
* Component rendering errors
* Lifecycle method errors
* Hook dependency errors
* Key prop warnings

---

## Error Boundaries

### Basic Error Boundary

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null, errorInfo: null };
  }

  static getDerivedStateFromError(error) {
    // Update state so the next render shows the fallback UI
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // Log error details
    console.error('Error caught by boundary:', error, errorInfo);
    
    // Update state with error details
    this.setState({
      error,
      errorInfo
    });
    
    // Send error to monitoring service
    this.logErrorToService(error, errorInfo);
  }

  logErrorToService = (error, errorInfo) => {
    // Send to error tracking service like Sentry, LogRocket, etc.
    if (window.Sentry) {
      window.Sentry.captureException(error, {
        contexts: {
          react: {
            componentStack: errorInfo.componentStack
          }
        }
      });
    }
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            <summary>Error details (click to expand)</summary>
            <p><strong>Error:</strong> {this.state.error && this.state.error.toString()}</p>
            <p><strong>Component Stack:</strong></p>
            <code>{this.state.errorInfo.componentStack}</code>
          </details>
          <button onClick={() => window.location.reload()}>
            Reload Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <ErrorBoundary>
      <Header />
      <MainContent />
      <Footer />
    </ErrorBoundary>
  );
}
```

### Advanced Error Boundary with Recovery

```jsx
class RecoverableErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { 
      hasError: false, 
      error: null, 
      errorId: null,
      retryCount: 0 
    };
  }

  static getDerivedStateFromError(error) {
    return { 
      hasError: true, 
      error,
      errorId: Date.now() // Unique ID for this error
    };
  }

  componentDidCatch(error, errorInfo) {
    // Track retry attempts
    this.setState(prevState => ({
      retryCount: prevState.retryCount + 1
    }));

    // Log with additional context
    const errorReport = {
      error: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      retryCount: this.state.retryCount,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent,
      url: window.location.href
    };

    this.props.onError?.(errorReport);
    
    // Auto-retry for certain types of errors
    if (this.shouldAutoRetry(error) && this.state.retryCount < 3) {
      setTimeout(() => {
        this.handleRetry();
      }, 1000 * this.state.retryCount); // Exponential backoff
    }
  }

  shouldAutoRetry = (error) => {
    // Auto-retry for network errors or temporary failures
    return error.message.includes('Network') || 
           error.message.includes('fetch') ||
           error.name === 'ChunkLoadError';
  };

  handleRetry = () => {
    this.setState({ 
      hasError: false, 
      error: null,
      errorId: null 
    });
  };

  render() {
    if (this.state.hasError) {
      const { fallback: Fallback, children } = this.props;
      
      if (Fallback) {
        return (
          <Fallback 
            error={this.state.error}
            retry={this.handleRetry}
            retryCount={this.state.retryCount}
          />
        );
      }

      return (
        <div className="error-boundary">
          <h2>Oops! Something went wrong</h2>
          <p>Don't worry, we're on it. You can try refreshing the page.</p>
          
          {this.state.retryCount < 3 && (
            <button onClick={this.handleRetry}>
              Try Again {this.state.retryCount > 0 && `(${this.state.retryCount}/3)`}
            </button>
          )}
          
          <button onClick={() => window.location.reload()}>
            Refresh Page
          </button>
          
          {process.env.NODE_ENV === 'development' && (
            <details>
              <summary>Error Details</summary>
              <pre>{this.state.error.stack}</pre>
            </details>
          )}
        </div>
      );
    }

    return children;
  }
}
```

### Granular Error Boundaries

```jsx
// Feature-specific error boundaries
function FeatureErrorBoundary({ feature, children, fallback }) {
  return (
    <RecoverableErrorBoundary
      fallback={({ error, retry }) => (
        <div className="feature-error">
          <h3>{feature} is temporarily unavailable</h3>
          <p>We're working to fix this issue.</p>
          <button onClick={retry}>Try Again</button>
          {fallback && fallback}
        </div>
      )}
    >
      {children}
    </RecoverableErrorBoundary>
  );
}

// Usage with granular boundaries
function Dashboard() {
  return (
    <div>
      <FeatureErrorBoundary feature="User Profile">
        <UserProfile />
      </FeatureErrorBoundary>
      
      <FeatureErrorBoundary feature="Analytics Dashboard">
        <Analytics />
      </FeatureErrorBoundary>
      
      <FeatureErrorBoundary feature="Recent Activity">
        <RecentActivity />
      </FeatureErrorBoundary>
    </div>
  );
}
```

---

## Async Error Handling

### API Error Handling with Custom Hook

```jsx
// useAsyncError.js - Hook for handling async errors
function useAsyncError() {
  const [_, setError] = useState();
  
  return useCallback((error) => {
    setError(() => {
      throw error;
    });
  }, []);
}

// useApiCall.js - Comprehensive API call hook
function useApiCall(apiFunction, dependencies = []) {
  const [state, setState] = useState({
    data: null,
    loading: false,
    error: null
  });
  
  const throwError = useAsyncError();

  const execute = useCallback(async (...args) => {
    try {
      setState(prev => ({ ...prev, loading: true, error: null }));
      
      const result = await apiFunction(...args);
      
      setState({ data: result, loading: false, error: null });
      return result;
    } catch (error) {
      const errorDetails = {
        message: error.message,
        status: error.status,
        timestamp: Date.now(),
        endpoint: error.config?.url
      };
      
      setState(prev => ({ 
        ...prev, 
        loading: false, 
        error: errorDetails 
      }));
      
      // Throw error to be caught by Error Boundary if needed
      if (error.status >= 500) {
        throwError(error);
      }
      
      return null;
    }
  }, dependencies);

  return { ...state, execute };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error, execute } = useApiCall(api.getUser);

  useEffect(() => {
    execute(userId);
  }, [userId, execute]);

  if (loading) return <ProfileSkeleton />;
  
  if (error) {
    return (
      <div className="error-state">
        <h3>Failed to load user profile</h3>
        <p>{error.message}</p>
        <button onClick={() => execute(userId)}>Retry</button>
      </div>
    );
  }

  return <UserDetails user={user} />;
}
```

### Promise Error Handling

```jsx
// usePromise.js - Hook for promise-based operations
function usePromise(promiseFunction, dependencies = []) {
  const [state, setState] = useState({
    data: null,
    loading: false,
    error: null
  });

  const throwError = useAsyncError();

  useEffect(() => {
    let cancelled = false;

    const executePromise = async () => {
      try {
        setState(prev => ({ ...prev, loading: true, error: null }));
        
        const result = await promiseFunction();
        
        if (!cancelled) {
          setState({ data: result, loading: false, error: null });
        }
      } catch (error) {
        if (!cancelled) {
          setState(prev => ({ 
            ...prev, 
            loading: false, 
            error 
          }));
          
          // Critical errors should bubble up to Error Boundary
          if (error.name === 'ChunkLoadError' || error.status >= 500) {
            throwError(error);
          }
        }
      }
    };

    executePromise();

    return () => {
      cancelled = true;
    };
  }, dependencies);

  return state;
}
```

---

## Form Error Handling

### Comprehensive Form Error Management

```jsx
// useFormErrors.js - Form error handling hook
function useFormErrors() {
  const [errors, setErrors] = useState({});

  const setFieldError = useCallback((field, error) => {
    setErrors(prev => ({ ...prev, [field]: error }));
  }, []);

  const clearFieldError = useCallback((field) => {
    setErrors(prev => {
      const newErrors = { ...prev };
      delete newErrors[field];
      return newErrors;
    });
  }, []);

  const setFormErrors = useCallback((errorObject) => {
    setErrors(errorObject);
  }, []);

  const clearAllErrors = useCallback(() => {
    setErrors({});
  }, []);

  const hasErrors = Object.keys(errors).length > 0;
  const getFieldError = useCallback((field) => errors[field], [errors]);

  return {
    errors,
    hasErrors,
    setFieldError,
    clearFieldError,
    setFormErrors,
    clearAllErrors,
    getFieldError
  };
}

// Form component with comprehensive error handling
function ContactForm({ onSubmit }) {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });
  
  const [submitting, setSubmitting] = useState(false);
  const { errors, setFieldError, clearFieldError, setFormErrors, clearAllErrors, getFieldError } = useFormErrors();

  const validateField = (name, value) => {
    switch (name) {
      case 'name':
        if (!value.trim()) return 'Name is required';
        if (value.length < 2) return 'Name must be at least 2 characters';
        break;
      case 'email':
        if (!value.trim()) return 'Email is required';
        if (!/\S+@\S+\.\S+/.test(value)) return 'Please enter a valid email';
        break;
      case 'message':
        if (!value.trim()) return 'Message is required';
        if (value.length < 10) return 'Message must be at least 10 characters';
        break;
      default:
        return null;
    }
    return null;
  };

  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
    
    // Clear error when user starts typing
    if (getFieldError(name)) {
      clearFieldError(name);
    }
  };

  const handleInputBlur = (e) => {
    const { name, value } = e.target;
    const error = validateField(name, value);
    
    if (error) {
      setFieldError(name, error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    clearAllErrors();

    // Validate all fields
    const validationErrors = {};
    Object.keys(formData).forEach(field => {
      const error = validateField(field, formData[field]);
      if (error) {
        validationErrors[field] = error;
      }
    });

    if (Object.keys(validationErrors).length > 0) {
      setFormErrors(validationErrors);
      return;
    }

    try {
      setSubmitting(true);
      await onSubmit(formData);
      
      // Reset form on success
      setFormData({ name: '', email: '', message: '' });
    } catch (error) {
      // Handle different types of errors
      if (error.status === 422 && error.data?.errors) {
        // Validation errors from server
        setFormErrors(error.data.errors);
      } else if (error.status === 400) {
        setFieldError('general', 'Please check your input and try again');
      } else {
        // Let Error Boundary handle server errors
        throw error;
      }
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} noValidate>
      {getFieldError('general') && (
        <div className="error-alert">
          {getFieldError('general')}
        </div>
      )}
      
      <div className="form-group">
        <label htmlFor="name">Name</label>
        <input
          id="name"
          name="name"
          type="text"
          value={formData.name}
          onChange={handleInputChange}
          onBlur={handleInputBlur}
          className={getFieldError('name') ? 'error' : ''}
        />
        {getFieldError('name') && (
          <span className="error-message">{getFieldError('name')}</span>
        )}
      </div>

      <div className="form-group">
        <label htmlFor="email">Email</label>
        <input
          id="email"
          name="email"
          type="email"
          value={formData.email}
          onChange={handleInputChange}
          onBlur={handleInputBlur}
          className={getFieldError('email') ? 'error' : ''}
        />
        {getFieldError('email') && (
          <span className="error-message">{getFieldError('email')}</span>
        )}
      </div>

      <div className="form-group">
        <label htmlFor="message">Message</label>
        <textarea
          id="message"
          name="message"
          value={formData.message}
          onChange={handleInputChange}
          onBlur={handleInputBlur}
          className={getFieldError('message') ? 'error' : ''}
        />
        {getFieldError('message') && (
          <span className="error-message">{getFieldError('message')}</span>
        )}
      </div>

      <button type="submit" disabled={submitting}>
        {submitting ? 'Sending...' : 'Send Message'}
      </button>
    </form>
  );
}
```

---

## Global Error Handling

### Global Error Context

```jsx
// ErrorContext.js
const ErrorContext = createContext();

export function ErrorProvider({ children }) {
  const [errors, setErrors] = useState([]);

  const addError = useCallback((error) => {
    const errorWithId = {
      id: Date.now(),
      message: error.message || 'An unexpected error occurred',
      type: error.type || 'error',
      timestamp: new Date(),
      ...error
    };
    
    setErrors(prev => [errorWithId, ...prev]);
    
    // Auto-remove after 5 seconds
    setTimeout(() => {
      removeError(errorWithId.id);
    }, 5000);
  }, []);

  const removeError = useCallback((id) => {
    setErrors(prev => prev.filter(error => error.id !== id));
  }, []);

  const clearAllErrors = useCallback(() => {
    setErrors([]);
  }, []);

  return (
    <ErrorContext.Provider value={{ 
      errors, 
      addError, 
      removeError, 
      clearAllErrors 
    }}>
      {children}
      <ErrorToastContainer errors={errors} onRemove={removeError} />
    </ErrorContext.Provider>
  );
}

export const useError = () => {
  const context = useContext(ErrorContext);
  if (!context) {
    throw new Error('useError must be used within ErrorProvider');
  }
  return context;
};

// ErrorToastContainer.jsx
function ErrorToastContainer({ errors, onRemove }) {
  return (
    <div className="error-toast-container">
      {errors.map(error => (
        <div key={error.id} className={`error-toast ${error.type}`}>
          <div className="error-content">
            <strong>{error.title || 'Error'}</strong>
            <p>{error.message}</p>
          </div>
          <button onClick={() => onRemove(error.id)}>×</button>
        </div>
      ))}
    </div>
  );
}
```

### Window Error Handling

```jsx
// GlobalErrorHandler.js
function GlobalErrorHandler() {
  const { addError } = useError();

  useEffect(() => {
    // Handle unhandled JavaScript errors
    const handleError = (event) => {
      addError({
        type: 'javascript',
        title: 'JavaScript Error',
        message: event.message,
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno,
        stack: event.error?.stack
      });
    };

    // Handle unhandled promise rejections
    const handleUnhandledRejection = (event) => {
      addError({
        type: 'promise',
        title: 'Unhandled Promise Rejection',
        message: event.reason?.message || 'Promise was rejected',
        stack: event.reason?.stack
      });
    };

    window.addEventListener('error', handleError);
    window.addEventListener('unhandledrejection', handleUnhandledRejection);

    return () => {
      window.removeEventListener('error', handleError);
      window.removeEventListener('unhandledrejection', handleUnhandledRejection);
    };
  }, [addError]);

  return null;
}

// App.js
function App() {
  return (
    <ErrorProvider>
      <GlobalErrorHandler />
      <ErrorBoundary>
        <Router>
          <Routes>
            <Route path="/" element={<Home />} />
            <Route path="/profile" element={<Profile />} />
          </Routes>
        </Router>
      </ErrorBoundary>
    </ErrorProvider>
  );
}
```

---

## Error Monitoring and Reporting

### Error Reporting Service

```jsx
// errorReporting.js
class ErrorReportingService {
  constructor() {
    this.isProduction = process.env.NODE_ENV === 'production';
    this.apiEndpoint = process.env.REACT_APP_ERROR_ENDPOINT;
  }

  reportError(error, context = {}) {
    const errorReport = {
      message: error.message,
      stack: error.stack,
      timestamp: new Date().toISOString(),
      url: window.location.href,
      userAgent: navigator.userAgent,
      userId: this.getUserId(),
      sessionId: this.getSessionId(),
      ...context
    };

    if (this.isProduction) {
      this.sendToService(errorReport);
    } else {
      console.group('🚨 Error Report');
      console.error('Error:', error);
      console.log('Context:', context);
      console.log('Full Report:', errorReport);
      console.groupEnd();
    }
  }

  sendToService(errorReport) {
    fetch(this.apiEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(errorReport)
    }).catch(err => {
      console.error('Failed to send error report:', err);
    });
  }

  getUserId() {
    // Get from auth context or localStorage
    return localStorage.getItem('userId') || 'anonymous';
  }

  getSessionId() {
    // Generate or retrieve session ID
    let sessionId = sessionStorage.getItem('sessionId');
    if (!sessionId) {
      sessionId = Date.now().toString();
      sessionStorage.setItem('sessionId', sessionId);
    }
    return sessionId;
  }
}

export const errorReporting = new ErrorReportingService();
```

---

## Testing Error Handling

### Testing Error Boundaries

```jsx
// ErrorBoundary.test.jsx
import { render, screen } from '@testing-library/react';
import ErrorBoundary from './ErrorBoundary';

// Component that throws an error
function ThrowError({ shouldThrow }) {
  if (shouldThrow) {
    throw new Error('Test error');
  }
  return <div>No error</div>;
}

describe('ErrorBoundary', () => {
  // Suppress console.error for tests
  const originalError = console.error;
  beforeAll(() => {
    console.error = jest.fn();
  });

  afterAll(() => {
    console.error = originalError;
  });

  test('renders children when no error', () => {
    render(
      <ErrorBoundary>
        <ThrowError shouldThrow={false} />
      </ErrorBoundary>
    );
    
    expect(screen.getByText('No error')).toBeInTheDocument();
  });

  test('renders error UI when child throws', () => {
    render(
      <ErrorBoundary>
        <ThrowError shouldThrow={true} />
      </ErrorBoundary>
    );
    
    expect(screen.getByText('Something went wrong')).toBeInTheDocument();
  });

  test('calls onError prop when error occurs', () => {
    const onError = jest.fn();
    
    render(
      <ErrorBoundary onError={onError}>
        <ThrowError shouldThrow={true} />
      </ErrorBoundary>
    );
    
    expect(onError).toHaveBeenCalledWith(
      expect.objectContaining({
        error: expect.any(String),
        componentStack: expect.any(String)
      })
    );
  });
});
```

---

## Best practices

* Use Error Boundaries at strategic points in your component tree
* Implement granular error boundaries for better user experience
* Log errors with sufficient context for debugging
* Provide meaningful fallback UIs
* Handle different types of errors appropriately
* Use proper error monitoring in production
* Test error scenarios thoroughly
* Implement retry mechanisms for recoverable errors
* Validate data at boundaries (API responses, user input)
* Use TypeScript to catch errors at compile time

---

## Interview-ready summary

React error handling uses Error Boundaries to catch component errors, custom hooks for async errors, and global error contexts for application-wide error management. Key patterns include granular boundaries for features, comprehensive form validation, promise rejection handling, and error reporting services. Error Boundaries only catch errors in component trees, not async operations or event handlers. Modern approaches combine Error Boundaries with custom hooks like `useAsyncError` to handle all error types. Always provide fallback UIs, implement retry mechanisms, and use proper error monitoring in production.
