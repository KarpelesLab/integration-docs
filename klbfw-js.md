# KarpelesLab Frontend Framework (klbfw)

A JavaScript library for communicating with KarpelesLab API services from both browser and Node.js environments.

[![npm version](https://img.shields.io/npm/v/@karpeleslab/klbfw.svg)](https://www.npmjs.com/package/@karpeleslab/klbfw)
[![license](https://img.shields.io/npm/l/@karpeleslab/klbfw.svg)](https://github.com/KarpelesLab/klbfw/blob/master/LICENSE)

## Table of Contents

- [Installation](#installation)
- [Overview](#overview)
- [Features](#features)
- [Environment Compatibility](#environment-compatibility)
- [API Reference](#api-reference)
  - [Framework Wrapper](#framework-wrapper)
  - [REST API Client](#rest-api-client)
  - [File Upload](#file-upload)
  - [Cookie Management](#cookie-management)
  - [Utility Functions](#utility-functions)
- [Usage Examples](#usage-examples)
  - [Browser Usage](#browser-usage)
  - [Node.js Usage](#nodejs-usage)
- [Advanced Usage](#advanced-usage)
  - [Upload Management](#upload-management)
  - [Context Manipulation](#context-manipulation)
  - [Internationalization](#internationalization)

## Installation

```bash
# Basic installation
npm install @karpeleslab/klbfw

# With optional dependencies for Node.js with file upload support
npm install @karpeleslab/klbfw node-fetch xmldom
```

## Overview

The KarpelesLab Frontend Framework (klbfw) provides a unified interface for interacting with KarpelesLab API services. It is designed to work consistently across different JavaScript environments including browsers, Node.js, and server-side rendering.

The library abstracts away environment differences and provides a consistent API for accessing KarpelesLab services, handling authentication, making REST API calls, uploading files, and managing context and state.

## Features

- **Cross-environment compatibility**: Works seamlessly in browser, Node.js, and SSR contexts
- **REST API client**: Simple and consistent interface for API requests
- **File upload**: Robust file uploads with both direct PUT and AWS S3 multipart protocols
- **Context handling**: Manages authentication, locale, and other contextual information
- **Cookie management**: Cross-platform cookie handling that works in any environment
- **Internationalization**: Easy access to i18n data

## Environment Compatibility

klbfw automatically detects the current environment and uses appropriate implementations:

| Feature | Browser | Node.js | SSR |
|---------|---------|---------|-----|
| REST API | ✓ | ✓ (with node-fetch) | ✓ |
| File Upload | ✓ | ✓ (with node-fetch, fs) | ✓ |
| Cookie Management | ✓ | ✓ | ✓ |
| Context API | ✓ | ✓ | ✓ |
| URL Management | ✓ | ✓ | ✓ |

## API Reference

### Framework Wrapper

These functions provide a safe wrapper around the global FW object with appropriate fallbacks.

```typescript
// Query parameters
GET(): Record<string, string>
Get(key?: string): string | Record<string, string> | undefined
flushGet(): void

// URL and path
getPrefix(): string
getUrl(): { path: string; full: string; host: string; query: string; scheme: string }
getPath(): string
getHostname(): string
trimPrefix(path: string): string

// Context and state
getSettings(): Record<string, any>
getRealm(): Record<string, any>
getContext(): Record<string, any>
setContext(key: string, value: any): void
getInitialState(): Record<string, any> | undefined
getMode(): string
getRegistry(): Record<string, any> | undefined
getLocale(): string
getUserGroup(): string | undefined
getCurrency(): string
getToken(): string | undefined
getUuid(): string | undefined
```

### REST API Client

Functions for making API calls to KarpelesLab services.

```typescript
rest(name: string, verb: string, params?: Record<string, any> | string, context?: Record<string, any>): Promise<any>
rest_get(name: string, params?: Record<string, any> | string): Promise<any> // Backward compatibility
restGet(name: string, params?: Record<string, any> | string): Promise<any>
```

#### Parameters

- `name`: The REST endpoint to call (e.g., 'User/Session:get')
- `verb`: HTTP method (GET, POST, PUT, DELETE)
- `params`: Request parameters (object or JSON string)
- `context`: Additional context information

#### Return Value

Returns a Promise that resolves with the API response or rejects with an error object:

```typescript
interface ErrorObject {
  message: string;
  body?: any;
  headers?: Record<string, string>;
  status?: number;
}
```

### File Upload

A comprehensive module for handling file uploads across different environments.

```typescript
interface UploadOptions {
  progress?: (progress: number) => void;
  endpoint?: string;
  headers?: Record<string, string>;
  retry?: number;
  chunk_size?: number;
  params?: Record<string, any>;
}

// Upload module
upload.init(path: string, params?: Record<string, any>, notify?: Function): Function
upload.append(path: string, file: File | { name: string; size: number; type: string; path: string }, params?: Record<string, any>, context?: Record<string, any>): Promise<any>
upload.getStatus(): { queue: Array<any>; running: Array<any>; failed: Array<any> }
upload.run(): void
upload.cancelItem(uploadId: string): void
upload.deleteItem(uploadId: string): void
upload.pauseItem(uploadId: string): void
upload.resumeItem(uploadId: string): void
upload.retryItem(uploadId: string): void
upload.resume(): void
```

### Cookie Management

Cross-platform cookie handling that works in any environment.

```typescript
getCookie(name: string): string | null
hasCookie(name: string): boolean
setCookie(name: string, value: string, expires?: Date | number, path?: string, domain?: string, secure?: boolean): void
```

### Utility Functions

Additional utility functions to assist with KarpelesLab API integration.

```typescript
getI18N(language?: string): Promise<Record<string, any>>
trimPrefix(path: string): string
```

## Usage Examples

### Browser Usage

```javascript
// Import the library
import { rest, restGet, getCookie, setCookie, getLocale } from '@karpeleslab/klbfw';

// Make REST API calls
restGet('User/Session:get')
  .then(session => {
    console.log('User session:', session);
  })
  .catch(error => {
    console.error('Error fetching session:', error.message);
  });

// Post data to an API
rest('User/Auth:login', 'POST', { email: 'user@example.com', password: 'securepassword' })
  .then(response => {
    console.log('Login successful:', response);
  })
  .catch(error => {
    console.error('Login failed:', error.message);
  });

// Use cookie management
if (!hasCookie('visited')) {
  setCookie('visited', 'true', new Date(Date.now() + 86400000)); // Expires in 1 day
}

// Get current locale
const locale = getLocale();
console.log('Current locale:', locale);
```

### Node.js Usage

```javascript
// Import the library
const { rest, upload, getI18N } = require('@karpeleslab/klbfw');

// Make an authenticated API call
rest('Misc/Debug:test', 'GET', { test: 'value' }, { token: 'auth-token' })
  .then(response => {
    console.log('API response:', response);
  })
  .catch(error => {
    console.error('API error:', error);
  });

// Upload a file
const file = {
  name: 'test.txt',
  size: 1024,
  type: 'text/plain',
  path: '/path/to/file.txt'
};

upload.append('Storage/File:upload', file, { public: true })
  .then(result => {
    console.log('Upload successful:', result);
  })
  .catch(error => {
    console.error('Upload failed:', error.message);
  });

// Get internationalization data
getI18N('en-US')
  .then(i18nData => {
    console.log('Loaded translations for en-US');
  });
```

## Advanced Usage

### Upload Management

The upload module provides comprehensive file upload capabilities with progress tracking, chunked uploads, and queue management.

```javascript
// Browser: Open file picker
upload.init('Storage/File:upload')()
  .then(result => console.log('Upload complete', result));

// Track upload progress
upload.onprogress = (status) => {
  console.log('Files in queue:', status.queue.length);
  console.log('Files uploading:', status.running.map(item => ({
    id: item.id,
    filename: item.file.name,
    progress: Math.round(item.status * 100) + '%'
  })));
  console.log('Failed uploads:', status.failed.length);
};

// Manage uploads
const status = upload.getStatus();
const failedUploads = status.failed;

if (failedUploads.length > 0) {
  // Retry all failed uploads
  upload.resume();
  
  // Or retry a specific upload
  upload.retryItem(failedUploads[0].id);
}

// Cancel an active upload
if (status.running.length > 0) {
  upload.cancelItem(status.running[0].id);
}
```

### Context Manipulation

The context API allows you to manage state and context information.

```javascript
// Get current context
const context = getContext();
console.log('Current user group:', getUserGroup());

// Access URL components
const url = getUrl();
console.log('Current path:', url.path);
console.log('Full URL:', url.full);
console.log('Host:', url.host);
console.log('Query string:', url.query);
console.log('Scheme:', url.scheme);

// Work with path prefixes
const prefix = getPrefix(); // e.g., "/l/en-US"
const path = getPath();     // path without prefix
```

### Internationalization

Working with translations and locale-specific data. This is usually best left to a specific module like `@karpeleslab/i18next-klb-backend`.

```javascript
// Get translations for current locale
getI18N().then(translations => {
  console.log('Available translations:', Object.keys(translations));
  
  // Use translations
  const welcomeMessage = translations['welcome_message'] || 'Welcome';
  console.log(welcomeMessage);
});

// Get translations for a specific locale
getI18N('ja-JP').then(translations => {
  console.log('Japanese welcome message:', translations['welcome_message']);
});

// Current locale and currency
console.log('Active locale:', getLocale());
console.log('Active currency:', getCurrency());
```
