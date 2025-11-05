# Build Tools and Bundlers Deep Dive (Vite, Webpack, esbuild, Rollup)

Modern build tools have revolutionized frontend development with faster builds, better developer experience, and optimized production outputs. This comprehensive guide covers Vite, Webpack, esbuild, and Rollup - their architectures, optimization strategies, plugin ecosystems, and enterprise deployment patterns.

## Vite: Next-Generation Frontend Tooling

### Vite Configuration and Optimization

```typescript
// vite.config.ts
import { defineConfig, loadEnv, splitVendorChunkPlugin } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';
import { visualizer } from 'rollup-plugin-visualizer';
import { createHtmlPlugin } from 'vite-plugin-html';
import legacy from '@vitejs/plugin-legacy';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig(({ mode, command }) => {
  const env = loadEnv(mode, process.cwd(), '');
  const isProduction = mode === 'production';
  const isDevelopment = command === 'serve';

  return {
    // Base configuration
    base: env.VITE_BASE_URL || '/',
    publicDir: 'public',
    
    // Development server configuration
    server: {
      host: true,
      port: parseInt(env.VITE_PORT) || 3000,
      open: true,
      cors: true,
      hmr: {
        overlay: true,
      },
      proxy: {
        '/api': {
          target: env.VITE_API_URL || 'http://localhost:8080',
          changeOrigin: true,
          secure: false,
          rewrite: (path) => path.replace(/^\/api/, ''),
        },
        '/ws': {
          target: 'ws://localhost:8080',
          ws: true,
        },
      },
    },

    // Preview server configuration
    preview: {
      host: true,
      port: 4173,
      cors: true,
    },

    // Build configuration
    build: {
      target: ['es2020', 'edge88', 'firefox78', 'chrome87', 'safari14'],
      outDir: 'dist',
      assetsDir: 'assets',
      sourcemap: isProduction ? 'hidden' : true,
      minify: isProduction ? 'esbuild' : false,
      
      // Chunk splitting strategy
      rollupOptions: {
        input: {
          main: resolve(__dirname, 'index.html'),
          admin: resolve(__dirname, 'admin.html'),
        },
        output: {
          chunkFileNames: (chunkInfo) => {
            if (chunkInfo.name?.includes('vendor')) {
              return 'js/vendor/[name]-[hash].js';
            }
            return 'js/[name]-[hash].js';
          },
          entryFileNames: 'js/[name]-[hash].js',
          assetFileNames: (assetInfo) => {
            const extType = assetInfo.name?.split('.').pop();
            if (/png|jpe?g|svg|gif|tiff|bmp|ico/i.test(extType || '')) {
              return 'images/[name]-[hash][extname]';
            }
            if (/css/i.test(extType || '')) {
              return 'css/[name]-[hash][extname]';
            }
            return 'assets/[name]-[hash][extname]';
          },
          manualChunks: {
            // Vendor chunks
            react: ['react', 'react-dom'],
            router: ['react-router-dom'],
            ui: ['@mui/material', '@mui/icons-material'],
            utils: ['lodash', 'date-fns', 'axios'],
            
            // Feature-based chunks
            auth: ['./src/features/auth'],
            dashboard: ['./src/features/dashboard'],
            analytics: ['./src/features/analytics'],
          },
        },
        external: isProduction ? [] : ['react', 'react-dom'],
      },

      // Performance optimization
      chunkSizeWarningLimit: 1000,
      reportCompressedSize: false,
      
      // CSS optimization
      cssCodeSplit: true,
      cssMinify: isProduction,
    },

    // Dependency optimization
    optimizeDeps: {
      include: [
        'react',
        'react-dom',
        'react-router-dom',
        '@mui/material',
        'axios',
        'lodash',
      ],
      exclude: ['@vite/client', '@vite/env'],
      esbuildOptions: {
        target: 'es2020',
        supported: {
          bigint: true,
        },
      },
    },

    // Path resolution
    resolve: {
      alias: {
        '@': resolve(__dirname, 'src'),
        '@components': resolve(__dirname, 'src/components'),
        '@hooks': resolve(__dirname, 'src/hooks'),
        '@utils': resolve(__dirname, 'src/utils'),
        '@types': resolve(__dirname, 'src/types'),
        '@assets': resolve(__dirname, 'src/assets'),
        '@styles': resolve(__dirname, 'src/styles'),
      },
      extensions: ['.ts', '.tsx', '.js', '.jsx', '.json'],
    },

    // Environment variables
    define: {
      __APP_VERSION__: JSON.stringify(process.env.npm_package_version),
      __BUILD_TIME__: JSON.stringify(new Date().toISOString()),
    },

    // CSS configuration
    css: {
      modules: {
        scopeBehaviour: 'local',
        generateScopedName: isDevelopment
          ? '[name]__[local]___[hash:base64:5]'
          : '[hash:base64:8]',
      },
      preprocessorOptions: {
        scss: {
          additionalData: `@import "@/styles/variables.scss";`,
        },
        less: {
          modifyVars: {
            '@primary-color': '#1DA57A',
          },
          javascriptEnabled: true,
        },
      },
      postCSS: {
        plugins: [
          require('autoprefixer'),
          ...(isProduction ? [require('cssnano')] : []),
        ],
      },
    },

    // Plugins
    plugins: [
      // React support
      react({
        babel: {
          plugins: [
            ['@babel/plugin-proposal-decorators', { legacy: true }],
            ['@babel/plugin-transform-class-properties', { loose: true }],
          ],
        },
      }),

      // HTML template processing
      createHtmlPlugin({
        minify: isProduction,
        inject: {
          data: {
            title: env.VITE_APP_TITLE || 'My App',
            description: env.VITE_APP_DESCRIPTION || 'Built with Vite',
          },
        },
      }),

      // Progressive Web App
      VitePWA({
        registerType: 'autoUpdate',
        includeAssets: ['favicon.ico', 'apple-touch-icon.png'],
        manifest: {
          name: env.VITE_APP_TITLE || 'My App',
          short_name: 'MyApp',
          description: env.VITE_APP_DESCRIPTION || 'Built with Vite',
          theme_color: '#ffffff',
          icons: [
            {
              src: 'pwa-192x192.png',
              sizes: '192x192',
              type: 'image/png',
            },
            {
              src: 'pwa-512x512.png',
              sizes: '512x512',
              type: 'image/png',
            },
          ],
        },
        workbox: {
          globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
          runtimeCaching: [
            {
              urlPattern: /^https:\/\/api\.example\.com\//,
              handler: 'CacheFirst',
              options: {
                cacheName: 'api-cache',
                expiration: {
                  maxEntries: 100,
                  maxAgeSeconds: 60 * 60 * 24, // 1 day
                },
              },
            },
          ],
        },
      }),

      // Legacy browser support
      legacy({
        targets: ['defaults', 'not IE 11'],
        additionalLegacyPolyfills: ['regenerator-runtime/runtime'],
        renderLegacyChunks: true,
        polyfills: [
          'es.symbol',
          'es.array.filter',
          'es.promise',
          'es.promise.finally',
          'es/map',
          'es/set',
          'es.array.for-each',
          'es.object.define-properties',
          'es.object.define-property',
          'es.object.get-own-property-descriptor',
          'es.object.get-own-property-descriptors',
          'es.object.keys',
          'es.object.to-string',
          'web.dom-collections.for-each',
          'esnext.global-this',
          'esnext.string.match-all',
        ],
      }),

      // Bundle analyzer
      ...(env.ANALYZE ? [
        visualizer({
          filename: 'dist/stats.html',
          open: true,
          gzipSize: true,
          brotliSize: true,
        }),
      ] : []),

      // Vendor chunk splitting
      splitVendorChunkPlugin(),
    ],

    // ESBuild configuration
    esbuild: {
      target: 'es2020',
      drop: isProduction ? ['console', 'debugger'] : [],
      pure: isProduction ? ['console.log', 'console.info'] : [],
      legalComments: 'none',
    },

    // Worker configuration
    worker: {
      format: 'es',
      plugins: [],
    },

    // Test configuration (for Vitest)
    test: {
      globals: true,
      environment: 'jsdom',
      setupFiles: ['./src/test/setup.ts'],
      include: ['src/**/*.{test,spec}.{js,ts,jsx,tsx}'],
      coverage: {
        provider: 'v8',
        reporter: ['text', 'html', 'json'],
        exclude: [
          'node_modules/',
          'src/test/',
          '**/*.d.ts',
          '**/*.config.{js,ts}',
        ],
      },
    },
  };
});
```

