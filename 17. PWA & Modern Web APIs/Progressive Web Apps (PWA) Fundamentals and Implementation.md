# Progressive Web Apps (PWA) Fundamentals and Implementation

Progressive Web Apps (PWAs) combine the best of web and mobile apps, providing native-like experiences through modern web technologies. This guide covers PWA fundamentals, implementation strategies, and advanced features for creating app-like web experiences.

## PWA Core Concepts and Architecture

### 1. PWA Fundamentals and Requirements
```typescript
// PWA Manifest Configuration
interface WebAppManifest {
  name: string;
  short_name: string;
  description: string;
  start_url: string;
  display: 'fullscreen' | 'standalone' | 'minimal-ui' | 'browser';
  orientation: 'portrait' | 'landscape' | 'any';
  theme_color: string;
  background_color: string;
  icons: ManifestIcon[];
  categories?: string[];
  screenshots?: ManifestScreenshot[];
  shortcuts?: ManifestShortcut[];
  related_applications?: ManifestRelatedApplication[];
  prefer_related_applications?: boolean;
  scope: string;
  id: string;
  dir: 'ltr' | 'rtl' | 'auto';
  lang: string;
}

interface ManifestIcon {
  src: string;
  sizes: string;
  type: string;
  purpose?: 'maskable' | 'any' | 'monochrome';
}

interface ManifestScreenshot {
  src: string;
  sizes: string;
  type: string;
  label?: string;
}

interface ManifestShortcut {
  name: string;
  short_name?: string;
  description?: string;
  url: string;
  icons?: ManifestIcon[];
}

interface ManifestRelatedApplication {
  platform: string;
  url?: string;
  id?: string;
}

// Generate optimized PWA manifest
export function generatePWAManifest(config: {
  appName: string;
  appDescription: string;
  themeColor: string;
  backgroundColor: string;
  startUrl?: string;
  scope?: string;
}): WebAppManifest {
  const {
    appName,
    appDescription,
    themeColor,
    backgroundColor,
    startUrl = '/',
    scope = '/',
  } = config;

  return {
    name: appName,
    short_name: appName.length > 12 ? appName.substring(0, 12) : appName,
    description: appDescription,
    start_url: startUrl,
    scope,
    display: 'standalone',
    orientation: 'portrait',
    theme_color: themeColor,
    background_color: backgroundColor,
    id: scope,
    dir: 'ltr',
    lang: 'en',
    
    // Comprehensive icon set for all platforms
    icons: [
      // Android Chrome
      {
        src: '/icons/icon-192x192.png',
        sizes: '192x192',
        type: 'image/png',
        purpose: 'maskable any',
      },
      {
        src: '/icons/icon-512x512.png',
        sizes: '512x512',
        type: 'image/png',
        purpose: 'maskable any',
      },
      // iOS Safari
      {
        src: '/icons/icon-180x180.png',
        sizes: '180x180',
        type: 'image/png',
        purpose: 'any',
      },
      // Windows
      {
        src: '/icons/icon-144x144.png',
        sizes: '144x144',
        type: 'image/png',
        purpose: 'any',
      },
      // Favicon
      {
        src: '/icons/icon-48x48.png',
        sizes: '48x48',
        type: 'image/png',
        purpose: 'any',
      },
    ],

    // App screenshots for app stores
    screenshots: [
      {
        src: '/screenshots/mobile-screenshot-1.png',
        sizes: '750x1334',
        type: 'image/png',
        label: 'Mobile app view',
      },
      {
        src: '/screenshots/desktop-screenshot-1.png',
        sizes: '1920x1080',
        type: 'image/png',
        label: 'Desktop app view',
      },
    ],

    // App shortcuts for quick actions
    shortcuts: [
      {
        name: 'New Post',
        short_name: 'New',
        description: 'Create a new post',
        url: '/create',
        icons: [
          {
            src: '/icons/shortcut-new.png',
            sizes: '96x96',
            type: 'image/png',
          },
        ],
      },
      {
        name: 'Profile',
        short_name: 'Profile',
        description: 'View your profile',
        url: '/profile',
        icons: [
          {
            src: '/icons/shortcut-profile.png',
            sizes: '96x96',
            type: 'image/png',
          },
        ],
      },
    ],

    // App categories for discoverability
    categories: ['productivity', 'social', 'utilities'],
  };
}

// PWA Installation Detection and Management
class PWAInstallManager {
  private deferredPrompt: any = null;
  private isInstalled = false;
  private installListeners = new Set<Function>();

  constructor() {
    this.initializeInstallation();
  }

  private initializeInstallation(): void {
    // Listen for the beforeinstallprompt event
    window.addEventListener('beforeinstallprompt', (e) => {
      // Prevent Chrome 67 and earlier from automatically showing the prompt
      e.preventDefault();
      
      // Store the event for later use
      this.deferredPrompt = e;
      
      // Notify listeners that app is installable
      this.notifyInstallListeners('installable');
    });

    // Listen for the appinstalled event
    window.addEventListener('appinstalled', () => {
      this.isInstalled = true;
      this.deferredPrompt = null;
      this.notifyInstallListeners('installed');
    });

    // Check if app is already installed
    this.checkInstallationStatus();
  }

  private async checkInstallationStatus(): Promise<void> {
    // Check if running in standalone mode (installed PWA)
    if (window.matchMedia('(display-mode: standalone)').matches) {
      this.isInstalled = true;
    }

    // Check if running in iOS Safari standalone mode
    if ((window.navigator as any).standalone === true) {
      this.isInstalled = true;
    }

    // Use getInstalledRelatedApps if available
    if ('getInstalledRelatedApps' in navigator) {
      try {
        const relatedApps = await (navigator as any).getInstalledRelatedApps();
        this.isInstalled = relatedApps.length > 0;
      } catch (error) {
        console.warn('getInstalledRelatedApps not available:', error);
      }
    }
  }

  // Show install prompt
  async showInstallPrompt(): Promise<{ outcome: 'accepted' | 'dismissed' }> {
    if (!this.deferredPrompt) {
      throw new Error('App install prompt not available');
    }

    // Show the install prompt
    this.deferredPrompt.prompt();

    // Wait for the user to respond to the prompt
    const { outcome } = await this.deferredPrompt.userChoice;

    // Clear the deferredPrompt for next time
    this.deferredPrompt = null;

    return { outcome };
  }

  // Check if app can be installed
  canInstall(): boolean {
    return !!this.deferredPrompt && !this.isInstalled;
  }

  // Check if app is installed
  getInstallationStatus(): boolean {
    return this.isInstalled;
  }

  // Add installation event listener
  onInstallEvent(callback: (event: 'installable' | 'installed') => void): () => void {
    this.installListeners.add(callback);
    
    // Return unsubscribe function
    return () => {
      this.installListeners.delete(callback);
    };
  }

  private notifyInstallListeners(event: 'installable' | 'installed'): void {
    this.installListeners.forEach(callback => {
      try {
        callback(event);
      } catch (error) {
        console.error('Error in install event listener:', error);
      }
    });
  }

  // Generate iOS-specific meta tags
  generateIOSMetaTags(): string {
    return `
      <!-- iOS Safari PWA support -->
      <meta name="apple-mobile-web-app-capable" content="yes">
      <meta name="apple-mobile-web-app-status-bar-style" content="default">
      <meta name="apple-mobile-web-app-title" content="App Name">
      
      <!-- iOS Icons -->
      <link rel="apple-touch-icon" href="/icons/icon-180x180.png">
      <link rel="apple-touch-icon" sizes="152x152" href="/icons/icon-152x152.png">
      <link rel="apple-touch-icon" sizes="144x144" href="/icons/icon-144x144.png">
      <link rel="apple-touch-icon" sizes="120x120" href="/icons/icon-120x120.png">
      <link rel="apple-touch-icon" sizes="114x114" href="/icons/icon-114x114.png">
      <link rel="apple-touch-icon" sizes="76x76" href="/icons/icon-76x76.png">
      <link rel="apple-touch-icon" sizes="72x72" href="/icons/icon-72x72.png">
      <link rel="apple-touch-icon" sizes="60x60" href="/icons/icon-60x60.png">
      <link rel="apple-touch-icon" sizes="57x57" href="/icons/icon-57x57.png">
      
      <!-- iOS Splash Screens -->
      <link rel="apple-touch-startup-image" href="/splash/splash-2048x2732.png" media="(device-width: 1024px) and (device-height: 1366px) and (-webkit-device-pixel-ratio: 2) and (orientation: portrait)">
      <link rel="apple-touch-startup-image" href="/splash/splash-1668x2224.png" media="(device-width: 834px) and (device-height: 1112px) and (-webkit-device-pixel-ratio: 2) and (orientation: portrait)">
      <link rel="apple-touch-startup-image" href="/splash/splash-1536x2048.png" media="(device-width: 768px) and (device-height: 1024px) and (-webkit-device-pixel-ratio: 2) and (orientation: portrait)">
      <link rel="apple-touch-startup-image" href="/splash/splash-1125x2436.png" media="(device-width: 375px) and (device-height: 812px) and (-webkit-device-pixel-ratio: 3) and (orientation: portrait)">
      <link rel="apple-touch-startup-image" href="/splash/splash-1242x2208.png" media="(device-width: 414px) and (device-height: 736px) and (-webkit-device-pixel-ratio: 3) and (orientation: portrait)">
      <link rel="apple-touch-startup-image" href="/splash/splash-750x1334.png" media="(device-width: 375px) and (device-height: 667px) and (-webkit-device-pixel-ratio: 2) and (orientation: portrait)">
      <link rel="apple-touch-startup-image" href="/splash/splash-640x1136.png" media="(device-width: 320px) and (device-height: 568px) and (-webkit-device-pixel-ratio: 2) and (orientation: portrait)">
    `;
  }
}

// React hook for PWA installation
export function usePWAInstall() {
  const [canInstall, setCanInstall] = useState(false);
  const [isInstalled, setIsInstalled] = useState(false);
  const [installManager] = useState(() => new PWAInstallManager());

  useEffect(() => {
    const unsubscribe = installManager.onInstallEvent((event) => {
      if (event === 'installable') {
        setCanInstall(true);
      } else if (event === 'installed') {
        setIsInstalled(true);
        setCanInstall(false);
      }
    });

    // Initial status check
    setCanInstall(installManager.canInstall());
    setIsInstalled(installManager.getInstallationStatus());

    return unsubscribe;
  }, [installManager]);

  const install = useCallback(async () => {
    try {
      const result = await installManager.showInstallPrompt();
      return result.outcome === 'accepted';
    } catch (error) {
      console.error('Failed to install PWA:', error);
      return false;
    }
  }, [installManager]);

  return {
    canInstall,
    isInstalled,
    install,
  };
}

// PWA Install Prompt Component
interface PWAInstallPromptProps {
  appName: string;
  appDescription?: string;
  className?: string;
  onInstallSuccess?: () => void;
  onInstallCancel?: () => void;
}

export const PWAInstallPrompt: React.FC<PWAInstallPromptProps> = ({
  appName,
  appDescription = 'Install this app for a better experience',
  className = '',
  onInstallSuccess,
  onInstallCancel,
}) => {
  const { canInstall, isInstalled, install } = usePWAInstall();
  const [isInstalling, setIsInstalling] = useState(false);

  const handleInstall = async () => {
    setIsInstalling(true);
    try {
      const success = await install();
      if (success) {
        onInstallSuccess?.();
      } else {
        onInstallCancel?.();
      }
    } catch (error) {
      console.error('Installation failed:', error);
      onInstallCancel?.();
    } finally {
      setIsInstalling(false);
    }
  };

  if (!canInstall || isInstalled) {
    return null;
  }

  return (
    <div className={`pwa-install-prompt ${className}`}>
      <div className="flex items-center justify-between p-4 bg-blue-50 border border-blue-200 rounded-lg">
        <div className="flex items-center space-x-3">
          <div className="flex-shrink-0">
            <svg className="w-8 h-8 text-blue-600" fill="currentColor" viewBox="0 0 20 20">
              <path fillRule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm3.293-7.707a1 1 0 011.414 0L9 10.586V3a1 1 0 112 0v7.586l1.293-1.293a1 1 0 111.414 1.414l-3 3a1 1 0 01-1.414 0l-3-3a1 1 0 010-1.414z" clipRule="evenodd" />
            </svg>
          </div>
          <div>
            <h3 className="text-sm font-medium text-blue-800">
              Install {appName}
            </h3>
            <p className="text-sm text-blue-600">{appDescription}</p>
          </div>
        </div>
        
        <div className="flex space-x-2">
          <button
            onClick={onInstallCancel}
            className="px-3 py-1 text-sm text-blue-600 hover:text-blue-800"
          >
            Not now
          </button>
          <button
            onClick={handleInstall}
            disabled={isInstalling}
            className="px-4 py-2 text-sm font-medium text-white bg-blue-600 rounded hover:bg-blue-700 disabled:opacity-50"
          >
            {isInstalling ? 'Installing...' : 'Install'}
          </button>
        </div>
      </div>
    </div>
  );
};

// PWA Status Indicator Component
export const PWAStatusIndicator: React.FC = () => {
  const { isInstalled } = usePWAInstall();
  const [isStandalone, setIsStandalone] = useState(false);

  useEffect(() => {
    // Check if running in standalone mode
    const checkStandalone = () => {
      const standalone = window.matchMedia('(display-mode: standalone)').matches ||
                        (window.navigator as any).standalone === true;
      setIsStandalone(standalone);
    };

    checkStandalone();
    
    // Listen for display mode changes
    const mediaQuery = window.matchMedia('(display-mode: standalone)');
    mediaQuery.addEventListener('change', checkStandalone);

    return () => {
      mediaQuery.removeEventListener('change', checkStandalone);
    };
  }, []);

  if (!isInstalled && !isStandalone) {
    return null;
  }

  return (
    <div className="fixed bottom-4 right-4 z-50">
      <div className="flex items-center space-x-2 px-3 py-2 bg-green-100 border border-green-200 rounded-lg shadow-sm">
        <div className="w-2 h-2 bg-green-500 rounded-full animate-pulse" />
        <span className="text-sm font-medium text-green-800">
          Running as PWA
        </span>
      </div>
    </div>
  );
};
```

