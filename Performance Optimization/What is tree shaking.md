# What is tree shaking

This file explains tree shaking - a dead code elimination technique that reduces bundle size by removing unused code.
Essential for understanding modern build optimization and bundle size management.

---

## What it is

Tree shaking is a static analysis technique that eliminates dead code (unused exports) from your JavaScript bundles. It works by:

* Analyzing your import/export statements
* Building a dependency graph of actually used code
* Removing unused exports from the final bundle
* Reducing bundle size and improving load performance

The name comes from shaking a tree to make dead leaves fall off, leaving only the living branches.

---

## How it works

### ES6 modules requirement

Tree shaking requires ES6 module syntax (`import`/`export`) because:

```js
// ✅ GOOD: Static imports - bundler can analyze at build time
import { useState, useEffect } from 'react';
import { debounce } from 'lodash-es';

// ❌ BAD: Dynamic requires - can't be statically analyzed
const React = require('react');
const _ = require('lodash'); // Imports entire library
```

### Build-time analysis

```js
// utils.js - Library with multiple exports
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
export function multiply(a, b) { return a * b; }
export function divide(a, b) { return a / b; }

// main.js - Only imports what's needed
import { add, multiply } from './utils.js';

console.log(add(2, 3));
console.log(multiply(4, 5));

// Result: subtract and divide functions are eliminated from final bundle
```

---

## Webpack configuration

### Basic setup

```js
// webpack.config.js
module.exports = {
  mode: 'production', // Enables tree shaking automatically
  optimization: {
    usedExports: true, // Mark unused exports
    sideEffects: false, // Safe to remove unused code
  },
};
```

### Package.json sideEffects

```json
{
  "name": "my-library",
  "sideEffects": false,
  "main": "dist/index.js",
  "module": "dist/index.esm.js"
}
```

```json
{
  "name": "my-app",
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
}
```

### Advanced configuration

```js
// webpack.config.js
module.exports = {
  optimization: {
    usedExports: true,
    sideEffects: false,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            dead_code: true,
            drop_debugger: true,
            drop_console: true,
          },
        },
      }),
    ],
  },
};
```

---

## Library tree shaking

### Lodash example

```js
// ❌ BAD: Imports entire lodash library (~70KB)
import _ from 'lodash';
const result = _.debounce(fn, 300);

// ✅ GOOD: Tree-shakeable version
import { debounce } from 'lodash-es';
const result = debounce(fn, 300);

// ✅ BETTER: Individual imports
import debounce from 'lodash/debounce';
const result = debounce(fn, 300);
```

### Date libraries

```js
// ❌ BAD: Moment.js (not tree-shakeable, ~67KB)
import moment from 'moment';
const formatted = moment().format('YYYY-MM-DD');

// ✅ GOOD: date-fns (tree-shakeable)
import { format } from 'date-fns';
const formatted = format(new Date(), 'yyyy-MM-dd');

// ✅ GOOD: Day.js (smaller alternative)
import dayjs from 'dayjs';
const formatted = dayjs().format('YYYY-MM-DD');
```

### UI libraries

```js
// ❌ BAD: Imports entire Material-UI
import { Button, TextField, Dialog } from '@material-ui/core';

// ✅ GOOD: Babel plugin for automatic optimization
// .babelrc
{
  "plugins": [
    ["import", {
      "libraryName": "@material-ui/core",
      "libraryDirectory": "",
      "camel2DashComponentName": false
    }, "core"]
  ]
}

// ✅ GOOD: Direct imports
import Button from '@material-ui/core/Button';
import TextField from '@material-ui/core/TextField';
```

---

## Writing tree-shakeable code

### Export individual functions

```js
// ❌ BAD: Default export of object
const utils = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
  multiply: (a, b) => a * b,
};
export default utils;

// ✅ GOOD: Named exports
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export const multiply = (a, b) => a * b;
```

### Avoid side effects

```js
// ❌ BAD: Side effect in module
console.log('Module loaded'); // Prevents tree shaking
export const add = (a, b) => a + b;

// ✅ GOOD: Pure module
export const add = (a, b) => a + b;

// ✅ GOOD: Conditional side effect
export const add = (a, b) => {
  if (process.env.NODE_ENV === 'development') {
    console.log('Adding:', a, b);
  }
  return a + b;
};
```

### Proper barrel exports

```js
// ❌ BAD: Re-exports everything
export * from './utils';
export * from './helpers';
export * from './constants';

// ✅ GOOD: Selective re-exports
export { add, subtract } from './math';
export { debounce, throttle } from './timing';
export { API_URL, DEFAULT_TIMEOUT } from './constants';
```

