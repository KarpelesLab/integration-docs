# Karpeles Lab Inc Integration Documentation

This repository contains comprehensive documentation for integrating with Karpeles Lab Inc systems and APIs.

## Contents

- [API Basics](apibasics.md) - Documentation for working with the PHP Platform REST API
- [Access Levels](access_levels.md) - Guide for understanding access levels, permissions, and user groups
- [OAuth2 Authentication](oauth2_auth.md) - Guide for OAuth2 token authentication and API key signing
- [User Flow](userflow.md) - Guide for implementing user authentication and registration flows
- [KLB API Describe Tool](klbfw-describe.md) - Manual for using the klbfw-describe command-line tool for API exploration
- [KLB Framework JavaScript](klbfw-js.md) - Documentation for the JavaScript framework integration
- [KLB Framework i18n](klbfw-i18next.md) - Guide for implementing internationalization with i18next
- [KLB Framework Flutter](klbfw-flutter.md) - Documentation for the Flutter package integration
- [Stripe Express Checkout](express_checkout.md) - Guide for implementing Stripe Express Checkout (Apple Pay, Google Pay)
- [Media Variations API](media_variation.md) - Documentation for transforming media files through API endpoints
- [Version Service Worker](install-version-service-worker.md) - Guide for setting up a service worker for version tracking in Vue.js apps
- [Upload Protocol](upload_lowlevel.md) - Low-level upload protocol for client implementations without a native SDK

## Overview

This documentation provides detailed information for developers integrating with Karpeles Lab Inc systems. The guides cover both API endpoints and user interface integration requirements.

## API Basics

The [API Basics](apibasics.md) guide covers:

- Authentication methods
- Request and response formats
- HTTP status codes
- Error handling
- Pagination
- Resource operations
- Query filters
- Rate limiting
- Example requests

## Access Levels and Permissions

The [Access Levels](access_levels.md) guide covers:

- Access level hierarchy (O, A, D, W, C, R, N)
- User groups and group membership
- How access checks work
- Understanding the `access` property in API responses
- Public resources and link-based sharing
- Parent-child access inheritance

## OAuth2 Authentication

The [OAuth2 Authentication](oauth2_auth.md) guide covers:

- OAuth2 bearer token authentication for user sessions
- Token refresh flow and automatic renewal
- API key authentication using Ed25519 signatures
- Request signing for server-to-server communication
- Error handling and best practices

## User Flow Documentation

The [User Flow](userflow.md) guide covers:

- Step-by-step process for user authentication and registration
- API response format
- UI element specifications
- Common flow actions
- Implementation guidelines
- Security considerations
- Example flows

## KLB API Describe Tool

The [KLB API Describe Tool](klbfw-describe.md) manual covers:

- Installation and basic usage
- Command-line options
- Output formats (formatted, raw JSON, TypeScript)
- Understanding different API endpoint types
- Best practices for API discovery and integration
- Calling conventions and API path structure
- Troubleshooting and common issues

## KLB Framework JavaScript

The [KLB Framework JavaScript](klbfw-js.md) documentation covers:

- Setting up and integrating the JavaScript framework
- Available components and utilities
- Authentication integration
- API communication
- Event handling
- Client-side validation
- Examples and usage patterns

## KLB Framework i18n

The [KLB Framework i18n](klbfw-i18next.md) guide covers:

- Setting up internationalization with i18next
- Preparing CSV translation files
- Using translations in your code
- Language detection
- Troubleshooting and best practices

## KLB Framework Flutter

The [KLB Framework Flutter](klbfw-flutter.md) documentation covers:

- Installing and initializing the Flutter package
- Authentication with OAuth2
- Making API requests (authenticated, unauthenticated, optional authentication)
- User management features
- File uploads with progress tracking
- Deep link handling
- Error handling

## Stripe Express Checkout

The [Stripe Express Checkout](express_checkout.md) guide covers:

- Integrating Stripe Express Checkout in Vue.js applications
- Setting up Apple Pay and Google Pay buttons
- Handling both express checkout and standard card payments
- Creating orders on-the-fly during payment
- Managing the complete payment lifecycle
- Handling redirects and payment confirmations
- Testing express checkout in development

## Media Variations API

The [Media Variations API](media_variation.md) documentation covers:

- Requesting transformed versions of images and audio files
- Available filters for image processing (scaling, effects, format conversion)
- Audio format conversion and processing options
- Handling multi-page documents and PDFs
- Best practices for responsive images and media optimization
- Performance considerations and CDN delivery

## Version Service Worker

The [Version Service Worker](install-version-service-worker.md) guide covers:

- Setting up a service worker that adds version headers to asset requests
- Implementing version tracking using Git commit hashes
- Configuring webpack plugins for automatic version injection
- Integrating version information into Vue.js applications
- Enabling better cache management and version-specific routing
- Browser compatibility and troubleshooting

## Upload Protocol

The [Upload Protocol](upload_lowlevel.md) specification covers:

- Low-level upload protocol for environments without a native klbfw SDK
- Two-phase upload process (negotiation and transfer)
- Direct PUT upload method with chunking support
- AWS S3 multipart upload method with request signing
- Error handling and retry behavior
- Implementation checklists and example code

## Getting Started

1. Review the [API Basics](apibasics.md) document to understand how to authenticate and make API requests
2. Understand the [Access Levels](access_levels.md) to work with permissions and user groups
3. Learn about [OAuth2 Authentication](oauth2_auth.md) for token management and API key signing
4. Follow the [User Flow](userflow.md) documentation to implement user authentication and registration
5. Use the [KLB API Describe Tool](klbfw-describe.md) to explore and understand available API endpoints
6. Integrate the [KLB Framework JavaScript](klbfw-js.md) for client-side implementation
7. Set up [i18n with i18next](klbfw-i18next.md) for multi-language support
8. Use the [KLB Framework Flutter](klbfw-flutter.md) for mobile application development
9. Implement [Stripe Express Checkout](express_checkout.md) for streamlined payment processing
10. Utilize [Media Variations API](media_variation.md) for optimized media delivery
11. Set up [Version Service Worker](install-version-service-worker.md) for asset version tracking

## Using with Claude

When working with Claude on KLB system projects, add the describe MCP tool to claude (replace "project" with "user" for global usage):

    claude mcp add klbfw-describe -s project -- npx -y @karpeleslab/klbfw-describe --mcp

## Contact

For support or questions about integrating with Karpeles Lab Inc systems, please contact the development team.
