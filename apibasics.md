# PHP Platform API Documentation

This document explains how to interact with the REST API system and provides practical information for API consumers.

## Table of Contents

1. [API Basics](#api-basics)
2. [Authentication](#authentication)
3. [Request Format](#request-format)
4. [Response Format](#response-format)
5. [HTTP Status Codes](#http-status-codes)
6. [Error Handling](#error-handling)
7. [Pagination](#pagination)
8. [Resource Operations](#resource-operations)
9. [Special Parameters](#special-parameters)
10. [Query Filters](#query-filters)
11. [Rate Limiting](#rate-limiting)
12. [Example Requests](#example-requests)

## API Basics

The API follows RESTful principles where resources are accessed through URL paths. Each resource supports standard HTTP methods for different operations.

Base URL structure: `/_rest/[ResourceType]/[ResourceID]`

Example:
- `/_rest/User` - Access the User collection
- `/_rest/User/usr-abcdef123` - Access a specific user

## Authentication

The API supports multiple authentication methods:

1. **Bearer Token Authentication**:
   ```
   Authorization: Bearer <token>
   ```

2. **API Key Authentication**:
   ```
   ?_key=<apikey>
   ```

3. **CSRF Token Authentication** (for browser-based applications):
   ```
   _csrf_token=<token>
   ```

Most API endpoints require authentication. Unauthenticated requests to protected endpoints will return a 403 error.

## Request Format

You can send data to the API in several formats:

1. **Query Parameters** (for GET requests):
   ```
   GET /_rest/User?Name=John&Email=john@example.com
   ```

2. **Form Data** (for POST/PATCH):
   ```
   Content-Type: application/x-www-form-urlencoded
   
   Name=John&Email=john@example.com
   ```

3. **JSON** (preferred for POST/PATCH):
   ```
   Content-Type: application/json
   
   {
     "Name": "John",
     "Email": "john@example.com"
   }
   ```

4. **JSON in GET requests** (using the `_` parameter):
   ```
   GET /_rest/User?_={"Name":"John","Age":{"$gt":21}}
   ```
   This is useful for complex queries that would be cumbersome as individual query parameters. If using an abstraction library the library will handle that automatically, so always pass the parameters as an object as you would usually do.

## Response Format

All API responses are wrapped in a standard JSON structure:

```json
{
  "result": "success",
  "request_id": "unique-request-id",
  "time": 0.123,
  "data": {
    // The actual response data
  }
}
```

When querying collections, the `data` field contains an array of items:

```json
{
  "result": "success",
  "request_id": "unique-request-id",
  "time": 0.123,
  "data": [
    { "id": "usr-123abc", "Name": "John Doe" },
    { "id": "usr-456def", "Name": "Jane Smith" }
  ],
  "paging": {
    "page_no": 1,
    "count": 42,
    "page_max": 3,
    "results_per_page": 20
  },
  "access": {
    "usrr-fensyz-rdgj-ff7m-2arf-biwzp7dm": {
      "required": "R",
      "available": "O",
      "expires": null,
      "user_group": "usg-3dz36z-v7s5-h6to-3qlp-nbcynpru",
      "group": {
        "User_Group__": "usg-3dz36z-v7s5-h6to-3qlp-nbcynpru",
        "User__": "usr-tif4np-ymkf-dcle-a6ez-uyioajly",
        "Name": "Group Name",
        "Type": "group",
        "Nickname": "groupnn",
        "Status": "private",
        "Backend": "none",
        "Owner": {
          "User_Id": "usr-tif4np-ymkf-dcle-a6ez-uyioajly",
          "User__": "usr-tif4np-ymkf-dcle-a6ez-uyioajly",
          "Index": "2",
          "Link": null,
          "UUID": "9a0bc6bf-0c51-4625-901e-26698438095e",
          "Profile": {
            "Display_Name": "MagicalTux",
            "Username": null,
            "Gender": null
          },
          "Display_Name": "MagicalTux",
          "Email": "m***@***.com",
        }
      }
    }
  }
}
```

The `access` field is included when the API response contains access right information for objects. It provides details about what access rights are required and available for each object, along with group information when applicable.

To receive raw data without the wrapper, add `?raw` to your request URL.

## HTTP Status Codes

| Code | Description                                    | Common Cases                                    |
|------|------------------------------------------------|------------------------------------------------|
| 200  | OK                                             | Successful request                              |
| 201  | Created                                        | Successful resource creation                    |
| 204  | No Content                                     | Successful request with no response body        |
| 400  | Bad Request                                    | Missing or invalid parameters                   |
| 401  | Unauthorized                                   | Authentication required                         |
| 302  | Found                                          | Redirect to another URL                         |
| 403  | Forbidden                                      | Insufficient permissions                        |
| 404  | Not Found                                      | Resource not found                              |
| 405  | Method Not Allowed                             | HTTP method not supported for this endpoint     |
| 408  | Request Timeout                                | Request took too long to process                |
| 429  | Too Many Requests                              | Rate limit exceeded                             |
| 500  | Internal Server Error                          | Server-side error                               |

## Error Handling

When an error occurs, the response structure changes:

```json
{
  "result": "error",
  "request_id": "unique-request-id",
  "time": 0.123,
  "error": "A human-readable error message",
  "token": "error_code"
}
```

Common error tokens include:
- `error_authentication_required`: Authentication needed
- `error_access_denied`: Insufficient permissions
- `error_missing_field`: Required parameter missing
- `error_invalid_value`: Invalid parameter value
- `error_not_found`: Resource not found
- `http_method_not_allowed`: Method not allowed for this endpoint
- `error_rate_limit_exceeded`: Rate limit exceeded
- `error_execution_timeout`: Request timed out

For field validation errors, additional field-specific information may be included.

### Redirects

Some API requests may result in a redirect response. In these cases, the response structure will be:

```json
{
  "result": "redirect",
  "request_id": "unique-request-id",
  "time": 0.123,
  "redirect": "https://example.com/new-location"
}
```

Common redirect scenarios include:
- Authentication required (redirects to login page)
- Resource moved to a new location
- Completing a workflow that requires redirection

The HTTP status code for redirects is typically 302 (Found).

## Pagination

Collection endpoints return paginated results. Pagination information is included in the `paging` field of the response:

```json
"paging": {
  "page_no": 1,
  "count": 42,
  "page_max": 3,
  "results_per_page": 20
}
```

To control pagination, use these parameters:
- `page_no`: The page number to retrieve (starting from 1)
- `results_per_page`: Number of results per page (default: 20, max: 100)

Example: `/_rest/User?page_no=2&results_per_page=10`

## Resource Operations

The API supports standard CRUD operations using HTTP methods:

| Operation | HTTP Method | URL                   | Description                           |
|-----------|-------------|------------------------|---------------------------------------|
| List      | GET         | `/_rest/Resource`     | Get a paginated list of resources     |
| Retrieve  | GET         | `/_rest/Resource/id`  | Get a specific resource               |
| Create    | POST        | `/_rest/Resource`     | Create a new resource                 |
| Update    | PATCH       | `/_rest/Resource/id`  | Update a resource (partial update)    |
| Delete    | DELETE      | `/_rest/Resource/id`  | Delete a resource                     |

Note that the API primarily supports PATCH for resource updates, not PUT.

For custom operations, append a colon and the method name:
```
GET /_rest/Resource:customMethod
POST /_rest/Resource:customMethod
```

## Special Parameters

Several special parameters modify API behavior:

- `_expand`: Include related resources in the response
- `_nonce`: Prevent duplicate requests (useful for POST operations)
- `raw`: Return raw data without the standard wrapper
- `pretty`: Format JSON response with indentation for readability
- `_select_fields`: Comma-separated list of fields to include in the response
- `_minimum_right`: Minimum access level required for the request (for certain endpoints)
- `_`: JSON object containing complex query parameters (for GET requests)

## Query Filters

The API supports a rich set of query filters to find resources matching specific criteria. These can be applied directly as query parameters or included in the `_` JSON parameter.

Basic comparison operators:

| Operator | Description           | Example                                |
|----------|-----------------------|----------------------------------------|
| Direct   | Exact match           | `?Name=John`                           |
| `$prefix`| Starts with           | `?Name[$prefix]=Jo`                    |
| `$anywhere`| Contains            | `?Name[$anywhere]=oh`                  |
| `$like`  | SQL LIKE pattern      | `?Name[$like]=Jo%n`                    |
| `$in`    | Match any value       | `?Status[$in]=active,pending`          |
| `$not`   | Not equal to          | `?Status[$not]=inactive`               |
| `$gt`    | Greater than          | `?Age[$gt]=18`                         |
| `$gte`   | Greater than or equal | `?Age[$gte]=18`                        |
| `$lt`    | Less than             | `?Price[$lt]=100`                      |
| `$lte`   | Less than or equal    | `?Price[$lte]=100`                     |
| `$between`| Between two values   | `?Age[$between]=[18,65]`               |
| `$null`  | Is null or not null   | `?Parent[$null]=true`                  |

Complex example using the `_` parameter:
```
/_rest/User?_={"Age":{"$gt":21},"Status":{"$in":["active","trial"]},"Name":{"$prefix":"J"}}
```

## Rate Limiting

API requests are subject to rate limiting. When a rate limit is exceeded, you'll receive a 429 error.

Rate limiting headers may be included in responses:
```
X-Rate-Limit: 60/60s;b=86400;k=rate_key_123.123.123.123
```

This indicates a limit of 60 requests per 60 seconds, with a block time of 86400 seconds (1 day) if exceeded.

## Example Requests

**Get a list of users**:
```
GET /_rest/User?results_per_page=10
```

**Get users with complex filtering**:
```
GET /_rest/User?_={"Age":{"$gt":21},"Status":"active"}
```

**Get a specific user**:
```
GET /_rest/User/usr-abcdef123
```

**Create a new user**:
```
POST /_rest/User
Content-Type: application/json

{
  "Name": "John Doe",
  "Email": "john@example.com"
}
```

**Update a user**:
```
PATCH /_rest/User/usr-abcdef123
Content-Type: application/json

{
  "Name": "John Smith"
}
```

**Delete a user**:
```
DELETE /_rest/User/usr-abcdef123
```

**Call a custom method**:
```
GET /_rest/Misc/Debug:serverTime
```

**Get raw server time without wrapper**:
```
GET /_rest/Misc/Debug:serverTime?raw
```
