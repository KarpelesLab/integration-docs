# OAuth2 Authentication

This document describes how OAuth2 authentication works with the API, including token management, automatic renewal, and API key authentication as an alternative.

## Table of Contents

1. [Overview](#overview)
2. [Authentication Methods](#authentication-methods)
3. [OAuth2 Token Authentication](#oauth2-token-authentication)
4. [Token Refresh Flow](#token-refresh-flow)
5. [API Key Authentication](#api-key-authentication)
6. [Error Handling](#error-handling)

## Overview

The API supports two primary authentication methods:

1. **OAuth2 Bearer Tokens** - Standard OAuth2 flow with automatic token refresh
2. **API Key Signing** - Ed25519-signed requests for server-to-server communication

Both methods attach credentials to the request context, and authentication is handled transparently for all subsequent API calls.

## Authentication Methods

| Method | Use Case | Credentials |
|--------|----------|-------------|
| OAuth2 Token | User authentication, web/mobile apps | Access token + Refresh token |
| API Key | Server-to-server, automated systems | Key ID + Ed25519 secret |

## OAuth2 Token Authentication

### Token Structure

An OAuth2 token consists of the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `access_token` | string | The bearer token used for API requests |
| `refresh_token` | string | Token used to obtain a new access token when expired |
| `token_type` | string | Token type, typically `"Bearer"` |
| `expires_in` | integer | Token lifetime in seconds (default: 3600) |
| `client_id` | string | OAuth2 client identifier (required for refresh) |

### Obtaining a Token

Tokens are obtained through the User Flow system (see [userflow.md](userflow.md)) or via standard OAuth2 flows:

1. **User Flow Login** - When a user completes authentication through the User Flow system, the response includes an OAuth2 token
2. **OAuth2 Authorization Code** - Standard OAuth2 authorization code flow with PKCE support
3. **OAuth2 Refresh** - Exchange a refresh token for a new access token

### Using a Token

Include the access token in the `Authorization` header for all authenticated requests:

```
Authorization: Bearer <access_token>
```

### Token Endpoint

**Endpoint:** `OAuth2:token`
**Methods:** `GET`, `POST`

#### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `grant_type` | Yes | The grant type (`refresh_token`, `authorization_code`) |
| `client_id` | No | OAuth2 client identifier |
| `redirect_uri` | No | Redirect URI for authorization code flow |
| `client_secret` | No | Client secret for confidential clients |
| `code_verifier` | No | PKCE code verifier |

#### Refresh Token Request

To refresh an expired token:

```
POST OAuth2:token
{
    "grant_type": "refresh_token",
    "client_id": "<client_id>",
    "refresh_token": "<refresh_token>"
}
```

#### Response

```json
{
    "result": "success",
    "data": {
        "access_token": "<new_access_token>",
        "refresh_token": "<new_refresh_token>",
        "token_type": "Bearer",
        "expires_in": 3600
    }
}
```

## Token Refresh Flow

The API client handles token expiration automatically through the following flow:

```
┌─────────────────────────────────────────────────────────────────┐
│                      API Request Flow                            │
└─────────────────────────────────────────────────────────────────┘

    ┌──────────┐      ┌──────────┐      ┌──────────────────────┐
    │  Client  │ ───▶ │   API    │ ───▶ │  Check Authorization │
    └──────────┘      └──────────┘      └──────────────────────┘
                                                   │
                           ┌───────────────────────┼───────────────────────┐
                           │                       │                       │
                           ▼                       ▼                       ▼
                    ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
                    │   Valid     │         │   Expired   │         │   Invalid   │
                    │   Token     │         │   Token     │         │   Token     │
                    └─────────────┘         └─────────────┘         └─────────────┘
                           │                       │                       │
                           │                       ▼                       │
                           │              ┌─────────────────┐              │
                           │              │  Refresh Token  │              │
                           │              │     Request     │              │
                           │              └─────────────────┘              │
                           │                       │                       │
                           │         ┌─────────────┴─────────────┐         │
                           │         ▼                           ▼         │
                           │  ┌─────────────┐             ┌─────────────┐  │
                           │  │   Success   │             │   Failure   │  │
                           │  │ New Token   │             │             │  │
                           │  └─────────────┘             └─────────────┘  │
                           │         │                           │         │
                           │         ▼                           │         │
                           │  ┌─────────────┐                    │         │
                           │  │   Retry     │                    │         │
                           │  │   Request   │                    │         │
                           │  └─────────────┘                    │         │
                           │         │                           │         │
                           ▼         ▼                           ▼         ▼
                    ┌─────────────────────────────────────────────────────────┐
                    │                    Return Response                       │
                    └─────────────────────────────────────────────────────────┘
```

### Expiration Detection

Token expiration is detected **server-side**. When a request is made with an expired token, the server responds with:

```json
{
    "result": "error",
    "token": "invalid_request_token",
    "extra": "token_expired"
}
```

### Automatic Refresh

When token expiration is detected:

1. The client automatically calls `OAuth2:token` with `grant_type=refresh_token`
2. If successful, the token is updated in-place with the new `access_token`
3. The original request is automatically retried with the new token
4. The response from the retried request is returned to the caller

### Refresh Requirements

For automatic token refresh to work:

- `client_id` must be set on the token
- `refresh_token` must be present and valid

If either is missing, the refresh fails with an appropriate error.

### Refresh Errors

| Error | Description |
|-------|-------------|
| `ErrNoClientID` | No `client_id` was provided for token renewal |
| `ErrNoRefreshToken` | No refresh token available and access token has expired |

## API Key Authentication

API keys provide an alternative authentication method using Ed25519 cryptographic signatures. This is ideal for server-to-server communication where OAuth2 flows are impractical.

### API Key Structure

| Field | Description |
|-------|-------------|
| `KeyID` | The API key identifier |
| `SecretKey` | Ed25519 private key (64 bytes, base64url-encoded) |

### Creating an API Key

API keys are created through the platform's key management interface. The secret is provided as a base64url-encoded Ed25519 private key.

### Request Signing

Each API request is signed with the following process:

1. **Add authentication parameters:**
   - `_key`: The API key ID
   - `_time`: Current Unix timestamp
   - `_nonce`: A unique UUID for replay protection

2. **Build the signing string:**
   ```
   METHOD\0PATH\0QUERY_STRING\0BODY_HASH
   ```
   Where:
   - `\0` is a null byte separator
   - `BODY_HASH` is the SHA-256 hash of the request body

3. **Sign with Ed25519:**
   - Sign the string using the Ed25519 private key
   - Encode the signature as base64url

4. **Add signature parameter:**
   - `_sign`: The base64url-encoded signature

### Signed Request Example

A signed request includes these query parameters:

```
?_key=key-12345&_time=1703808000&_nonce=550e8400-e29b-41d4-a716-446655440000&_sign=<signature>
```

### Security Properties

- **Replay Protection**: The `_nonce` parameter prevents request replay
- **Time-bound**: The `_time` parameter allows servers to reject old requests
- **Tamper-proof**: The signature covers method, path, query parameters, and body

## Error Handling

### Authentication Errors

| HTTP Code | Error Token | Description |
|-----------|-------------|-------------|
| 401 | `error_authentication_required` | No authentication provided |
| 401 | `invalid_request_token` | Token is invalid or malformed |
| 401 | `token_expired` | Token has expired (check `extra` field) |
| 403 | `error_access_denied` | Authenticated but insufficient permissions |

### Token Expiration Response

```json
{
    "result": "error",
    "token": "invalid_request_token",
    "extra": "token_expired",
    "error": "The access token has expired"
}
```

### Best Practices

1. **Store refresh tokens securely** - Refresh tokens have longer lifetimes and should be protected
2. **Handle refresh failures gracefully** - If refresh fails, prompt the user to re-authenticate
3. **Use appropriate token lifetimes** - Default is 3600 seconds (1 hour)
4. **Implement retry logic** - Clients should handle the automatic retry after token refresh
5. **For server applications, prefer API keys** - They don't expire and are simpler for automated systems
