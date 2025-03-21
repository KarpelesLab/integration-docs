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

# AtOnline Login for Flutter Applications

This guide explains how to implement the AtOnline login flow in your Flutter application using the `atonline_login` package.

## Installation

Add the package to your `pubspec.yaml` file:

```yaml
dependencies:
  atonline_login: ^0.6.0
  atonline_api: ^0.4.19+2  # Required dependency
```

Then run:

```bash
flutter pub get
```

## Basic Setup

### 1. Initialize AtOnline API

First, initialize the AtOnline API in your application as explained above.

### 2. Configure OAuth2 Callback URL Scheme

For OAuth2 authentication flows, register a custom URL scheme in your app:

#### For Android:
Add to `android/app/src/main/AndroidManifest.xml` inside the `<application>` tag:

```xml
<activity android:name="com.linusu.flutter_web_auth_2.CallbackActivity" android:exported="true">
  <intent-filter android:label="flutter_web_auth_2">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="YOUR_APP_SCHEME" />
  </intent-filter>
</activity>
```

#### For iOS:
Add to `ios/Runner/Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLName</key>
    <string>YOUR_BUNDLE_ID</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>YOUR_APP_SCHEME</string>
    </array>
  </dict>
</array>
```

## Implementing the Login Flow

### Using the Built-in Login Widget

The simplest way to implement login is using the provided `AtOnlineLoginPageBody` widget:

```dart
import 'package:atonline_login/atonline_login.dart';
import 'package:flutter/material.dart';

class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Login')),
      body: AtOnlineLoginPageBody(
        atOnlineApi,
        callbackUrlScheme: 'YOUR_APP_SCHEME',
        onComplete: () {
          // Navigate to home screen or main app content
          Navigator.of(context).pushReplacementNamed('/home');
        },
      ),
    );
  }
}
```

## Handling Authentication State

To check if a user is logged in and manage authentication state:

```dart
// Check login status
bool isLoggedIn = atOnlineApi.user.isLoggedIn();

// Get current user info
if (isLoggedIn) {
  final userData = atOnlineApi.user.userData;
  print('Logged in as: ${userData['Profile']['Display_Name']}');
}

// Logout
void logout() async {
  await atOnlineApi.voidToken();
  // Navigate to login screen
}
```

## Customizing the Login Experience

### Custom Styling

The login widget adapts to your app's theme. To customize the appearance, wrap it in a Theme:

```dart
Theme(
  data: Theme.of(context).copyWith(
    primaryColor: Colors.blue,
    elevatedButtonTheme: ElevatedButtonThemeData(
      style: ElevatedButton.styleFrom(
        backgroundColor: Colors.blue,
        foregroundColor: Colors.white,
      ),
    ),
  ),
  child: AtOnlineLoginPageBody(
    atOnlineApi,
    callbackUrlScheme: 'YOUR_APP_SCHEME',
    onComplete: () => Navigator.of(context).pushReplacementNamed('/home'),
  ),
)
```

## Advanced Usage

### Custom Actions

The login flow supports different actions beyond the default "login":

```dart
AtOnlineLoginPageBody(
  atOnlineApi,
  action: 'register', // Other options: 'login', 'password_reset', etc.
  callbackUrlScheme: 'YOUR_APP_SCHEME',
  onComplete: () => Navigator.of(context).pushReplacementNamed('/home'),
)
```

### File Upload Support

The login widget supports file uploads, such as profile images during registration. This is handled automatically through the `ImagePickerWidget` component.

## Troubleshooting

- **OAuth2 Errors**: Ensure your callback URL scheme is correctly configured in both the app manifest/plist and when initializing the widget.
- **Login Failures**: Check that your app ID and realm configuration are correct.
- **Image Upload Issues**: Verify that file permissions are properly configured for both Android and iOS.

## Example Implementation

```dart
import 'package:atonline_api/atonline_api.dart';
import 'package:atonline_login/atonline_login.dart';
import 'package:flutter/material.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  final AtOnline atOnlineApi = AtOnline(appId: 'YOUR_APP_ID');
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'AtOnline Login Demo',
      theme: ThemeData(primarySwatch: Colors.blue),
      routes: {
        '/': (context) => LoginScreen(atOnlineApi: atOnlineApi),
        '/home': (context) => HomeScreen(atOnlineApi: atOnlineApi),
      },
    );
  }
}

class LoginScreen extends StatelessWidget {
  final AtOnline atOnlineApi;
  
  LoginScreen({required this.atOnlineApi});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Login')),
      body: AtOnlineLoginPageBody(
        atOnlineApi,
        callbackUrlScheme: 'YOUR_APP_SCHEME',
        onComplete: () => Navigator.of(context).pushReplacementNamed('/home'),
      ),
    );
  }
}

class HomeScreen extends StatelessWidget {
  final AtOnline atOnlineApi;
  
  HomeScreen({required this.atOnlineApi});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Home'),
        actions: [
          IconButton(
            icon: Icon(Icons.logout),
            onPressed: () async {
              await atOnlineApi.voidToken();
              Navigator.of(context).pushReplacementNamed('/');
            },
          ),
        ],
      ),
      body: Center(
        child: Text(
          'Welcome ${atOnlineApi.user.userData?['Profile']?['Display_Name'] ?? 'User'}!',
          style: TextStyle(fontSize: 24),
        ),
      ),
    );
  }
}
```