### 2. PWA Performance and Optimization
```typescript
// PWA Performance Optimization Strategies
interface PWAPerformanceConfig {
  enableCriticalResourcePreload: boolean;
  enableResourcePrefetch: boolean;
  enableImageOptimization: boolean;
  enableCodeSplitting: boolean;
  cacheStrategy: 'cacheFirst' | 'networkFirst' | 'staleWhileRevalidate';
}

class PWAPerformanceOptimizer {
  private config: PWAPerformanceConfig;
  private observer: PerformanceObserver | null = null;
  private metrics = new Map<string, number>();

  constructor(config: PWAPerformanceConfig) {
    this.config = config;
    this.initializePerformanceMonitoring();
  }

  // Initialize performance monitoring
  private initializePerformanceMonitoring(): void {
    if ('PerformanceObserver' in window) {
      this.observer = new PerformanceObserver((list) => {
        const entries = list.getEntries();
        
        for (const entry of entries) {
          this.processPerformanceEntry(entry);
        }
      });

      // Observe various performance entry types
      try {
        this.observer.observe({ entryTypes: ['navigation', 'paint', 'largest-contentful-paint', 'layout-shift'] });
      } catch (error) {
        console.warn('Some performance entry types not supported:', error);
      }
    }
  }

  private processPerformanceEntry(entry: PerformanceEntry): void {
    switch (entry.entryType) {
      case 'navigation':
        this.processNavigationEntry(entry as PerformanceNavigationTiming);
        break;
      case 'paint':
        this.processPaintEntry(entry);
        break;
      case 'largest-contentful-paint':
        this.metrics.set('LCP', entry.startTime);
        break;
      case 'layout-shift':
        this.processLayoutShiftEntry(entry as any);
        break;
    }
  }

  private processNavigationEntry(entry: PerformanceNavigationTiming): void {
    // Time to First Byte (TTFB)
    this.metrics.set('TTFB', entry.responseStart - entry.requestStart);
    
    // DOM Content Loaded
    this.metrics.set('DCL', entry.domContentLoadedEventEnd - entry.navigationStart);
    
    // Load Complete
    this.metrics.set('loadComplete', entry.loadEventEnd - entry.navigationStart);
  }

  private processPaintEntry(entry: PerformanceEntry): void {
    if (entry.name === 'first-paint') {
      this.metrics.set('FP', entry.startTime);
    } else if (entry.name === 'first-contentful-paint') {
      this.metrics.set('FCP', entry.startTime);
    }
  }

  private processLayoutShiftEntry(entry: any): void {
    // Cumulative Layout Shift (CLS)
    const currentCLS = this.metrics.get('CLS') || 0;
    this.metrics.set('CLS', currentCLS + entry.value);
  }

  // Preload critical resources
  preloadCriticalResources(resources: Array<{
    href: string;
    as: 'script' | 'style' | 'font' | 'image';
    type?: string;
    crossorigin?: boolean;
  }>): void {
    if (!this.config.enableCriticalResourcePreload) return;

    resources.forEach(resource => {
      const link = document.createElement('link');
      link.rel = 'preload';
      link.href = resource.href;
      link.as = resource.as;
      
      if (resource.type) {
        link.type = resource.type;
      }
      
      if (resource.crossorigin) {
        link.crossOrigin = 'anonymous';
      }

      document.head.appendChild(link);
    });
  }

  // Prefetch resources for next navigation
  prefetchResources(resources: string[]): void {
    if (!this.config.enableResourcePrefetch) return;

    resources.forEach(href => {
      const link = document.createElement('link');
      link.rel = 'prefetch';
      link.href = href;
      document.head.appendChild(link);
    });
  }

  // Lazy load images with intersection observer
  enableLazyImageLoading(): void {
    if (!this.config.enableImageOptimization) return;

    const imageObserver = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const img = entry.target as HTMLImageElement;
          
          if (img.dataset.src) {
            img.src = img.dataset.src;
            img.removeAttribute('data-src');
            imageObserver.unobserve(img);
          }
        }
      });
    }, {
      rootMargin: '50px 0px',
      threshold: 0.01,
    });

    // Observe all images with data-src attribute
    document.querySelectorAll('img[data-src]').forEach(img => {
      imageObserver.observe(img);
    });
  }

  // Implement resource hints
  addResourceHints(): void {
    const hints = [
      // DNS prefetch for external domains
      { rel: 'dns-prefetch', href: '//fonts.googleapis.com' },
      { rel: 'dns-prefetch', href: '//api.example.com' },
      
      // Preconnect to critical origins
      { rel: 'preconnect', href: 'https://fonts.gstatic.com', crossorigin: true },
      
      // Module preload for critical JS
      { rel: 'modulepreload', href: '/js/critical.js' },
    ];

    hints.forEach(hint => {
      const link = document.createElement('link');
      link.rel = hint.rel;
      link.href = hint.href;
      
      if (hint.crossorigin) {
        link.crossOrigin = 'anonymous';
      }
      
      document.head.appendChild(link);
    });
  }

  // Get performance metrics
  getMetrics(): Record<string, number> {
    return Object.fromEntries(this.metrics);
  }

  // Calculate performance score
  calculatePerformanceScore(): {
    score: number;
    details: Record<string, { score: number; threshold: number; actual: number }>;
  } {
    const thresholds = {
      FCP: { good: 1800, poor: 3000 },
      LCP: { good: 2500, poor: 4000 },
      CLS: { good: 0.1, poor: 0.25 },
      TTFB: { good: 800, poor: 1800 },
    };

    const details: Record<string, { score: number; threshold: number; actual: number }> = {};
    let totalScore = 0;
    let metricCount = 0;

    for (const [metric, threshold] of Object.entries(thresholds)) {
      const value = this.metrics.get(metric);
      
      if (value !== undefined) {
        let score: number;
        
        if (metric === 'CLS') {
          // CLS - lower is better
          score = value <= threshold.good ? 100 : 
                 value <= threshold.poor ? 50 : 0;
        } else {
          // Time-based metrics - lower is better
          score = value <= threshold.good ? 100 :
                 value <= threshold.poor ? 50 : 0;
        }

        details[metric] = {
          score,
          threshold: threshold.good,
          actual: value,
        };
        
        totalScore += score;
        metricCount++;
      }
    }

    return {
      score: metricCount > 0 ? Math.round(totalScore / metricCount) : 0,
      details,
    };
  }

  // Report Core Web Vitals
  reportCoreWebVitals(callback: (metric: {
    name: string;
    value: number;
    rating: 'good' | 'needs-improvement' | 'poor';
  }) => void): void {
    // First Contentful Paint
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      
      for (const entry of entries) {
        if (entry.name === 'first-contentful-paint') {
          callback({
            name: 'FCP',
            value: entry.startTime,
            rating: entry.startTime <= 1800 ? 'good' : 
                   entry.startTime <= 3000 ? 'needs-improvement' : 'poor',
          });
        }
      }
    });

    observer.observe({ entryTypes: ['paint'] });

    // Web Vitals library integration would be here
    // import { getFCP, getLCP, getCLS } from 'web-vitals';
  }

  // Cleanup
  disconnect(): void {
    if (this.observer) {
      this.observer.disconnect();
    }
  }
}

// React hook for PWA performance monitoring
export function usePWAPerformance(config?: Partial<PWAPerformanceConfig>) {
  const [metrics, setMetrics] = useState<Record<string, number>>({});
  const [performanceScore, setPerformanceScore] = useState<number>(0);
  const optimizerRef = useRef<PWAPerformanceOptimizer>();

  useEffect(() => {
    const fullConfig: PWAPerformanceConfig = {
      enableCriticalResourcePreload: true,
      enableResourcePrefetch: true,
      enableImageOptimization: true,
      enableCodeSplitting: true,
      cacheStrategy: 'staleWhileRevalidate',
      ...config,
    };

    optimizerRef.current = new PWAPerformanceOptimizer(fullConfig);

    // Update metrics periodically
    const updateMetrics = () => {
      if (optimizerRef.current) {
        const currentMetrics = optimizerRef.current.getMetrics();
        const scoreData = optimizerRef.current.calculatePerformanceScore();
        
        setMetrics(currentMetrics);
        setPerformanceScore(scoreData.score);
      }
    };

    const interval = setInterval(updateMetrics, 5000); // Update every 5 seconds
    updateMetrics(); // Initial update

    return () => {
      clearInterval(interval);
      optimizerRef.current?.disconnect();
    };
  }, [config]);

  const preloadCriticalResources = useCallback((resources: any[]) => {
    optimizerRef.current?.preloadCriticalResources(resources);
  }, []);

  const prefetchResources = useCallback((resources: string[]) => {
    optimizerRef.current?.prefetchResources(resources);
  }, []);

  const enableLazyLoading = useCallback(() => {
    optimizerRef.current?.enableLazyImageLoading();
  }, []);

  return {
    metrics,
    performanceScore,
    preloadCriticalResources,
    prefetchResources,
    enableLazyLoading,
  };
}

// PWA Performance Dashboard Component
export const PWAPerformanceDashboard: React.FC = () => {
  const { metrics, performanceScore } = usePWAPerformance();

  const getScoreColor = (score: number): string => {
    if (score >= 90) return 'text-green-600';
    if (score >= 50) return 'text-yellow-600';
    return 'text-red-600';
  };

  const getScoreBgColor = (score: number): string => {
    if (score >= 90) return 'bg-green-100';
    if (score >= 50) return 'bg-yellow-100';
    return 'bg-red-100';
  };

  return (
    <div className="pwa-performance-dashboard p-6 bg-white rounded-lg shadow">
      <h2 className="text-2xl font-bold mb-6">PWA Performance</h2>
      
      {/* Overall Score */}
      <div className={`mb-6 p-4 rounded-lg ${getScoreBgColor(performanceScore)}`}>
        <div className="flex items-center justify-between">
          <span className="text-lg font-medium">Performance Score</span>
          <span className={`text-3xl font-bold ${getScoreColor(performanceScore)}`}>
            {performanceScore}
          </span>
        </div>
      </div>

      {/* Core Web Vitals */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        {Object.entries(metrics).map(([metric, value]) => (
          <div key={metric} className="p-4 border rounded-lg">
            <div className="text-sm font-medium text-gray-600 uppercase">
              {metric}
            </div>
            <div className="text-2xl font-bold mt-1">
              {metric === 'CLS' ? value.toFixed(3) : `${Math.round(value)}ms`}
            </div>
            <div className="text-xs text-gray-500 mt-1">
              {getMetricDescription(metric)}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};

function getMetricDescription(metric: string): string {
  const descriptions = {
    FCP: 'First Contentful Paint',
    LCP: 'Largest Contentful Paint',
    CLS: 'Cumulative Layout Shift',
    TTFB: 'Time to First Byte',
    DCL: 'DOM Content Loaded',
    loadComplete: 'Load Complete',
  };
  
  return descriptions[metric] || metric;
}
```

