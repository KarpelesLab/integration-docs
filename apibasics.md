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

Base URL structure: `[ResourceType]/[ResourceID]`

Example:
- `User` - List users
- `User/usr-abcdef123` - Access a specific user
- `User/@` - Some APIs provide short hand calls to get, in this case, the current user
- `User/@/Wallet` - List User/Wallet objects for the current user

## Request Format

Always use the API for the language you're using to make requests.

Typical API request format is rest(path, method, parameters, context)

- Target is the path of the API to call, such as User/@
- Method can be one of GET, POST, PATCH, DELETE, OPTIONS
- Parameters are named (key value) entries to be passed to the API
- Context allows specifying or overriding details relative to the call such as l (language) or g (User/Group id or the "all" string)

## Response Format

All API responses are wrapped in a standard structure:

```json
{
  "result": "success",
  "request_id": "unique-request-id",
  "time": 0.123,
  "data": X // The actual response data
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

Example: `/User` `{"page_no":2,"results_per_page":10}`

## Resource Operations

The API supports standard CRUD operations using HTTP methods:

| Operation | HTTP Method | URL            | Description                           |
|-----------|-------------|----------------|---------------------------------------|
| List      | GET         | `Resource`     | Get a paginated list of resources     |
| Retrieve  | GET         | `Resource/id`  | Get a specific resource               |
| Create    | POST        | `Resource`     | Create a new resource                 |
| Update    | PATCH       | `Resource/id`  | Update a resource (partial update)    |
| Delete    | DELETE      | `Resource/id`  | Delete a resource                     |

Note that the API primarily supports PATCH for resource updates, not PUT.

For custom operations, append a colon and the method name:
```
GET Resource:customMethod
POST Resource:customMethod
```

## Special Parameters

Several special parameters modify API behavior:

- `_expand`: Include related resources in the response
- `pretty`: Format JSON response with indentation for readability
- `_minimum_right`: Minimum access level required for the request (for certain endpoints)

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

Complex example using fictional columns:
```
User {"Age":{"$gt":21},"Status":{"$in":["active","trial"]},"Name":{"$prefix":"J"}}
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
GET User {"results_per_page":10}
```

**Get users with complex filtering**:
```
GET User {"Age":{"$gt":21},"Status":"active"}
```

**Get a specific user**:
```
GET User/usr-abcdef123
```

**Create a new user**:
```
POST User

{
  "Name": "John Doe",
  "Email": "john@example.com"
}
```

**Update a user**:
```
PATCH User/usr-abcdef123
Content-Type: application/json

{
  "Name": "John Smith"
}
```

**Delete a user**:
```
DELETE User/usr-abcdef123
```

**Call a custom method**:
```
GET Misc/Debug:serverTime
```
