# Conditional rendering & Error Boundaries

This file covers techniques for conditionally rendering components and implementing error boundaries to handle component failures gracefully.
Essential for building robust user interfaces with proper error handling.

---

## Conditional Rendering Patterns

### Basic conditional rendering

```jsx
// 1. Logical AND operator - most common
function WelcomeMessage({ user }) {
  return (
    <div>
      {user && <h1>Welcome, {user.name}!</h1>}
      {user?.isAdmin && <AdminPanel />}
    </div>
  );
}

// 2. Ternary operator - for either/or scenarios
function LoginStatus({ isLoggedIn, user }) {
  return (
    <div>
      {isLoggedIn ? (
        <UserDashboard user={user} />
      ) : (
        <LoginForm />
      )}
    </div>
  );
}

// 3. Early return - for cleaner component structure
function ProfilePage({ user, loading, error }) {
  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!user) return <LoginRequired />;
  
  return <UserProfile user={user} />;
}

// 4. Switch statements for multiple conditions
function StatusIndicator({ status }) {
  const renderStatusContent = () => {
    switch (status) {
      case 'loading':
        return <Spinner />;
      case 'success':
        return <CheckIcon className="text-green-500" />;
      case 'error':
        return <ErrorIcon className="text-red-500" />;
      case 'warning':
        return <WarningIcon className="text-yellow-500" />;
      default:
        return <InfoIcon className="text-blue-500" />;
    }
  };

  return (
    <div className="status-indicator">
      {renderStatusContent()}
    </div>
  );
}
```

### Advanced conditional patterns

```jsx
// Conditional rendering with multiple checks
function FeatureComponent({ user, feature, permissions }) {
  // Multiple conditions using helper functions
  const canViewFeature = () => {
    return user?.isActive && 
           feature?.isEnabled && 
           permissions?.includes('view_feature');
  };

  const hasProAccess = () => {
    return user?.subscription === 'pro' || user?.isAdmin;
  };

  if (!canViewFeature()) {
    return <FeatureUnavailable reason="access_denied" />;
  }

  return (
    <div className="feature">
      <FeatureHeader title={feature.name} />
      
      {hasProAccess() ? (
        <ProFeatureContent feature={feature} />
      ) : (
        <div>
          <BasicFeatureContent feature={feature} />
          <UpgradePrompt />
        </div>
      )}
      
      {user?.isAdmin && <AdminControls feature={feature} />}
    </div>
  );
}

// Conditional rendering with custom hooks
function usePermissions(user) {
  return useMemo(() => ({
    canEdit: user?.role === 'admin' || user?.role === 'editor',
    canDelete: user?.role === 'admin',
    canView: user?.isActive,
    canCreate: user?.isActive && (user?.role === 'admin' || user?.role === 'editor')
  }), [user]);
}

function DocumentList({ documents, user }) {
  const permissions = usePermissions(user);

  return (
    <div className="document-list">
      {permissions.canCreate && (
        <CreateDocumentButton />
      )}
      
      {documents.map(doc => (
        <div key={doc.id} className="document-item">
          <DocumentTitle title={doc.title} />
          
          <div className="document-actions">
            {permissions.canView && (
              <ViewButton documentId={doc.id} />
            )}
            {permissions.canEdit && (
              <EditButton documentId={doc.id} />
            )}
            {permissions.canDelete && (
              <DeleteButton documentId={doc.id} />
            )}
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Conditional styling and classes

```jsx
// Conditional CSS classes
function Button({ variant, size, disabled, loading, children, ...props }) {
  const buttonClasses = [
    'btn',
    `btn--${variant}`,
    `btn--${size}`,
    disabled && 'btn--disabled',
    loading && 'btn--loading'
  ].filter(Boolean).join(' ');

  return (
    <button 
      className={buttonClasses}
      disabled={disabled || loading}
      {...props}
    >
      {loading && <Spinner size="small" />}
      {children}
    </button>
  );
}

// Using classnames library for complex conditions
import classNames from 'classnames';

function Alert({ type, dismissible, children, onDismiss }) {
  const alertClasses = classNames('alert', {
    'alert--success': type === 'success',
    'alert--error': type === 'error',
    'alert--warning': type === 'warning',
    'alert--dismissible': dismissible
  });

  return (
    <div className={alertClasses}>
      <div className="alert-content">{children}</div>
      {dismissible && (
        <button className="alert-dismiss" onClick={onDismiss}>
          ×
        </button>
      )}
    </div>
  );
}