### Advanced Vite Plugins

```typescript
// plugins/vite-plugin-env-validation.ts
import { Plugin } from 'vite';
import { z } from 'zod';

interface EnvValidationOptions {
  schema: z.ZodObject<any>;
  envPrefix?: string;
}

export function envValidation({ schema, envPrefix = 'VITE_' }: EnvValidationOptions): Plugin {
  return {
    name: 'env-validation',
    configResolved(config) {
      const envVars = Object.keys(process.env)
        .filter(key => key.startsWith(envPrefix))
        .reduce((acc, key) => {
          acc[key] = process.env[key];
          return acc;
        }, {} as Record<string, string | undefined>);

      try {
        schema.parse(envVars);
        console.log('✅ Environment variables validated successfully');
      } catch (error) {
        console.error('❌ Environment validation failed:', error);
        process.exit(1);
      }
    },
  };
}

// plugins/vite-plugin-api-mock.ts
export function apiMock(options: { 
  enabled?: boolean;
  mockDir?: string;
  patterns?: string[];
}): Plugin {
  const { enabled = true, mockDir = 'mocks', patterns = ['/api/**'] } = options;

  if (!enabled) return { name: 'api-mock-disabled' };

  return {
    name: 'api-mock',
    configureServer(server) {
      server.middlewares.use('/api', (req, res, next) => {
        const mockFile = path.join(mockDir, req.url + '.json');
        
        if (fs.existsSync(mockFile)) {
          const mockData = JSON.parse(fs.readFileSync(mockFile, 'utf-8'));
          res.setHeader('Content-Type', 'application/json');
          res.end(JSON.stringify(mockData));
        } else {
          next();
        }
      });
    },
  };
}

// plugins/vite-plugin-feature-flags.ts
export function featureFlags(flags: Record<string, boolean>): Plugin {
  return {
    name: 'feature-flags',
    transform(code, id) {
      if (id.includes('node_modules')) return;

      let transformedCode = code;
      
      Object.entries(flags).forEach(([flag, enabled]) => {
        const regex = new RegExp(`__FEATURE_${flag.toUpperCase()}__`, 'g');
        transformedCode = transformedCode.replace(regex, String(enabled));
      });

      return transformedCode;
    },
  };
}

// Usage in vite.config.ts
const envSchema = z.object({
  VITE_API_URL: z.string().url(),
  VITE_APP_TITLE: z.string().min(1),
  VITE_ENABLE_ANALYTICS: z.enum(['true', 'false']).optional(),
});

export default defineConfig({
  plugins: [
    // ... other plugins
    envValidation({ schema: envSchema }),
    apiMock({ 
      enabled: process.env.NODE_ENV === 'development',
      mockDir: './src/mocks',
    }),
    featureFlags({
      NEW_DASHBOARD: true,
      EXPERIMENTAL_FEATURE: false,
    }),
  ],
});
```

