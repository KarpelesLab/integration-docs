# AtOnline API Flutter Package

AtOnline API is a Flutter package that provides tools for interacting with AtOnline.com APIs including CMS, Naka, and other services.

## Installation

Add the package to your `pubspec.yaml`:

```yaml
dependencies:
  atonline_api: ^0.4.17
```

## Features

- OAuth2 Authentication with AtOnline services
- REST API wrapper for AtOnline endpoints
- User management
- Deep link handling
- File upload support with progress callbacks

## Basic Usage

### Initialize the API Client

```dart
import 'package:atonline_api/atonline_api.dart';

// Initialize the client with your app ID
final api = AtOnline('your_app_id_here');
```

### Authentication

The package supports OAuth2 authentication. To set up a login flow:

```dart
// Create a login page using the provided widget
final loginPage = LoginPage(api, 'your_redirect_uri');

// Navigate to the login page
Navigator.of(context).push(MaterialPageRoute(builder: (context) => loginPage));
```

### Making API Requests

#### Unauthenticated Requests

```dart
// Simple GET request
var result = await api.req('path/to/endpoint');

// POST request with body
var result = await api.req(
  'path/to/endpoint', 
  method: 'POST',
  body: {'key': 'value'}
);
```

#### Authenticated Requests

```dart
// Request requiring authentication
var result = await api.authReq('path/to/endpoint');

// POST with authentication
var result = await api.authReq(
  'path/to/endpoint',
  method: 'POST',
  body: {'key': 'value'}
);
```

#### Optional Authentication Requests

For endpoints that can work with or without authentication, but may return additional information when authenticated:

```dart
// Will include authentication if available, but won't fail if not logged in
var result = await api.optAuthReq('path/to/endpoint');

// Example: Getting CMS content that includes user-specific data when authenticated
// - Without auth: Basic content is returned
// - With auth: Additional fields like "has been liked" are included
var cmsEntry = await api.optAuthReq('Content/Cms/Entry', 
  body: {'entry_id': 'some_id'}
);

// POST with optional authentication
var result = await api.optAuthReq(
  'path/to/endpoint',
  method: 'POST',
  body: {'key': 'value'}
);
```

This approach is particularly useful for applications where you want to display content to both anonymous and logged-in users, while providing enhanced functionality to authenticated users without writing separate code paths.

### User Management

```dart
// Check if user is logged in
if (api.user.isLoggedIn()) {
  // User is logged in
  print('User email: ${api.user.info?.email}');
  print('User name: ${api.user.info?.displayName}');
}

// Fetch user information
await api.user.fetchLogin();

// Update user profile
await api.user.updateProfile({
  'Display_Name': 'New Name'
});

// Set profile picture
File imageFile = File('path/to/image.jpg');
await api.user.setProfilePicture(imageFile);

// Logout
await api.user.logout();
```

### File Uploads

```dart
File fileToUpload = File('path/to/file.pdf');

// Upload with progress tracking
var result = await api.authReqUpload(
  'path/to/upload/endpoint',
  fileToUpload,
  progress: (double progress) {
    print('Upload progress: ${(progress * 100).toStringAsFixed(2)}%');
  }
);
```

### Deep Link Handling

Set up deep link handling for your app:

```dart
// Initialize links handler
await Links.init();

// Add listener for specific paths
Links().addListener('/login', (Uri link) {
  // Handle the login link
  print('Got login link: $link');
});

// Remove listener when done
Links().removeListener('/login', yourListener);
```

## Error Handling

The package includes custom exception classes for different error types:

```dart
try {
  var result = await api.authReq('path/to/endpoint');
  // Process result
} on AtOnlineLoginException catch (e) {
  // Handle login/authentication errors
  print('Login error: ${e.msg}');
} on AtOnlinePlatformException catch (e) {
  // Handle platform-specific errors
  print('Platform error: ${e.data}');
} on AtOnlineNetworkException catch (e) {
  // Handle network-related errors
  print('Network error: ${e.msg}');
}
```

## Additional Notes

- The package automatically manages OAuth tokens including refresh
- Secure storage is used for tokens
- The API client is a singleton per app ID
- Results from API calls are returned as iterable `AtOnlineApiResult` objects