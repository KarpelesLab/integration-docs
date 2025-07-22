# Version Service Worker Installation Guide

This guide explains how to set up a service worker that adds version headers to asset requests, enabling better cache management and version tracking in your Vue.js application.

## Overview

The version service worker system consists of:
1. A static service worker that intercepts fetch requests
2. A version injection plugin that gets the Git commit hash during build
3. HTML modifications to register the service worker and pass version information
4. Build configuration to enable the version injection

## Installation Steps

### Step 1: Create the Service Worker

Create a file `public/service-worker.js` with the following content:

```javascript
// Static service worker that adds version headers to asset requests
// This file remains the same across all deployments

let currentVersion = null;

// Listen for a message from the main thread to set the version
self.addEventListener('message', event => {
  if (event.data && event.data.type === 'SET_VERSION') {
    currentVersion = event.data.version;
  } else if (event.data && event.data.type === 'RESET_VERSION') {
    currentVersion = null;
  }
});

// Intercept fetch requests
self.addEventListener('fetch', event => {
  // Skip modification if we don't have a version
  if (!currentVersion) {
    return;
  }
  
  // Skip API requests (paths starting with /_)
  const url = new URL(event.request.url);
  if (url.pathname.startsWith('/_')) {
    return;
  }
  
  // Skip document requests (HTML pages)
  if (event.request.destination === 'document') {
    return;
  }
  
  // Only handle same-origin requests
  if (url.origin !== self.location.origin) {
    return;
  }
  
  // Clone the request to modify it
  const modifiedRequest = new Request(event.request.url, {
    method: event.request.method,
    headers: new Headers(event.request.headers),
    mode: 'cors',
    credentials: event.request.credentials,
    redirect: event.request.redirect
  });
  
  // Add our version header
  modifiedRequest.headers.set('X-Version-Hint', currentVersion);
  
  // Use the modified request
  event.respondWith(
    fetch(modifiedRequest)
      .catch(error => {
        console.error('Service worker fetch error:', error);
        // Fall back to the original request if our modification fails
        return fetch(event.request);
      })
  );
});

// Cache management
self.addEventListener('install', event => {
  self.skipWaiting();
});

self.addEventListener('activate', event => {
  event.waitUntil(self.clients.claim());
});
```

### Step 2: Create the Version Injector Plugin

Create a file `src/plugins/version-injector.js`:

```javascript
// src/plugins/version-injector.js
const childProcess = require('child_process');

/**
 * Webpack plugin to inject app version from Git commit hash
 */
class VersionInjectorPlugin {
  constructor(options = {}) {
    this.options = {
      // Default options
      placeholder: '%GIT_VERSION%',
      ...options
    };
  }

  apply(compiler) {
    compiler.hooks.compilation.tap('VersionInjectorPlugin', (compilation) => {
      // Get git commit hash
      const version = this.getGitCommitHash();
      
      // Hook into HTML webpack plugin
      try {
        // For html-webpack-plugin v4 and above
        const HtmlWebpackPlugin = require('html-webpack-plugin');
        const hooks = HtmlWebpackPlugin.getHooks(compilation);
        
        hooks.beforeEmit.tapAsync('VersionInjectorPlugin', (data, callback) => {
          data.html = data.html.replace(new RegExp(this.options.placeholder, 'g'), version);
          callback(null, data);
        });
      } catch (error) {
        console.warn('Could not find HtmlWebpackPlugin hooks - this is ok if you\'re not using HtmlWebpackPlugin');
      }
    });
  }

  getGitCommitHash() {
    try {
      return childProcess.execSync('git rev-parse --short=7 HEAD')
        .toString()
        .trim();
    } catch (error) {
      console.warn('Unable to get git commit hash', error);
      return 'unknown';
    }
  }
}

module.exports = VersionInjectorPlugin;
```

### Step 3: Update Vue Configuration

Update your `vue.config.js` file:

```javascript
const { defineConfig } = require('@vue/cli-service')
const VersionInjectorPlugin = require('./src/plugins/version-injector')

module.exports = defineConfig({
  transpileDependencies: true,
  configureWebpack: {
    plugins: [
      new VersionInjectorPlugin()
    ]
  }
})
```