## Webpack Advanced Configuration

### Production-Optimized Webpack Setup

```javascript
// webpack.config.js
const path = require('path');
const webpack = require('webpack');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
const CompressionPlugin = require('compression-webpack-plugin');
const WorkboxPlugin = require('workbox-webpack-plugin');
const ESLintPlugin = require('eslint-webpack-plugin');
const ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin');

const isProduction = process.env.NODE_ENV === 'production';

module.exports = {
  mode: isProduction ? 'production' : 'development',
  
  entry: {
    main: './src/index.tsx',
    vendor: ['react', 'react-dom', 'react-router-dom'],
    polyfills: './src/polyfills.ts',
  },

  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: isProduction 
      ? 'js/[name].[contenthash:8].js'
      : 'js/[name].js',
    chunkFilename: isProduction
      ? 'js/[name].[contenthash:8].chunk.js'
      : 'js/[name].chunk.js',
    assetModuleFilename: 'assets/[hash][ext][query]',
    clean: true,
    publicPath: process.env.PUBLIC_PATH || '/',
    
    // Enable module federation
    library: {
      type: 'module',
    },
  },

  experiments: {
    outputModule: true,
    topLevelAwait: true,
  },

  resolve: {
    extensions: ['.tsx', '.ts', '.jsx', '.js', '.json'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components'),
      '@hooks': path.resolve(__dirname, 'src/hooks'),
      '@utils': path.resolve(__dirname, 'src/utils'),
      '@assets': path.resolve(__dirname, 'src/assets'),
    },
    fallback: {
      // Node.js polyfills for browser
      buffer: require.resolve('buffer'),
      crypto: require.resolve('crypto-browserify'),
      stream: require.resolve('stream-browserify'),
      util: require.resolve('util'),
    },
  },

  module: {
    rules: [
      // TypeScript/JavaScript
      {
        test: /\.(ts|tsx|js|jsx)$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'babel-loader',
            options: {
              presets: [
                [
                  '@babel/preset-env',
                  {
                    useBuiltIns: 'usage',
                    corejs: 3,
                    targets: {
                      browsers: ['last 2 versions', 'ie >= 11'],
                    },
                  },
                ],
                '@babel/preset-react',
                '@babel/preset-typescript',
              ],
              plugins: [
                '@babel/plugin-proposal-class-properties',
                '@babel/plugin-transform-runtime',
                ['babel-plugin-styled-components', { displayName: !isProduction }],
                ...(isProduction ? [] : ['react-refresh/babel']),
              ],
              cacheDirectory: true,
            },
          },
        ],
      },

      // CSS/SCSS
      {
        test: /\.(css|scss|sass)$/,
        use: [
          isProduction ? MiniCssExtractPlugin.loader : 'style-loader',
          {
            loader: 'css-loader',
            options: {
              modules: {
                auto: true,
                localIdentName: isProduction
                  ? '[hash:base64:8]'
                  : '[name]__[local]___[hash:base64:5]',
              },
              sourceMap: !isProduction,
            },
          },
          {
            loader: 'postcss-loader',
            options: {
              postcssOptions: {
                plugins: [
                  'autoprefixer',
                  ...(isProduction ? ['cssnano'] : []),
                ],
              },
            },
          },
          {
            loader: 'sass-loader',
            options: {
              implementation: require('sass'),
              additionalData: `@import "@/styles/variables.scss";`,
            },
          },
        ],
      },

      // Images
      {
        test: /\.(png|jpe?g|gif|svg|webp)$/i,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024, // 8kb
          },
        },
        generator: {
          filename: 'images/[name].[hash:8][ext]',
        },
      },

      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/i,
        type: 'asset/resource',
        generator: {
          filename: 'fonts/[name].[hash:8][ext]',
        },
      },

      // Other assets
      {
        test: /\.(pdf|txt|xml)$/i,
        type: 'asset/resource',
        generator: {
          filename: 'assets/[name].[hash:8][ext]',
        },
      },
    ],
  },

  optimization: {
    minimize: isProduction,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: isProduction,
            drop_debugger: isProduction,
            pure_funcs: isProduction ? ['console.log', 'console.info'] : [],
          },
          mangle: {
            safari10: true,
          },
          format: {
            comments: false,
          },
        },
        extractComments: false,
      }),
      new CssMinimizerPlugin({
        minimizerOptions: {
          preset: [
            'default',
            {
              discardComments: { removeAll: true },
            },
          ],
        },
      }),
    ],

    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true,
        },
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: -10,
          chunks: 'all',
        },
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          priority: 20,
          chunks: 'all',
        },
        common: {
          name: 'common',
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
          chunks: 'all',
        },
      },
    },

    runtimeChunk: {
      name: entrypoint => `runtime-${entrypoint.name}`,
    },

    moduleIds: 'deterministic',
    chunkIds: 'deterministic',
  },

  plugins: [
    // HTML generation
    new HtmlWebpackPlugin({
      template: './public/index.html',
      filename: 'index.html',
      inject: true,
      minify: isProduction ? {
        removeComments: true,
        collapseWhitespace: true,
        removeRedundantAttributes: true,
        useShortDoctype: true,
        removeEmptyAttributes: true,
        removeStyleLinkTypeAttributes: true,
        keepClosingSlash: true,
        minifyJS: true,
        minifyCSS: true,
        minifyURLs: true,
      } : false,
    }),

    // CSS extraction
    ...(isProduction ? [
      new MiniCssExtractPlugin({
        filename: 'css/[name].[contenthash:8].css',
        chunkFilename: 'css/[name].[contenthash:8].chunk.css',
      }),
    ] : []),

    // Environment variables
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      'process.env.PUBLIC_URL': JSON.stringify(process.env.PUBLIC_URL || ''),
      __VERSION__: JSON.stringify(require('./package.json').version),
      __BUILD_TIME__: JSON.stringify(new Date().toISOString()),
    }),

    // TypeScript checking
    new ForkTsCheckerWebpackPlugin({
      async: !isProduction,
      typescript: {
        configFile: path.resolve(__dirname, 'tsconfig.json'),
      },
    }),

    // ESLint
    new ESLintPlugin({
      extensions: ['js', 'jsx', 'ts', 'tsx'],
      exclude: 'node_modules',
      cache: true,
      cacheLocation: path.resolve(__dirname, '.eslintcache'),
    }),

    // Compression
    ...(isProduction ? [
      new CompressionPlugin({
        algorithm: 'gzip',
        test: /\.(js|css|html|svg)$/,
        threshold: 8192,
        minRatio: 0.8,
      }),
    ] : []),

    // Progressive Web App
    ...(isProduction ? [
      new WorkboxPlugin.GenerateSW({
        clientsClaim: true,
        skipWaiting: true,
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/fonts\.googleapis\.com/,
            handler: 'StaleWhileRevalidate',
            options: {
              cacheName: 'google-fonts-stylesheets',
            },
          },
          {
            urlPattern: /^https:\/\/fonts\.gstatic\.com/,
            handler: 'CacheFirst',
            options: {
              cacheName: 'google-fonts-webfonts',
              expiration: {
                maxEntries: 30,
                maxAgeSeconds: 60 * 60 * 24 * 365, // 1 year
              },
            },
          },
        ],
      }),
    ] : []),

    // Bundle analysis
    ...(process.env.ANALYZE ? [
      new BundleAnalyzerPlugin({
        analyzerMode: 'static',
        openAnalyzer: false,
        reportFilename: 'bundle-report.html',
      }),
    ] : []),

    // Hot module replacement for development
    ...(isProduction ? [] : [
      new webpack.HotModuleReplacementPlugin(),
    ]),
  ],

  devServer: {
    static: {
      directory: path.join(__dirname, 'public'),
    },
    compress: true,
    port: parseInt(process.env.PORT) || 3000,
    hot: true,
    open: true,
    historyApiFallback: true,
    client: {
      overlay: {
        errors: true,
        warnings: false,
      },
    },
    proxy: {
      '/api': {
        target: process.env.API_URL || 'http://localhost:8080',
        changeOrigin: true,
        secure: false,
      },
    },
  },

  devtool: isProduction ? 'source-map' : 'eval-source-map',

  stats: {
    preset: 'normal',
    moduleTrace: true,
    errorDetails: true,
  },

  performance: {
    hints: isProduction ? 'warning' : false,
    maxEntrypointSize: 250000,
    maxAssetSize: 250000,
  },

  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },
};
```