---

## Real-world examples

### React application

```js
// Before tree shaking analysis
import React from 'react'; // ~40KB
import { BrowserRouter, Route, Link } from 'react-router-dom'; // ~20KB
import { Provider } from 'react-redux'; // ~15KB
import _ from 'lodash'; // ~70KB
// Total: ~145KB

// After optimization
import React from 'react'; // ~40KB (used)
import { BrowserRouter, Route, Link } from 'react-router-dom'; // ~15KB (tree-shaken)
import { Provider } from 'react-redux'; // ~8KB (tree-shaken)
import { debounce, throttle } from 'lodash-es'; // ~2KB (only needed functions)
// Total: ~65KB (55% reduction)
```

### Custom library structure

```js
// src/index.js - Library entry point
export { Button } from './components/Button';
export { Input } from './components/Input';
export { Modal } from './components/Modal';
export { Table } from './components/Table';

// src/components/Button/index.js
export { Button } from './Button';

// src/components/Button/Button.js
import React from 'react';
import './Button.css'; // Side effect - needs to be marked

export const Button = ({ children, onClick, variant = 'primary' }) => (
  <button className={`btn btn-${variant}`} onClick={onClick}>
    {children}
  </button>
);
```

---

## Analyzing tree shaking effectiveness

### Webpack Bundle Analyzer

```bash
npm install --save-dev webpack-bundle-analyzer
```

```js
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    })
  ]
};
```

### Using webpack-cli

```bash
# Generate stats file
npx webpack --profile --json > stats.json

# Analyze unused exports
npx webpack-unused -s stats.json

# Check what's actually used
npx webpack --display-used-exports --display-optimization-bailout
```

### Rollup analyzer

```js
// rollup.config.js
import { visualizer } from 'rollup-plugin-visualizer';

export default {
  plugins: [
    visualizer({
      filename: 'bundle-analysis.html',
      open: true
    })
  ]
};
```

---

## Common pitfalls and solutions

### 1. Side effects blocking tree shaking

```js
// ❌ BAD: Implicit side effect
import './polyfills'; // Runs polyfill code
export const myFunction = () => {};

// ✅ GOOD: Mark as side effect in package.json
{
  "sideEffects": ["./src/polyfills.js"]
}
```

### 2. CommonJS mixed with ES6

```js
// ❌ BAD: Mixed module systems
const React = require('react'); // CommonJS
import { Component } from 'react'; // ES6

// ✅ GOOD: Consistent ES6 modules
import React, { Component } from 'react';
```

### 3. Dynamic imports breaking analysis

```js
// ❌ BAD: Dynamic property access
import * as utils from './utils';
const methodName = getMethodName();
utils[methodName](); // Can't be statically analyzed

// ✅ GOOD: Static imports
import { add, subtract } from './utils';
const result = condition ? add(a, b) : subtract(a, b);
```

---

## Modern tools and tree shaking

### Vite

```js
// vite.config.js - Tree shaking enabled by default
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash-es', 'date-fns']
        }
      }
    }
  }
};
```

### esbuild

```js
// esbuild automatically performs tree shaking
require('esbuild').build({
  entryPoints: ['src/index.js'],
  bundle: true,
  treeShaking: true, // Default in production
  minify: true,
  outfile: 'dist/bundle.js',
});
```

### Parcel

```json
{
  "name": "my-app",
  "source": "src/index.html",
  "sideEffects": false,
  "browserslist": "> 0.5%, last 2 versions, not dead"
}
```

---

## Best practices checklist

* [ ] Use ES6 modules (`import`/`export`) consistently
* [ ] Mark packages as side-effect-free in `package.json`
* [ ] Prefer named exports over default exports for libraries
* [ ] Use tree-shakeable versions of libraries (lodash-es vs lodash)
* [ ] Avoid importing entire libraries when only using few functions
* [ ] Configure webpack/bundler for production mode
* [ ] Analyze bundle composition with webpack-bundle-analyzer
* [ ] Test tree shaking effectiveness with build size comparisons
* [ ] Document side effects clearly in your packages
* [ ] Use Babel plugins for automatic import optimization

---

## Interview-ready summary

Tree shaking eliminates dead code from JavaScript bundles by analyzing ES6 import/export statements and removing unused exports. It requires static imports, proper `sideEffects` configuration, and production build settings. Key benefits include smaller bundle sizes and faster load times. Common pitfalls include mixing CommonJS with ES6 modules, undeclared side effects, and importing entire libraries instead of specific functions. Modern bundlers like Webpack, Rollup, and Vite support tree shaking automatically in production mode.