### 3. App Shell Architecture
```typescript
// App Shell Architecture Implementation
interface AppShellConfig {
  shellFiles: string[];
  contentRoutes: string[];
  cacheStrategy: 'shell-first' | 'content-first' | 'network-first';
  offlinePageUrl: string;
  enableShellUpdates: boolean;
}

class AppShellManager {
  private config: AppShellConfig;
  private shellCache = 'app-shell-v1';
  private contentCache = 'app-content-v1';

  constructor(config: AppShellConfig) {
    this.config = config;
  }

  // Cache app shell resources
  async cacheAppShell(): Promise<void> {
    try {
      const cache = await caches.open(this.shellCache);
      
      // Add shell files to cache
      await cache.addAll(this.config.shellFiles);
      
      console.log('App shell cached successfully');
    } catch (error) {
      console.error('Failed to cache app shell:', error);
      throw error;
    }
  }

  // Serve from shell cache
  async serveFromShell(request: Request): Promise<Response | null> {
    try {
      const cache = await caches.open(this.shellCache);
      const response = await cache.match(request);
      
      if (response) {
        return response;
      }

      // Fallback to index.html for navigation requests
      if (request.mode === 'navigate') {
        const shellResponse = await cache.match('/');
        if (shellResponse) {
          return shellResponse;
        }
      }

      return null;
    } catch (error) {
      console.error('Error serving from shell cache:', error);
      return null;
    }
  }

  // Update app shell
  async updateAppShell(): Promise<boolean> {
    if (!this.config.enableShellUpdates) {
      return false;
    }

    try {
      // Check for updates by comparing ETags or timestamps
      const updateCheckResponse = await fetch('/api/shell-version');
      const remoteVersion = await updateCheckResponse.json();
      
      const cache = await caches.open(this.shellCache);
      const cachedVersionResponse = await cache.match('/api/shell-version');
      
      let needsUpdate = true;
      
      if (cachedVersionResponse) {
        const cachedVersion = await cachedVersionResponse.json();
        needsUpdate = remoteVersion.version !== cachedVersion.version;
      }

      if (needsUpdate) {
        // Update shell cache
        await this.cacheAppShell();
        
        // Cache new version info
        await cache.put('/api/shell-version', new Response(JSON.stringify(remoteVersion)));
        
        return true;
      }

      return false;
    } catch (error) {
      console.error('Failed to update app shell:', error);
      return false;
    }
  }

  // Handle content caching
  async cacheContent(request: Request, response: Response): Promise<void> {
    // Only cache GET requests
    if (request.method !== 'GET') {
      return;
    }

    // Skip caching for certain URLs
    const url = new URL(request.url);
    if (url.pathname.startsWith('/api/') && !url.pathname.includes('/api/content/')) {
      return;
    }

    try {
      const cache = await caches.open(this.contentCache);
      await cache.put(request, response.clone());
    } catch (error) {
      console.error('Failed to cache content:', error);
    }
  }

  // Serve content with appropriate strategy
  async serveContent(request: Request): Promise<Response> {
    const url = new URL(request.url);
    
    switch (this.config.cacheStrategy) {
      case 'shell-first':
        return this.shellFirstStrategy(request);
      case 'content-first':
        return this.contentFirstStrategy(request);
      case 'network-first':
        return this.networkFirstStrategy(request);
      default:
        return this.shellFirstStrategy(request);
    }
  }

  private async shellFirstStrategy(request: Request): Promise<Response> {
    // Try shell cache first
    const shellResponse = await this.serveFromShell(request);
    if (shellResponse) {
      return shellResponse;
    }

    // Try content cache
    const contentCache = await caches.open(this.contentCache);
    const cachedResponse = await contentCache.match(request);
    if (cachedResponse) {
      return cachedResponse;
    }

    // Fallback to network
    try {
      const networkResponse = await fetch(request);
      
      if (networkResponse.ok) {
        await this.cacheContent(request, networkResponse);
      }
      
      return networkResponse;
    } catch (error) {
      // Return offline page as last resort
      return this.getOfflinePage();
    }
  }

  private async contentFirstStrategy(request: Request): Promise<Response> {
    // Try content cache first
    const contentCache = await caches.open(this.contentCache);
    const cachedResponse = await contentCache.match(request);
    
    if (cachedResponse) {
      // Fetch from network in background to update cache
      this.updateCacheInBackground(request);
      return cachedResponse;
    }

    // Try network
    try {
      const networkResponse = await fetch(request);
      
      if (networkResponse.ok) {
        await this.cacheContent(request, networkResponse);
      }
      
      return networkResponse;
    } catch (error) {
      // Try shell cache as fallback
      const shellResponse = await this.serveFromShell(request);
      if (shellResponse) {
        return shellResponse;
      }

      return this.getOfflinePage();
    }
  }

  private async networkFirstStrategy(request: Request): Promise<Response> {
    try {
      const networkResponse = await fetch(request);
      
      if (networkResponse.ok) {
        await this.cacheContent(request, networkResponse);
      }
      
      return networkResponse;
    } catch (error) {
      // Try content cache
      const contentCache = await caches.open(this.contentCache);
      const cachedResponse = await contentCache.match(request);
      if (cachedResponse) {
        return cachedResponse;
      }

      // Try shell cache
      const shellResponse = await this.serveFromShell(request);
      if (shellResponse) {
        return shellResponse;
      }

      return this.getOfflinePage();
    }
  }

  private async updateCacheInBackground(request: Request): Promise<void> {
    try {
      const networkResponse = await fetch(request);
      
      if (networkResponse.ok) {
        await this.cacheContent(request, networkResponse);
      }
    } catch (error) {
      // Silent failure for background updates
      console.warn('Background cache update failed:', error);
    }
  }

  private async getOfflinePage(): Promise<Response> {
    const cache = await caches.open(this.shellCache);
    const offlineResponse = await cache.match(this.config.offlinePageUrl);
    
    if (offlineResponse) {
      return offlineResponse;
    }

    // Return basic offline response
    return new Response(
      `
      <!DOCTYPE html>
      <html>
        <head>
          <title>Offline</title>
          <meta charset="utf-8">
          <meta name="viewport" content="width=device-width, initial-scale=1">
        </head>
        <body>
          <div style="text-align: center; padding: 50px;">
            <h1>You're offline</h1>
            <p>Please check your internet connection and try again.</p>
          </div>
        </body>
      </html>
      `,
      {
        status: 200,
        headers: { 'Content-Type': 'text/html' },
      }
    );
  }

  // Clean up old caches
  async cleanupOldCaches(currentCaches: string[]): Promise<void> {
    try {
      const cacheNames = await caches.keys();
      
      const deletePromises = cacheNames
        .filter(cacheName => !currentCaches.includes(cacheName))
        .map(cacheName => caches.delete(cacheName));

      await Promise.all(deletePromises);
      
      console.log('Old caches cleaned up');
    } catch (error) {
      console.error('Failed to cleanup old caches:', error);
    }
  }
}

// React Hook for App Shell Management
export function useAppShell(config: AppShellConfig) {
  const [shellManager] = useState(() => new AppShellManager(config));
  const [isShellCached, setIsShellCached] = useState(false);
  const [hasUpdates, setHasUpdates] = useState(false);

  useEffect(() => {
    const initializeShell = async () => {
      try {
        await shellManager.cacheAppShell();
        setIsShellCached(true);

        // Check for updates
        const hasUpdate = await shellManager.updateAppShell();
        setHasUpdates(hasUpdate);
      } catch (error) {
        console.error('Failed to initialize app shell:', error);
      }
    };

    initializeShell();
  }, [shellManager]);

  const updateShell = useCallback(async () => {
    try {
      const updated = await shellManager.updateAppShell();
      setHasUpdates(false);
      return updated;
    } catch (error) {
      console.error('Failed to update shell:', error);
      return false;
    }
  }, [shellManager]);

  return {
    isShellCached,
    hasUpdates,
    updateShell,
    shellManager,
  };
}

// App Shell Update Notification Component
interface AppShellUpdateNotificationProps {
  onUpdate: () => Promise<void>;
  onDismiss: () => void;
}

export const AppShellUpdateNotification: React.FC<AppShellUpdateNotificationProps> = ({
  onUpdate,
  onDismiss,
}) => {
  const [isUpdating, setIsUpdating] = useState(false);

  const handleUpdate = async () => {
    setIsUpdating(true);
    try {
      await onUpdate();
      window.location.reload(); // Reload to use new shell
    } catch (error) {
      console.error('Update failed:', error);
    } finally {
      setIsUpdating(false);
    }
  };

  return (
    <div className="fixed bottom-4 left-4 right-4 z-50 md:left-auto md:w-96">
      <div className="bg-blue-600 text-white p-4 rounded-lg shadow-lg">
        <div className="flex items-start justify-between">
          <div className="flex-1">
            <h3 className="font-semibold">App Update Available</h3>
            <p className="text-sm opacity-90 mt-1">
              A new version of the app is available. Update now for the latest features.
            </p>
          </div>
          <button
            onClick={onDismiss}
            className="ml-2 text-white opacity-70 hover:opacity-100"
          >
            ×
          </button>
        </div>
        
        <div className="flex space-x-2 mt-4">
          <button
            onClick={onDismiss}
            className="px-3 py-1 text-sm text-blue-100 hover:text-white"
          >
            Later
          </button>
          <button
            onClick={handleUpdate}
            disabled={isUpdating}
            className="px-4 py-2 text-sm font-medium bg-white text-blue-600 rounded hover:bg-blue-50 disabled:opacity-50"
          >
            {isUpdating ? 'Updating...' : 'Update Now'}
          </button>
        </div>
      </div>
    </div>
  );
};
```

## Interview-Ready Summary

**Progressive Web Apps (PWA) Implementation requires:**

1. **PWA Fundamentals** - Web App Manifest configuration, installation prompts, app shell architecture, platform-specific optimizations
2. **Performance Optimization** - Critical resource preloading, lazy loading, Core Web Vitals monitoring, performance scoring
3. **App Shell Pattern** - Shell-first caching strategy, content management, offline fallbacks, cache cleanup
4. **Installation Management** - Install prompts, installation detection, platform compatibility, update notifications

**Key PWA features:**
- **App-like Experience** - Standalone display mode, native-like navigation, platform integration
- **Offline Functionality** - Service Worker caching, offline pages, background sync
- **Performance** - Fast loading, smooth animations, optimized resource delivery
- **Installability** - Add to home screen, app store distribution, automatic updates

**PWA requirements:** HTTPS, Web App Manifest, Service Worker, responsive design, proper icon sets, offline functionality.

**Best practices:** Implement app shell architecture, optimize critical rendering path, provide offline fallbacks, handle installation gracefully, monitor Core Web Vitals, support multiple platforms, enable background updates, cache strategically.