// Conditional inline styles
function ProgressBar({ progress, color, showPercentage = true }) {
  const progressStyle = {
    width: `${Math.min(100, Math.max(0, progress))}%`,
    backgroundColor: color || '#007bff',
    transition: 'width 0.3s ease'
  };

  return (
    <div className="progress-bar">
      <div className="progress-fill" style={progressStyle} />
      {showPercentage && (
        <span className="progress-text">
          {Math.round(progress)}%
        </span>
      )}
    </div>
  );
}
```

### Complex conditional rendering scenarios

```jsx
// Multi-step form with conditional steps
function MultiStepForm({ onSubmit }) {
  const [currentStep, setCurrentStep] = useState(1);
  const [formData, setFormData] = useState({});
  const [completedSteps, setCompletedSteps] = useState(new Set());

  const steps = [
    { id: 1, title: 'Personal Info', component: PersonalInfoStep },
    { id: 2, title: 'Address', component: AddressStep, 
      condition: () => formData.needsShipping },
    { id: 3, title: 'Payment', component: PaymentStep },
    { id: 4, title: 'Review', component: ReviewStep }
  ];

  const visibleSteps = steps.filter(step => 
    !step.condition || step.condition()
  );

  const currentStepData = visibleSteps.find(step => step.id === currentStep);
  const CurrentStepComponent = currentStepData?.component;

  const canProceed = () => {
    switch (currentStep) {
      case 1:
        return formData.firstName && formData.lastName && formData.email;
      case 2:
        return formData.address && formData.city && formData.zipCode;
      case 3:
        return formData.paymentMethod && formData.cardNumber;
      default:
        return true;
    }
  };

  return (
    <div className="multi-step-form">
      <FormProgress 
        steps={visibleSteps}
        currentStep={currentStep}
        completedSteps={completedSteps}
      />

      {CurrentStepComponent && (
        <CurrentStepComponent
          data={formData}
          onChange={setFormData}
          onNext={() => {
            setCompletedSteps(prev => new Set([...prev, currentStep]));
            setCurrentStep(currentStep + 1);
          }}
          onPrevious={() => setCurrentStep(currentStep - 1)}
          canProceed={canProceed()}
        />
      )}
    </div>
  );
}

