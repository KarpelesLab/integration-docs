# User Flow v2 Documentation

## Overview

The User Flow v2 system provides a flexible, step-by-step process for user authentication, registration, account management and other user-related workflows. The system is designed to be used by client applications (web or mobile) that communicate with the API, with the client responsible for rendering the UI components returned by the API.

## Flow Process

1. The client application initiates a flow by calling the API endpoint with an action parameter
2. The server responds with UI elements that should be rendered by the client
3. The client renders these elements and collects user input
4. The client sends the input back to the server along with the session token
5. This process repeats until the flow is complete (indicated by `complete: true` in the response)

## API Response Format

The API returns a JSON object with the following structure:

```json
{
  "complete": false,           // Whether the flow is complete
  "initial": true,             // Whether this is the first step
  "email": "user@example.com", // Email if available
  "req": ["field1", "field2"], // Required fields that must be submitted
  "fields": [],                // UI elements to render (see below)
  "message": {"@token": ["flow_action_login"]}, // Message to display
  "session": "encrypted_session_token",         // Session token to pass back
  "realm_flags": ["flag1", "flag2"],            // Realm flags
  "user": {}                   // User object if available
}
```

When the flow is complete, the response will be:

```json
{
  "complete": true,           // Flow is complete
  "user": {},                 // User object
  "Redirect": "/destination", // URL to redirect to
  "Token": "oauth_token"      // OAuth token if applicable
}
```

## UI Elements (Fields)

The `fields` array contains UI elements that should be rendered by the client. Each element has a specific structure based on its type:

### Label

```json
{
  "cat": "label",
  "type": "label",
  "label": {"@token": ["user_email_input_help"]},
  "style": "error",  // Optional - can be "error" for error messages
  "link": "url_or_action" // Optional - makes the label clickable
}
```

### Text Input

```json
{
  "cat": "input",
  "name": "email",
  "type": "email", // Can be: text, email, password, phone
  "label": {"@token": ["input_field_email"]},
  "format": "AAAA-AAAA", // Optional format hint
  "attributes": {"autocomplete": "off"} // Optional HTML attributes
}
```

### Checkbox Input

If checked the value true (or 1 if sending as url encoded or MIME POST data) must be passed into the form.

```json
{
  "cat": "input",
  "name": "agree_terms",
  "type": "checkbox",
  "label": {"@token": ["agree_to_terms"]},
}
```

### Select Input

```json
{
  "cat": "input",
  "type": "select",
  "name": "country",
  "label": {"@token": ["lbl_country_select"]},
  "values": [
    {
      "value": "us",
      "display": "United States"
    },
    {
      "value": "ca",
      "display": "Canada"
    }
  ],
  "default": "us"
}
```

### OAuth2 Button

```json
{
  "type": "oauth2",
  "id": "provider_id",
  "info": {}, // Provider information
  "button": { // Button style
    "text": "Sign in with Google",
    "color": "#4285F4",
    "textColor": "#ffffff",
    "icon": "google_icon.svg"
  }
}
```

### Special Elements

```json
{
  "cat": "special",
  "name": "profile_pic",
  "type": "image",
  "target": "User/@/Profile:addImage",
  "param": {"purpose": "main"}
}
```

## Common Flow Actions

The system supports various flow actions:

- `login`: User authentication
- `register`: New user registration
- `recover_account`: Account recovery
- `delete_account`: Account deletion
- `reset_password`: Password reset
- `change_email`: Email change
- `switch`: Switch between accounts (e.g., parent/child)

## Client Implementation Guide

1. **Start Flow:**
   - Call the API with the desired action (e.g., `login`)
   - Receive initial fields to render

2. **Render UI:**
   - Render the UI elements from the `fields` array
   - Display any messages from the `message` object
   - Mark required fields based on the `req` array

3. **Submit Data:**
   - When user inputs data and submits, send the form data to the API
   - Always include the `session` token in subsequent requests

4. **Continue Flow:**
   - Process the response and render new fields if provided
   - Check the `complete` flag to know when the flow is finished
   - When `complete` is true, follow any redirect or use the provided token

5. **Handle Errors:**
   - If the API returns an error, display it to the user
   - Fields with errors may have a `style: "error"` attribute

## OAuth2 Integration

When the flow presents an OAuth2 button, you need to handle the OAuth2 authentication flow:

1. **OAuth2 Button Handling:**
   - When an OAuth2 button appears in the `fields` array, render it with the styling information provided
   - The button should be associated with the provider's ID from the `id` field

2. **Starting the OAuth2 Flow:**
   - When the user clicks an OAuth2 button, send a request to the flow API with:
     ```
     {
       "oauth2": "[provider_id]",
       "session": "[current_session_token]"
     }
     ```

3. **Redirect to Provider:**
   - The API will respond with a JSON object containing a URL:
     ```json
     {
       "complete": false,
       "url": "https://provider.com/auth?params..."
     }
     ```
   - Redirect the user to this URL to authenticate with the provider

4. **Handling the Callback:**
   - The provider will redirect back to your application with query parameters including:
     ```
     ?code=[authorization_code]&session=[updated_session_token]&state=[state]
     ```
   - Extract the updated `session` parameter from the URL

5. **Continuing the Flow:**
   - Send a request to the flow API with only the updated session token:
     ```
     {
       "session": "[updated_session_token_from_callback]"
     }
     ```
   - The API will exchange the authorization code automatically and continue the flow

6. **Completion:**
   - The API will respond with `"complete": true` and tokens or user data
   - Use the returned tokens for API access or follow any provided redirect

## Security Considerations

- The session token is encrypted and has a 30-minute expiration
- Sensitive flows require OTP verification
- Password inputs should use proper security measures (autocomplete, etc.)
- OAuth flows include proper validation of redirect URIs

## Example Flows

### Example Flow: Login

1. Client initiates login flow
2. Server returns email input field
3. User enters email, client submits
4. Server returns password field
5. User enters password, client submits
6. If OTP is enabled, server returns OTP input
7. User enters OTP, client submits
8. Server returns `complete: true` with user data
9. Client redirects to the designated page or application

### Example Flow: Registration

1. Client initiates registration flow
2. Server returns registration form fields (name, email, password, etc.)
3. User fills out form, client submits
4. If email verification is required, server sends email and returns code input
5. User enters verification code, client submits
6. Server returns `complete: true` with user data
7. Client redirects to the designated page or application

### Example Flow: OAuth2 Login

1. Client initiates login flow
2. Server returns login options including OAuth2 provider buttons
3. User clicks an OAuth2 provider button (e.g., Google)
4. Client sends request with `oauth2: "google"` and current session
5. Client receives URL and redirects user to Google's auth page
6. User authenticates with Google
7. Google redirects back to the application with code and updated session
8. Client extracts the updated session from the URL
9. Client sends request with only the updated session token
10. Server completes the flow and returns user data and tokens
11. Client uses the provided tokens for API access