### Custom Webpack Loaders and Plugins

```javascript
// loaders/svg-sprite-loader.js
const loaderUtils = require('loader-utils');
const SVGSpriter = require('svg-sprite');

module.exports = function(content) {
  const callback = this.async();
  const options = loaderUtils.getOptions(this) || {};

  const spriter = new SVGSpriter({
    mode: {
      symbol: {
        dest: '.',
        sprite: 'sprite.svg',
        inline: true,
      },
    },
  });

  spriter.add(this.resourcePath, null, content);
  
  spriter.compile((error, result) => {
    if (error) {
      return callback(error);
    }

    const sprite = result.symbol.sprite;
    callback(null, `export default ${JSON.stringify(sprite.contents.toString())};`);
  });
};

// plugins/webpack-plugin-manifest.js
class ManifestPlugin {
  constructor(options = {}) {
    this.options = {
      fileName: 'manifest.json',
      writeToFileEmit: true,
      ...options,
    };
  }

  apply(compiler) {
    compiler.hooks.emit.tapAsync('ManifestPlugin', (compilation, callback) => {
      const manifest = {};
      
      // Extract entry points
      for (const [name, entrypoint] of compilation.entrypoints) {
        manifest[name] = {
          js: entrypoint.getFiles().filter(file => file.endsWith('.js')),
          css: entrypoint.getFiles().filter(file => file.endsWith('.css')),
        };
      }

      // Add chunks
      for (const chunk of compilation.chunks) {
        if (chunk.name && !manifest[chunk.name]) {
          manifest[chunk.name] = {
            js: chunk.files.filter(file => file.endsWith('.js')),
            css: chunk.files.filter(file => file.endsWith('.css')),
          };
        }
      }

      const manifestContent = JSON.stringify(manifest, null, 2);
      
      compilation.assets[this.options.fileName] = {
        source: () => manifestContent,
        size: () => manifestContent.length,
      };

      callback();
    });
  }
}

module.exports = ManifestPlugin;

// plugins/webpack-plugin-critical-css.js
const critical = require('critical');

class CriticalCSSPlugin {
  constructor(options = {}) {
    this.options = {
      inline: true,
      minify: true,
      extract: false,
      dimensions: [
        { width: 1200, height: 900 },
        { width: 320, height: 568 },
      ],
      ...options,
    };
  }

  apply(compiler) {
    compiler.hooks.afterEmit.tapAsync('CriticalCSSPlugin', (compilation, callback) => {
      const htmlFiles = Object.keys(compilation.assets).filter(file => 
        file.endsWith('.html')
      );

      const promises = htmlFiles.map(file => {
        const htmlPath = path.join(compilation.outputOptions.path, file);
        
        return critical.generate({
          ...this.options,
          base: compilation.outputOptions.path,
          src: file,
          dest: file,
        });
      });

      Promise.all(promises)
        .then(() => callback())
        .catch(callback);
    });
  }
}

module.exports = CriticalCSSPlugin;
```