// Dynamic component rendering based on data
function DynamicContent({ contentBlocks }) {
  const renderBlock = (block) => {
    switch (block.type) {
      case 'text':
        return <TextBlock key={block.id} content={block.content} />;
      case 'image':
        return <ImageBlock key={block.id} src={block.src} alt={block.alt} />;
      case 'video':
        return <VideoBlock key={block.id} url={block.url} />;
      case 'form':
        return <FormBlock key={block.id} fields={block.fields} />;
      case 'chart':
        return block.chartType === 'line' ? 
          <LineChart key={block.id} data={block.data} /> :
          <BarChart key={block.id} data={block.data} />;
      default:
        return <UnknownBlock key={block.id} type={block.type} />;
    }
  };

  return (
    <div className="dynamic-content">
      {contentBlocks.map(renderBlock)}
    </div>
  );
}
```

---

## Error Boundaries Implementation

### Basic Error Boundary

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { 
      hasError: false, 
      error: null, 
      errorInfo: null 
    };
  }

  static getDerivedStateFromError(error) {
    // Update state to render fallback UI
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // Log error details
    console.error('Error caught by boundary:', error, errorInfo);
    
    this.setState({
      error,
      errorInfo
    });

    // Report to error tracking service
    this.reportError(error, errorInfo);
  }

  reportError = (error, errorInfo) => {
    // Send error to monitoring service
    if (typeof window.gtag === 'function') {
      window.gtag('event', 'exception', {
        description: error.toString(),
        fatal: false
      });
    }

    // Or use error tracking service like Sentry
    // Sentry.captureException(error, { extra: errorInfo });
  };

  render() {
    if (this.state.hasError) {
      // Fallback UI
      return (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <p>We're sorry for the inconvenience. Please try refreshing the page.</p>
          
          <button onClick={() => window.location.reload()}>
            Refresh Page
          </button>
          
          {process.env.NODE_ENV === 'development' && (
            <details className="error-details">
              <summary>Error Details (Development Only)</summary>
              <pre>{this.state.error && this.state.error.toString()}</pre>
              <pre>{this.state.errorInfo.componentStack}</pre>
            </details>
          )}
        </div>
      );
    }

    return this.props.children;
  }
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
      retryCount: 0,
      lastErrorTime: null
    };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    const now = Date.now();
    
    this.setState(prevState => ({
      retryCount: prevState.retryCount + 1,
      lastErrorTime: now
    }));

    // Enhanced error reporting
    const errorReport = {
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      retryCount: this.state.retryCount,
      userAgent: navigator.userAgent,
      timestamp: new Date().toISOString(),
      url: window.location.href,
      userId: this.props.userId || 'anonymous'
    };

    this.props.onError?.(errorReport);

    // Auto-recovery for certain error types
    if (this.shouldAutoRecover(error) && this.state.retryCount < 3) {
      setTimeout(() => {
        this.handleRetry();
      }, Math.min(1000 * Math.pow(2, this.state.retryCount), 5000));
    }
  }

  shouldAutoRecover = (error) => {
    // Auto-recover from network-related errors
    const recoverableErrors = [
      'ChunkLoadError',
      'Loading chunk',
      'Network Error',
      'fetch'
    ];
    
    return recoverableErrors.some(errorType => 
      error.message.includes(errorType) || error.name.includes(errorType)
    );
  };

  handleRetry = () => {
    this.setState({ 
      hasError: false, 
      error: null 
    });
  };

  render() {
    if (this.state.hasError) {
      // Custom fallback component
      if (this.props.fallback) {
        return this.props.fallback({
          error: this.state.error,
          retry: this.handleRetry,
          retryCount: this.state.retryCount
        });
      }

      // Default fallback UI
      return (
        <ErrorFallback
          error={this.state.error}
          onRetry={this.handleRetry}
          retryCount={this.state.retryCount}
          maxRetries={3}
        />
      );
    }

    return this.props.children;
  }
}

// Reusable error fallback component
function ErrorFallback({ error, onRetry, retryCount, maxRetries = 3 }) {
  const canRetry = retryCount < maxRetries;

  return (
    <div className="error-fallback">
      <div className="error-icon">⚠️</div>
      <h3>Oops! Something went wrong</h3>
      <p className="error-message">
        {error?.message || 'An unexpected error occurred'}
      </p>
      
      <div className="error-actions">
        {canRetry ? (
          <button 
            className="retry-button"
            onClick={onRetry}
          >
            Try Again {retryCount > 0 && `(${retryCount}/${maxRetries})`}
          </button>
        ) : (
          <button 
            className="refresh-button"
            onClick={() => window.location.reload()}
          >
            Refresh Page
          </button>
        )}
        
        <button 
          className="home-button"
          onClick={() => window.location.href = '/'}
        >
          Go Home
        </button>
      </div>
    </div>
  );
}
```

### Feature-specific Error Boundaries

```jsx
// Error boundary for specific features
function FeatureErrorBoundary({ 
  feature, 
  fallback, 
  onError, 
  children 
}) {
  return (
    <RecoverableErrorBoundary
      onError={(error) => {
        onError?.({
          ...error,
          feature,
          context: 'feature_boundary'
        });
      }}
      fallback={({ error, retry, retryCount }) => (
        fallback ? fallback({ error, retry, retryCount }) : (
          <div className="feature-error">
            <h4>{feature} is temporarily unavailable</h4>
            <p>We're working to fix this issue.</p>
            <button onClick={retry}>Try Again</button>
          </div>
        )
      )}
    >
      {children}
    </RecoverableErrorBoundary>
  );
}

// Usage in application
function Dashboard() {
  const handleFeatureError = (errorReport) => {
    console.error(`Feature error in ${errorReport.feature}:`, errorReport);
    // Send to analytics or error tracking
  };

  return (
    <div className="dashboard">
      <FeatureErrorBoundary 
        feature="Analytics" 
        onError={handleFeatureError}
      >
        <AnalyticsWidget />
      </FeatureErrorBoundary>

      <FeatureErrorBoundary 
        feature="User Profile" 
        onError={handleFeatureError}
        fallback={({ retry }) => (
          <div className="profile-error">
            <p>Profile unavailable</p>
            <button onClick={retry}>Reload Profile</button>
          </div>
        )}
      >
        <UserProfile />
      </FeatureErrorBoundary>

      <FeatureErrorBoundary 
        feature="Recent Activity" 
        onError={handleFeatureError}
      >
        <RecentActivity />
      </FeatureErrorBoundary>
    </div>
  );
}
```

### Error Boundary with Async Error Handling

