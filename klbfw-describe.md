# KLB API Describe Tool Manual

## Introduction

The `klbfw-describe` tool is a command-line utility designed to help developers explore, understand, and interact with KLB API endpoints. It provides detailed information about API endpoints including available methods, arguments, fields, and sub-endpoints, making API discovery and integration faster and easier.

This tool performs OPTIONS requests on API endpoints to discover their capabilities and structure, presenting the information in a user-friendly format with examples for immediate use.

## Installation

You can use the tool directly without installation using `npx`:

```bash
npx @karpeleslab/klbfw-describe User
```

Or install it globally for easier access:

```bash
npm install -g @karpeleslab/klbfw-describe
klbfw-describe User
```

## Basic Usage

```
klbfw-describe [options] <api-path>
```

If run without an API path, the tool will display a list of all available API objects that you can explore further.

### Options

- `--raw`: Show the raw JSON response without formatting
- `--ts, --types`: Generate TypeScript type definitions for the API structures
- `--host <hostname>`: Specify a custom API host (default: ws.atonline.com)
- `--help, -h`: Display help information

### Examples

```bash
# Display all available API objects
klbfw-describe

# Describe a specific API endpoint
klbfw-describe User

# Describe a nested endpoint
klbfw-describe Misc/Debug

# Describe a specific procedure on an endpoint
klbfw-describe Misc/Debug:testUpload

# Get raw JSON output for advanced use cases
klbfw-describe --raw User

# Generate TypeScript type definitions
klbfw-describe --ts User

# Use a custom API host
klbfw-describe --host api.example.com User
```

## Output Types

The tool provides different output formats depending on your needs:

1. **Default formatted output**: A human-readable, colorized representation of the API endpoint
2. **Raw JSON output** (`--raw`): The complete, unformatted JSON response from the API
3. **TypeScript definitions** (`--ts`): Generated TypeScript interfaces and types for use in your projects

## Understanding API Endpoint Types

The KLB API consists of three primary endpoint types, and `klbfw-describe` adapts its output based on the type:

### 1. Resources

Resources represent database-backed objects with CRUD operations.

Example output for a resource:
```
API: User

Type: Resource
Methods: GET, POST, PATCH, DELETE
Access: user

Resource Details:
Name: User
Fields: 15 fields
Primary Key: User__

All Fields:
  User__ (char) *
  Email (varchar) *
    User's email address
  ...
```

### 2. Procedures

Procedures are API methods that perform specific operations.

Example output for a procedure:
```
API: Misc/Debug:testUpload

Type: Procedure
Methods: POST
Access: internal

Procedure Details:
Name: testUpload
Type: Static Method

Description:
Test file uploads with this endpoint

Usage:
# JavaScript
klbfw.rest('Misc/Debug:testUpload', 'POST', {file: "..."})

Arguments:
  file (file) *
    File to upload for testing
```

### 3. Collections

Collections group related endpoints or provide multiple methods.

Example output for a collection:
```
API: Payment

Type: Collection
Methods: GET, POST
Access: user

Available Methods:
  processPayment (static)
    Process a new payment
    Arguments:
      amount (number) *
        Payment amount
      currency (string) *
        Currency code (USD, EUR, etc.)

Sub-endpoints:
  M: Method, Provider
  T: Transaction
```

## Best Practices for API Integration

### 1. API Discovery

Start by exploring available endpoints:
```bash
klbfw-describe
```

This shows all available top-level API objects. Navigate deeper by specifying paths:
```bash
klbfw-describe User
klbfw-describe Payment/Transaction
```

### 2. Understanding Resource Structures

When working with resources, examine their structure:
```bash
klbfw-describe User
```

Pay attention to:
- Required fields (marked with *)
- Field types and constraints
- Primary keys for referencing objects

### 3. Working with Procedures and Methods

For procedures and methods, focus on:
- Required arguments and their types
- Usage examples provided by the tool
- Return type information when available

Use the examples directly in your code with minimal adjustments.

### 4. TypeScript Integration

For TypeScript projects, generate type definitions:
```bash
klbfw-describe --ts User > user.types.ts
```

This creates a file with TypeScript interfaces for:
- Resource fields
- Method arguments
- Response structures
- KlbDateTime fields (if applicable)

### 5. Error Handling

When integrating with the KLB API:
- Check for required fields before sending requests
- Handle HTTP status codes appropriately
- Parse error messages from the response body when requests fail

## Understanding API Paths and Calling Conventions

### Resource API Paths

For resources, use these conventions:

- **Collection**: `/Resource` - Works with multiple objects
  ```javascript
  // List all users
  klbfw.rest('User', 'GET')
  
  // Create a new user
  klbfw.rest('User', 'POST', { email: 'user@example.com', ... })
  ```

- **Individual resource**: `/Resource/{id}` - Works with a specific object
  ```javascript
  // Get a specific user
  klbfw.rest('User/123', 'GET')
  
  // Update a user
  klbfw.rest('User/123', 'PATCH', { email: 'new@example.com' })
  
  // Delete a user
  klbfw.rest('User/123', 'DELETE')
  ```

### Procedures

For procedures, use these conventions:

- **Static procedures**: `/Resource:procedure` - Called on the resource type
  ```javascript
  // Call a static procedure
  klbfw.rest('User:resetPassword', 'POST', { email: 'user@example.com' })
  ```

- **Instance procedures**: `/Resource/{id}:procedure` - Called on specific instance
  ```javascript
  // Call a procedure on a specific resource
  klbfw.rest('User/123:verifyEmail', 'POST', { code: '123456' })
  ```

## Advanced Usage

### Exploring Raw Responses

For debugging or advanced integration, you can examine the raw JSON:
```bash
klbfw-describe --raw User
```

This shows the complete API description including internal fields that may not be shown in the formatted output.

## Troubleshooting

### Common Issues

1. **API Request Failed**: If you receive an error about being unable to fetch API information:
   - Check your network connection
   - Verify the hostname is correct
   - Ensure you have proper access permissions

2. **Unknown API Path**: If you specify an invalid API path:
   - Run without parameters to see available top-level objects
   - Check for typos in the path
   - Try exploring parent paths to find the correct structure

3. **Authentication Issues**: Some API endpoints require authentication:
   - This tool focuses on discovery and does not handle authentication
   - Use the information gathered with this tool to make authenticated requests in your application

## Conclusion

The `klbfw-describe` tool is designed to make API exploration and integration easier by providing clear, structured information about API endpoints. By combining the usage examples, TypeScript definitions, and the information provided in the formatted output, you can quickly understand and integrate with the KLB API in your applications.

Remember to use the `--help` option anytime you need a quick reference to the available commands and options.