## esbuild Integration and Optimization

### esbuild Configuration

```javascript
// build.js - esbuild configuration
const esbuild = require('esbuild');
const { sassPlugin } = require('esbuild-sass-plugin');
const { wasmLoader } = require('esbuild-plugin-wasm');
const { copy } = require('esbuild-plugin-copy');

const isProduction = process.env.NODE_ENV === 'production';

// Custom plugin for environment variables
const envPlugin = {
  name: 'env',
  setup(build) {
    build.onResolve({ filter: /^env$/ }, args => ({
      path: args.path,
      namespace: 'env-ns',
    }));

    build.onLoad({ filter: /.*/, namespace: 'env-ns' }, () => ({
      contents: JSON.stringify(process.env),
      loader: 'json',
    }));
  },
};

// Custom plugin for virtual modules
const virtualModulePlugin = {
  name: 'virtual-module',
  setup(build) {
    build.onResolve({ filter: /^virtual:/ }, args => ({
      path: args.path,
      namespace: 'virtual',
    }));

    build.onLoad({ filter: /^virtual:feature-flags$/, namespace: 'virtual' }, () => ({
      contents: `
        export const FEATURE_FLAGS = {
          NEW_DASHBOARD: ${process.env.FEATURE_NEW_DASHBOARD === 'true'},
          DARK_MODE: ${process.env.FEATURE_DARK_MODE === 'true'},
          ANALYTICS: ${process.env.FEATURE_ANALYTICS === 'true'},
        };
      `,
      loader: 'js',
    }));
  },
};

// Custom plugin for TypeScript path mapping
const pathMappingPlugin = {
  name: 'path-mapping',
  setup(build) {
    const path = require('path');
    
    build.onResolve({ filter: /^@\// }, args => ({
      path: path.resolve(__dirname, 'src', args.path.slice(2)),
    }));
  },
};

async function build() {
  try {
    const result = await esbuild.build({
      entryPoints: {
        main: './src/index.tsx',
        worker: './src/worker.ts',
      },
      bundle: true,
      minify: isProduction,
      sourcemap: true,
      target: ['chrome90', 'firefox88', 'safari14'],
      format: 'esm',
      splitting: true,
      outdir: 'dist',
      
      // Asset handling
      assetNames: 'assets/[name]-[hash]',
      chunkNames: 'chunks/[name]-[hash]',
      entryNames: '[dir]/[name]-[hash]',
      
      // Loader configuration
      loader: {
        '.png': 'file',
        '.jpg': 'file',
        '.jpeg': 'file',
        '.svg': 'dataurl',
        '.gif': 'file',
        '.woff': 'file',
        '.woff2': 'file',
        '.eot': 'file',
        '.ttf': 'file',
      },

      // External dependencies (for CDN loading)
      external: isProduction ? [] : ['react', 'react-dom'],

      // Plugin configuration
      plugins: [
        envPlugin,
        virtualModulePlugin,
        pathMappingPlugin,
        
        // SASS support
        sassPlugin({
          filter: /\.(s[ac]ss|css)$/,
          type: 'css',
          cache: true,
        }),

        // WASM support
        wasmLoader(),

        // File copying
        copy({
          assets: [
            {
              from: './public/*',
              to: './dist',
            },
          ],
        }),

        // Custom HTML generation plugin
        {
          name: 'html',
          setup(build) {
            build.onEnd(result => {
              const fs = require('fs');
              const path = require('path');
              
              const htmlTemplate = `
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My App</title>
  ${result.outputFiles
    ?.filter(file => file.path.endsWith('.css'))
    .map(file => `<link rel="stylesheet" href="${path.basename(file.path)}">`)
    .join('\n  ') || ''}
</head>
<body>
  <div id="root"></div>
  ${result.outputFiles
    ?.filter(file => file.path.endsWith('.js') && !file.path.includes('worker'))
    .map(file => `<script type="module" src="${path.basename(file.path)}"></script>`)
    .join('\n  ') || ''}
</body>
</html>`;

              fs.writeFileSync(
                path.join(__dirname, 'dist', 'index.html'),
                htmlTemplate
              );
            });
          },
        },
      ],

      // Tree shaking
      treeShaking: true,

      // Define global constants
      define: {
        'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
        __VERSION__: JSON.stringify(require('./package.json').version),
      },

      // JSX configuration
      jsx: 'automatic',
      jsxFactory: 'React.createElement',
      jsxFragment: 'React.Fragment',

      // Banner and footer
      banner: {
        js: '/* Built with esbuild */',
        css: '/* Styles built with esbuild */',
      },

      // Metafile for analysis
      metafile: true,

      // Platform
      platform: 'browser',

      // Conditions
      conditions: ['import', 'module', 'browser'],

      // Main fields
      mainFields: ['browser', 'module', 'main'],

      // Write files
      write: true,

      // Legal comments
      legalComments: 'none',

      // Keep names for better debugging
      keepNames: !isProduction,
    });

    if (result.metafile) {
      const fs = require('fs');
      fs.writeFileSync('meta.json', JSON.stringify(result.metafile));
      
      // Generate bundle analysis
      const text = await esbuild.analyzeMetafile(result.metafile, {
        verbose: true,
      });
      console.log(text);
    }

    console.log('✅ Build completed successfully');
  } catch (error) {
    console.error('❌ Build failed:', error);
    process.exit(1);
  }
}

// Development server
async function serve() {
  const ctx = await esbuild.context({
    // ... same build options as above
  });

  const { host, port } = await ctx.serve({
    servedir: 'dist',
    host: 'localhost',
    port: 3000,
  });

  console.log(`🚀 Development server running at http://${host}:${port}`);

  // Watch for changes
  await ctx.watch();
}