```jsx
// Hook to bubble async errors to Error Boundary
function useAsyncError() {
  const [_, setError] = useState();
  
  return useCallback((error) => {
    setError(() => {
      throw error;
    });
  }, []);
}

// Component that handles async operations
function AsyncDataComponent({ dataId }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const throwAsyncError = useAsyncError();

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        const result = await api.getData(dataId);
        setData(result);
      } catch (error) {
        // Determine if error should bubble to Error Boundary
        if (error.status >= 500 || error.name === 'ChunkLoadError') {
          throwAsyncError(error);
        } else {
          // Handle client errors locally
          setData(null);
        }
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [dataId, throwAsyncError]);

  if (loading) return <LoadingSpinner />;
  if (!data) return <DataNotFound />;

  return <DataDisplay data={data} />;
}

// Usage with Error Boundary
function App() {
  return (
    <RecoverableErrorBoundary>
      <AsyncDataComponent dataId="123" />
    </RecoverableErrorBoundary>
  );
}
```

---

## Combining Conditional Rendering with Error Boundaries

### Progressive Enhancement Pattern

```jsx
function ProgressiveFeature({ 
  user, 
  feature, 
  fallbackComponent: Fallback 
}) {
  // Multiple layers of conditional rendering
  const renderFeature = () => {
    // Check user permissions
    if (!user?.hasAccess(feature.id)) {
      return <AccessDenied feature={feature.name} />;
    }

    // Check feature flags
    if (!feature.isEnabled) {
      return <FeatureDisabled feature={feature.name} />;
    }

    // Check browser capabilities
    if (!feature.browserSupport.every(checkBrowserSupport)) {
      return <BrowserNotSupported requirements={feature.browserSupport} />;
    }

    // Render main feature
    return <MainFeature feature={feature} user={user} />;
  };

  return (
    <FeatureErrorBoundary 
      feature={feature.name}
      fallback={({ error, retry }) => (
        Fallback ? (
          <Fallback error={error} onRetry={retry} />
        ) : (
          <DefaultFeatureFallback 
            featureName={feature.name}
            error={error}
            onRetry={retry}
          />
        )
      )}
    >
      {renderFeature()}
    </FeatureErrorBoundary>
  );
}

// Smart loading states with error handling
function SmartLoader({ 
  loading, 
  error, 
  data, 
  children, 
  loadingComponent: Loading = DefaultLoading,
  errorComponent: Error = DefaultError,
  emptyComponent: Empty = DefaultEmpty 
}) {
  return (
    <RecoverableErrorBoundary>
      {loading && <Loading />}
      {error && <Error error={error} />}
      {!loading && !error && !data && <Empty />}
      {!loading && !error && data && children}
    </RecoverableErrorBoundary>
  );
}
```

---

## Best Practices

### Conditional Rendering
* Use early returns for cleaner code structure
* Prefer logical AND (&&) for simple show/hide scenarios
* Use ternary operators for either/or cases
* Extract complex conditions into helper functions or custom hooks
* Be careful with falsy values (0, "", null, undefined)

### Error Boundaries
* Place Error Boundaries at strategic points in component tree
* Implement granular boundaries for better user experience
* Provide meaningful fallback UIs
* Include retry mechanisms where appropriate
* Log errors with sufficient context for debugging
* Don't catch errors in event handlers (use try/catch instead)

### Testing
```jsx
// Testing conditional rendering
describe('ConditionalComponent', () => {
  test('shows loading state', () => {
    render(<ConditionalComponent loading={true} />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  test('shows error state', () => {
    const error = new Error('Test error');
    render(<ConditionalComponent error={error} />);
    expect(screen.getByText('Test error')).toBeInTheDocument();
  });

  test('shows content when loaded', () => {
    const data = { name: 'Test' };
    render(<ConditionalComponent data={data} />);
    expect(screen.getByText('Test')).toBeInTheDocument();
  });
});

// Testing Error Boundaries
describe('ErrorBoundary', () => {
  test('catches errors and shows fallback', () => {
    const ThrowError = () => {
      throw new Error('Test error');
    };

    render(
      <ErrorBoundary>
        <ThrowError />
      </ErrorBoundary>
    );

    expect(screen.getByText('Something went wrong')).toBeInTheDocument();
  });
});
```

---

## Interview-ready summary

Conditional rendering uses logical AND (&&), ternary operators, early returns, and switch statements to show/hide components based on state, props, or conditions. Common patterns include loading states, user permissions, feature flags, and progressive enhancement. Error Boundaries catch JavaScript errors in component trees using `componentDidCatch` and `getDerivedStateFromError`, providing fallback UIs and error recovery. They don't catch errors in event handlers, async code, or during SSR. Best practices include granular boundaries, meaningful fallbacks, retry mechanisms, and proper error logging. Combine both techniques for robust UIs that handle various states and failures gracefully.