### Step 4: Modify index.html

Update your `public/index.html` file to include the version meta tag and service worker registration. Add the following within the `<head>` section:

```html
<meta name="app-version" content="%GIT_VERSION%">
```

And add this script before the closing `</head>` tag:

```html
<script>
  // Register service worker with version information
  if ('serviceWorker' in navigator) {
    // Get version from meta tag
    const versionMeta = document.querySelector('meta[name="app-version"]');
    const version = versionMeta ? versionMeta.getAttribute('content') : '%GIT_VERSION%';
    console.log('Page loaded with version:', version);
    
    // Check for existing service worker controller first
    if (navigator.serviceWorker.controller) {
      console.log('Found existing service worker controller');
      // Send version to existing controller
      navigator.serviceWorker.controller.postMessage({
        type: 'SET_VERSION',
        version: version
      });
    }
    
    // Register the service worker
    navigator.serviceWorker.register('/service-worker.js')
      .then(registration => {
        console.log('Service worker registered');
        
        // If there's already an active service worker
        if (registration.active) {
          console.log('Active service worker found, sending version');
          registration.active.postMessage({
            type: 'SET_VERSION',
            version: version
          });
        }
        
        // Handle waiting service worker
        if (registration.waiting) {
          console.log('Waiting service worker found, sending version');
          registration.waiting.postMessage({
            type: 'SET_VERSION',
            version: version
          });
        }
        
        // Handle future installations
        registration.onupdatefound = () => {
          const installing = registration.installing;
          console.log('New service worker installing');
          
          if (installing) {
            installing.onstatechange = () => {
              console.log('Service worker state changed:', installing.state);
              
              if (installing.state === 'activated') {
                console.log('Service worker activated, sending version');
                installing.postMessage({
                  type: 'SET_VERSION',
                  version: version
                });
              }
            };
          }
        };
      })
      .catch(error => {
        console.error('Service worker registration failed:', error);
      });
      
    // Listen for controlled change events
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      console.log('Service worker controller changed');
      if (navigator.serviceWorker.controller) {
        console.log('Sending version to new controller');
        navigator.serviceWorker.controller.postMessage({
          type: 'SET_VERSION',
          version: version
        });
      }
    });
  }
</script>
```

## Building and Testing

1. Ensure you have a Git repository initialized in your project
2. Run the build command:
   ```bash
   yarn build
   ```
3. The build process will:
   - Extract the current Git commit hash (7 characters)
   - Replace all instances of `%GIT_VERSION%` in the HTML with the actual commit hash
   - Generate the production build in the `dist` directory

## How It Works

1. **During Build**: The `VersionInjectorPlugin` gets the Git commit hash and replaces `%GIT_VERSION%` placeholders in the HTML
2. **On Page Load**: The browser reads the version from the meta tag and registers the service worker
3. **Service Worker Communication**: The page sends the version to the service worker using `postMessage`
4. **Request Interception**: The service worker intercepts outgoing requests and adds an `X-Version-Hint` header with the current version
5. **Server-Side Usage**: Your server can use this header for cache management, logging, or version-specific routing

## Features

- **Static Service Worker**: The service worker file itself never changes, making it cache-friendly
- **Version Headers**: All asset requests include the `X-Version-Hint` header
- **API Request Exclusion**: Requests to paths starting with `/_` are not modified
- **Document Request Exclusion**: HTML page requests are not modified
- **Error Handling**: Falls back to original requests if modification fails
- **Cross-Origin Safety**: Only modifies same-origin requests

## Troubleshooting

- If the version shows as "unknown", ensure your project is a Git repository with at least one commit
- Check the browser console for service worker registration messages
- Use the Network tab in DevTools to verify the `X-Version-Hint` header is being added to asset requests
- The service worker will not modify requests until it receives a version via `postMessage`

## Browser Compatibility

This implementation requires browser support for:
- Service Workers
- `postMessage` API
- Modern JavaScript features (const, arrow functions, template literals)

Most modern browsers (Chrome, Firefox, Safari, Edge) support these features.