if (process.argv.includes('--serve')) {
  serve();
} else {
  build();
}
```

## Rollup Advanced Configuration

### Rollup with Plugin Ecosystem

```javascript
// rollup.config.js
import { defineConfig } from 'rollup';
import typescript from '@rollup/plugin-typescript';
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import terser from '@rollup/plugin-terser';
import babel from '@rollup/plugin-babel';
import replace from '@rollup/plugin-replace';
import json from '@rollup/plugin-json';
import url from '@rollup/plugin-url';
import svg from 'rollup-plugin-svg';
import postcss from 'rollup-plugin-postcss';
import copy from 'rollup-plugin-copy';
import { visualizer } from 'rollup-plugin-visualizer';
import { generateSW } from 'rollup-plugin-workbox';
import serve from 'rollup-plugin-serve';
import livereload from 'rollup-plugin-livereload';

const isProduction = process.env.NODE_ENV === 'production';
const isDevelopment = !isProduction;

// Custom plugin for feature flags
function featureFlags(flags) {
  return {
    name: 'feature-flags',
    transform(code, id) {
      if (id.includes('node_modules')) return null;

      let transformedCode = code;
      Object.entries(flags).forEach(([flag, enabled]) => {
        const regex = new RegExp(`__FEATURE_${flag.toUpperCase()}__`, 'g');
        transformedCode = transformedCode.replace(regex, String(enabled));
      });

      return transformedCode;
    },
  };
}

// Custom plugin for environment-specific imports
function conditionalImports() {
  return {
    name: 'conditional-imports',
    resolveId(id, importer) {
      if (id.endsWith('.dev.js') && isProduction) {
        return id.replace('.dev.js', '.prod.js');
      }
      if (id.endsWith('.prod.js') && isDevelopment) {
        return id.replace('.prod.js', '.dev.js');
      }
      return null;
    },
  };
}

export default defineConfig([
  // Main application bundle
  {
    input: {
      main: 'src/index.tsx',
      admin: 'src/admin.tsx',
    },
    
    output: [
      // ES modules
      {
        dir: 'dist/esm',
        format: 'es',
        entryFileNames: '[name]-[hash].js',
        chunkFileNames: 'chunks/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash][extname]',
        sourcemap: true,
        generatedCode: 'es2015',
      },
      
      // CommonJS (for Node.js compatibility)
      ...(isProduction ? [{
        dir: 'dist/cjs',
        format: 'cjs',
        entryFileNames: '[name].js',
        chunkFileNames: 'chunks/[name].js',
        assetFileNames: 'assets/[name][extname]',
        exports: 'auto',
        interop: 'auto',
      }] : []),
    ],

    plugins: [
      // TypeScript compilation
      typescript({
        tsconfig: './tsconfig.json',
        declaration: true,
        declarationDir: 'dist/types',
        exclude: ['**/*.test.*', '**/*.spec.*'],
      }),

      // Node modules resolution
      resolve({
        browser: true,
        preferBuiltins: false,
        extensions: ['.tsx', '.ts', '.jsx', '.js', '.json'],
      }),

      // CommonJS to ES modules
      commonjs({
        include: /node_modules/,
        transformMixedEsModules: true,
      }),

      // Babel transformation
      babel({
        babelHelpers: 'bundled',
        exclude: 'node_modules/**',
        extensions: ['.tsx', '.ts', '.jsx', '.js'],
        presets: [
          ['@babel/preset-env', {
            targets: { browsers: ['last 2 versions'] },
            modules: false,
          }],
          '@babel/preset-react',
          '@babel/preset-typescript',
        ],
        plugins: [
          '@babel/plugin-proposal-class-properties',
          ['babel-plugin-styled-components', { displayName: !isProduction }],
        ],
      }),

      // Environment variables and feature flags
      replace({
        'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
        __VERSION__: JSON.stringify(process.env.npm_package_version),
        __BUILD_TIME__: JSON.stringify(new Date().toISOString()),
        preventAssignment: true,
      }),

      featureFlags({
        NEW_UI: true,
        EXPERIMENTAL: false,
      }),

      conditionalImports(),

      // JSON imports
      json(),

      // Asset handling
      url({
        limit: 8192, // 8kb
        include: ['**/*.png', '**/*.jpg', '**/*.jpeg', '**/*.gif'],
        fileName: 'assets/[name]-[hash][extname]',
      }),

      svg({
        base64: false,
      }),

      // CSS processing
      postcss({
        extract: true,
        minimize: isProduction,
        sourceMap: true,
        plugins: [
          require('autoprefixer'),
          ...(isProduction ? [require('cssnano')] : []),
        ],
        modules: {
          scopeBehaviour: 'local',
          generateScopedName: isProduction
            ? '[hash:base64:8]'
            : '[name]__[local]___[hash:base64:5]',
        },
      }),

      // File copying
      copy({
        targets: [
          { src: 'public/*', dest: 'dist' },
          { src: 'src/assets/images/*', dest: 'dist/assets/images' },
        ],
      }),

      // Development server
      ...(isDevelopment ? [
        serve({
          contentBase: ['dist'],
          host: 'localhost',
          port: 3000,
          headers: {
            'Access-Control-Allow-Origin': '*',
          },
        }),
        livereload('dist'),
      ] : []),

      // Production optimizations
      ...(isProduction ? [
        terser({
          compress: {
            drop_console: true,
            drop_debugger: true,
            pure_funcs: ['console.log', 'console.info'],
          },
          mangle: {
            safari10: true,
          },
          format: {
            comments: false,
          },
        }),

        // Service worker generation
        generateSW({
          swDest: 'dist/sw.js',
          globDirectory: 'dist',
          globPatterns: ['**/*.{html,js,css,png,jpg,jpeg,svg}'],
          runtimeCaching: [
            {
              urlPattern: /^https:\/\/api\.example\.com\//,
              handler: 'NetworkFirst',
              options: {
                cacheName: 'api-cache',
                expiration: {
                  maxEntries: 100,
                  maxAgeSeconds: 60 * 60 * 24, // 1 day
                },
              },
            },
          ],
        }),

        // Bundle analysis
        ...(process.env.ANALYZE ? [
          visualizer({
            filename: 'dist/stats.html',
            open: true,
            gzipSize: true,
            brotliSize: true,
          }),
        ] : []),
      ] : []),
    ],

    external: ['react', 'react-dom'],

    // Optimization
    treeshake: {
      moduleSideEffects: false,
      propertyReadSideEffects: false,
      unknownGlobalSideEffects: false,
    },

    // Preserve modules for better tree shaking
    preserveModules: false,

    // Code splitting
    manualChunks: {
      vendor: ['react', 'react-dom'],
      utils: ['lodash', 'date-fns'],
    },
  },

  // Library build (for npm publishing)
  ...(isProduction ? [{
    input: 'src/lib/index.ts',
    output: [
      {
        file: 'dist/lib/index.esm.js',
        format: 'es',
        sourcemap: true,
      },
      {
        file: 'dist/lib/index.cjs.js',
        format: 'cjs',
        sourcemap: true,
        exports: 'auto',
      },
      {
        file: 'dist/lib/index.umd.js',
        format: 'umd',
        name: 'MyLibrary',
        sourcemap: true,
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM',
        },
      },
    ],
    plugins: [
      typescript({
        tsconfig: './tsconfig.lib.json',
        declaration: true,
        declarationDir: 'dist/lib/types',
      }),
      resolve(),
      commonjs(),
      terser(),
    ],
    external: ['react', 'react-dom'],
  }] : []),
]);
```

## Build Performance Optimization

### Benchmarking and Monitoring

```typescript
// scripts/build-performance.ts
import { performance } from 'perf_hooks';
import { spawn } from 'child_process';
import fs from 'fs/promises';
import path from 'path';

interface BuildMetrics {
  tool: string;
  duration: number;
  bundleSize: number;
  chunkCount: number;
  timestamp: Date;
  environment: string;
}

class BuildPerformanceTracker {
  private metrics: BuildMetrics[] = [];
  private metricsFile = path.join(process.cwd(), 'build-metrics.json');

  async trackBuild(tool: string, buildCommand: string): Promise<BuildMetrics> {
    console.log(`🔨 Starting ${tool} build...`);
    
    const startTime = performance.now();
    
    try {
      await this.runCommand(buildCommand);
      const endTime = performance.now();
      
      const metrics: BuildMetrics = {
        tool,
        duration: endTime - startTime,
        bundleSize: await this.calculateBundleSize(),
        chunkCount: await this.countChunks(),
        timestamp: new Date(),
        environment: process.env.NODE_ENV || 'development',
      };

      this.metrics.push(metrics);
      await this.saveMetrics();
      
      console.log(`✅ ${tool} build completed in ${(metrics.duration / 1000).toFixed(2)}s`);
      console.log(`📦 Bundle size: ${(metrics.bundleSize / 1024 / 1024).toFixed(2)} MB`);
      console.log(`🧩 Chunks: ${metrics.chunkCount}`);
      
      return metrics;
    } catch (error) {
      console.error(`❌ ${tool} build failed:`, error);
      throw error;
    }
  }

  private runCommand(command: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const [cmd, ...args] = command.split(' ');
      const process = spawn(cmd, args, { stdio: 'pipe' });
      
      process.on('close', (code) => {
        if (code === 0) {
          resolve();
        } else {
          reject(new Error(`Command failed with code ${code}`));
        }
      });
    });
  }

  private async calculateBundleSize(): Promise<number> {
    const distDir = path.join(process.cwd(), 'dist');
    let totalSize = 0;

    try {
      const files = await fs.readdir(distDir, { recursive: true });
      
      for (const file of files) {
        const filePath = path.join(distDir, file as string);
        const stats = await fs.stat(filePath);
        
        if (stats.isFile()) {
          totalSize += stats.size;
        }
      }
    } catch (error) {
      console.warn('Could not calculate bundle size:', error);
    }

    return totalSize;
  }

  private async countChunks(): Promise<number> {
    const distDir = path.join(process.cwd(), 'dist');
    
    try {
      const files = await fs.readdir(distDir, { recursive: true });
      return files.filter(file => 
        typeof file === 'string' && file.endsWith('.js')
      ).length;
    } catch (error) {
      console.warn('Could not count chunks:', error);
      return 0;
    }
  }

  private async saveMetrics(): Promise<void> {
    try {
      await fs.writeFile(
        this.metricsFile,
        JSON.stringify(this.metrics, null, 2)
      );
    } catch (error) {
      console.warn('Could not save metrics:', error);
    }
  }

  async loadMetrics(): Promise<BuildMetrics[]> {
    try {
      const data = await fs.readFile(this.metricsFile, 'utf-8');
      this.metrics = JSON.parse(data);
      return this.metrics;
    } catch (error) {
      return [];
    }
  }

  generateReport(): void {
    if (this.metrics.length === 0) {
      console.log('No metrics available');
      return;
    }

    console.log('\n📊 Build Performance Report\n');
    
    const groupedMetrics = this.metrics.reduce((acc, metric) => {
      if (!acc[metric.tool]) {
        acc[metric.tool] = [];
      }
      acc[metric.tool].push(metric);
      return acc;
    }, {} as Record<string, BuildMetrics[]>);

    for (const [tool, metrics] of Object.entries(groupedMetrics)) {
      const avgDuration = metrics.reduce((sum, m) => sum + m.duration, 0) / metrics.length;
      const avgBundleSize = metrics.reduce((sum, m) => sum + m.bundleSize, 0) / metrics.length;
      const latestMetric = metrics[metrics.length - 1];

      console.log(`${tool}:`);
      console.log(`  Average Duration: ${(avgDuration / 1000).toFixed(2)}s`);
      console.log(`  Average Bundle Size: ${(avgBundleSize / 1024 / 1024).toFixed(2)} MB`);
      console.log(`  Latest Chunks: ${latestMetric.chunkCount}`);
      console.log('');
    }
  }
}

// Usage example
async function compareBuildTools() {
  const tracker = new BuildPerformanceTracker();
  await tracker.loadMetrics();

  const tools = [
    { name: 'Vite', command: 'npm run build:vite' },
    { name: 'Webpack', command: 'npm run build:webpack' },
    { name: 'esbuild', command: 'npm run build:esbuild' },
    { name: 'Rollup', command: 'npm run build:rollup' },
  ];

  for (const tool of tools) {
    try {
      await tracker.trackBuild(tool.name, tool.command);
      // Clean dist folder between builds
      await fs.rm(path.join(process.cwd(), 'dist'), { recursive: true, force: true });
    } catch (error) {
      console.error(`Failed to track ${tool.name} build:`, error);
    }
  }

  tracker.generateReport();
}

// Cache optimization utility
class BuildCacheManager {
  private cacheDir = path.join(process.cwd(), '.build-cache');

  async initializeCache(): Promise<void> {
    try {
      await fs.mkdir(this.cacheDir, { recursive: true });
    } catch (error) {
      console.warn('Could not initialize cache directory:', error);
    }
  }

  async getCacheKey(files: string[]): Promise<string> {
    const crypto = await import('crypto');
    const hash = crypto.createHash('sha256');

    for (const file of files) {
      try {
        const content = await fs.readFile(file);
        hash.update(content);
      } catch (error) {
        console.warn(`Could not read file for cache key: ${file}`);
      }
    }

    return hash.digest('hex');
  }

  async isCacheValid(cacheKey: string): Promise<boolean> {
    const cacheFile = path.join(this.cacheDir, `${cacheKey}.json`);
    
    try {
      await fs.access(cacheFile);
      return true;
    } catch {
      return false;
    }
  }

  async saveToCache(cacheKey: string, data: any): Promise<void> {
    const cacheFile = path.join(this.cacheDir, `${cacheKey}.json`);
    
    try {
      await fs.writeFile(cacheFile, JSON.stringify(data));
    } catch (error) {
      console.warn('Could not save to cache:', error);
    }
  }

  async loadFromCache(cacheKey: string): Promise<any> {
    const cacheFile = path.join(this.cacheDir, `${cacheKey}.json`);
    
    try {
      const data = await fs.readFile(cacheFile, 'utf-8');
      return JSON.parse(data);
    } catch (error) {
      console.warn('Could not load from cache:', error);
      return null;
    }
  }

  async clearCache(): Promise<void> {
    try {
      await fs.rm(this.cacheDir, { recursive: true, force: true });
      await this.initializeCache();
    } catch (error) {
      console.warn('Could not clear cache:', error);
    }
  }
}

if (require.main === module) {
  compareBuildTools().catch(console.error);
}

export { BuildPerformanceTracker, BuildCacheManager };
```

## Interview-Ready Build Tools Summary

**Modern Build Tool Ecosystem:**
1. **Vite** - ESM-native development with instant HMR and optimized production builds
2. **Webpack** - Mature, configurable bundler with extensive plugin ecosystem
3. **esbuild** - Ultra-fast JavaScript bundler and minifier written in Go
4. **Rollup** - Tree-shaking focused bundler optimized for libraries

**Advanced Configuration Patterns:**
- Multi-entry point builds with code splitting strategies
- Environment-specific optimizations and feature flags
- Custom loaders and plugins for specialized asset handling
- Performance monitoring and build cache management

**Enterprise Optimization:**
- Bundle analysis and size optimization techniques
- Progressive web app integration with service workers
- Legacy browser support with polyfill strategies
- Development experience enhancements with HMR and debugging tools

**Key Interview Topics:** Build tool architecture differences, webpack vs Vite trade-offs, esbuild performance benefits, tree shaking optimization, code splitting strategies, plugin development, performance measurement, and modern development workflow optimization.