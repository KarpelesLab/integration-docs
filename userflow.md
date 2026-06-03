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
  "message": "Login", // Message to display (was flow_action_login token)
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

If a redirection is needed (oauth2, etc), the response will look as follows:

```json
{
  "complete": false,
  "url": "https://...",       // URL to redirect to
}
```

## UI Elements (Fields)

The `fields` array contains UI elements that should be rendered by the client. Each element has a specific structure based on its type:

### Label

```json
{
  "cat": "label",
  "type": "label",
  "label": "Please provide your email in order to login",
  "style": "error",  // Optional - can be "error" for error messages
  "link": "url_or_action" // Optional - makes the label clickable with a URL or action
}
```

#### Action Links

Actions can be triggered using the `@action` syntax in the `link` field:

```json
{
  "cat": "label",
  "type": "label",
  "label": "Forgot your password?",
  "link": "@action=reset_password"
}
```

When the user clicks on a label with an `@action` link, the client should:
1. Reset the current session data
2. Initiate a new flow with the specified action
3. Stay on the same UI view
4. Send a request to the API with the new action but no session token

The server will respond with the initial step of the new flow action. This allows for seamless transitions between different flows (e.g., from login to password reset) without redirecting the user.

### Text Input

```json
{
  "cat": "input",
  "name": "email",
  "type": "email", // Can be: text, email, password, phone
  "label": "Email",
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
  "label": "I agree to the Terms of Service",
}
```

### Select Input

```json
{
  "cat": "input",
  "type": "select",
  "name": "country",
  "label": "Select your country",
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

### Dynamic Select Input

For select inputs that need to fetch options dynamically from an API:

```json
{
  "cat": "input",
  "type": "select",
  "name": "country",
  "label": "Select your country",
  "source": {
    "api": "Country",
    "label_field": "Name",
    "key_field": "Country__"
  },
  "default": "US"
}
```

When a select field has a `source` property:
- The API specified in `api` must be called to retrieve the select options
- `label_field` specifies the field name to use for display text of each option
- `key_field` specifies the field name to use for the value of each option
- If a value is already set for this field, it should be pre-selected
- If no value is set but `default` is provided, that value should be pre-selected

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

### Passkey

A passkey field invites the browser to perform a WebAuthn assertion (login) or attestation (registration). It is rendered as a "Sign in with passkey" button, and on supporting platforms it can also drive *conditional autofill* on the email input next to it.

```json
{
  "cat": "special",
  "type": "passkey",
  "rp_id": "hub.atonline.com",
  "rp_name": "AtOnline",
  "challenge": "7TjygqRej68y8eiWmd01yDQu2qxMEJJH",
  "timeout": 120000,
  "submit_as": "passkey",
  "submit_register_as": "passkey_register"
}
```

| Field | Meaning |
|---|---|
| `rp_id` | The WebAuthn relying-party identifier — pass to `navigator.credentials.{get,create}` as `publicKey.rpId`. |
| `rp_name` | Human-readable relying-party name. Only used at registration time. |
| `challenge` | Base64-encoded one-time challenge. Must be decoded to raw bytes before being passed to the browser, and *also* echoed back in the submission body so the server can validate the single-use TempVal. |
| `timeout` | Milliseconds. Pass through to `publicKey.timeout`. |
| `submit_as` | The key under which the login assertion should be POSTed on the next flow tick (e.g. `"passkey"`). |
| `submit_register_as` | The key under which a registration attestation should be POSTed. Only relevant if the form is also being used for first-time enrollment. |

A passkey field can appear in three places:

1. **On the initial login form** — alongside the email input. Calling `navigator.credentials.get(...)` here is a passwordless first-factor login that completes the entire flow in one round trip (the server marks the OTP step as already satisfied because the authenticator's user-verification is itself a multi-factor).
2. **Alongside the OTP input on `check_otp login`** — when the user has at least one enrolled passkey, the field appears next to the standard OTP code input. The user can pick either one; whichever response is submitted is accepted.
3. **As the only field during `add_passkey`** — the `submit_register_as` key is what the client should use after `navigator.credentials.create(...)` succeeds.

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

### Input Validation

Input fields can include a `validation` property to enforce specific validation rules on the client side.

#### Validation Types

| Type | Description |
|------|-------------|
| `equal_other_field` | Validates that the field value matches another field's value |

#### equal_other_field

Used to validate that one field's value matches another field. Common use case is password confirmation:

```json
{
  "cat": "input",
  "name": "password2",
  "type": "password",
  "label": "Confirm Password",
  "validation": {
    "type": "equal_other_field",
    "field": "password"
  }
}
```

When a field has this validation:
- The client should validate that the value entered matches the value of the field specified in `field`
- Validation should occur before form submission
- If validation fails, display an appropriate error message (e.g., "Passwords do not match")

## Common Flow Actions

The system supports various flow actions:

- `login`: User authentication
- `register`: New user registration
- `recover_account`: Account recovery
- `delete_account`: Account deletion
- `reset_password`: Password reset
- `change_email`: Email change
- `switch`: Switch between accounts (e.g., parent/child)
- `add_passkey`: Enroll a new passkey on the current account (requires a fresh OTP check)

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
   
6. **Handle Flow Transitions:**
   - Labels with `link` containing "@action=..." indicate flow navigation options
   - When a user clicks on an @action link, the client should reset the session and start a new flow with the specified action
   - This keeps the user on the same UI while changing the logical flow (e.g., switching from login to password reset)

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
     ?session=[updated_session_token]
     ```
   - Extract the updated `session` parameter from the URL

5. **Continuing the Flow:**
   - Send a request to the flow API with only the updated session token:
     ```
     {
       "session": "[updated_session_token_from_callback]"
     }
     ```
   - The API will continue the flow

6. **Completion:**
   - The API will respond with `"complete": true` and tokens or user data
   - Use the returned tokens for API access or follow any provided redirect

## Passkey Integration

Passkeys are exposed through the [passkey field](#passkey). The client is responsible for the browser-side `navigator.credentials.{get,create}` calls; the server is responsible for minting and validating challenges. All payload encoding is base64 unless noted as base64url.

### Login (assertion)

1. **Find the field**: a `cat: "special", type: "passkey"` element appeared in the latest `fields` array.

2. **Call the browser**: decode `field.challenge` to raw bytes and call:

    ```js
    const cred = await navigator.credentials.get({
      publicKey: {
        challenge: <decoded bytes>,
        rpId: field.rp_id,
        allowCredentials: [],            // discoverable
        userVerification: 'required',
        timeout: field.timeout,
      },
      mediation: 'optional',             // see "Conditional autofill" below
    });
    ```

   If no eligible passkey exists or the user dismisses the prompt, `get()` rejects — fall back to email + password.

3. **Submit the assertion**: POST the next flow tick with the same `session` token and a single new key whose name is `field.submit_as` (typically `passkey`):

    ```json
    {
      "session": "<token>",
      "passkey": {
        "challenge":         "<echo of field.challenge>",
        "id":                "<base64url(cred.rawId)>",
        "authenticatorData": "<base64(response.authenticatorData)>",
        "clientDataJSON":    "<base64(response.clientDataJSON)>",
        "signature":         "<base64(response.signature)>",
        "userHandle":        "<base64url(response.userHandle) or null>"
      }
    }
    ```

4. **Flow continues**: the server identifies the user from `userHandle`, verifies the signature, and treats the assertion as both authentication *and* second factor. The next response is typically `complete: true`.

### Registration (attestation) via `add_passkey`

The supported way to enroll a new passkey on an existing account is the `add_passkey` flow action, which first requires the user to authenticate (`authenticate login_only`) and pass a fresh security OTP (`check_otp security`).

1. Client initiates the flow: `{ "action": "add_passkey" }`.
2. Server steps the user through authentication and OTP (the standard fields apply).
3. Once those are cleared, the server returns a passkey field whose `submit_register_as` key indicates how to send the attestation back. Some attestation-related options accompany the challenge — `pubKeyCredParams`, `authenticatorSelection`, `attestation`, `user`, etc. (Browser-specific WebAuthn defaults can be used where the field is silent.)
4. Client calls `navigator.credentials.create({...})` with those parameters.
5. Client submits:

    ```json
    {
      "session": "<token>",
      "passkey_create": {
        "id":                "<base64url(cred.rawId)>",
        "clientDataJSON":    "<base64(response.clientDataJSON)>",
        "attestationObject": "<base64(response.attestationObject)>"
      }
    }
    ```

6. Server validates the attestation, stores the credential against the user, and returns `complete: true`.

### Conditional autofill (recommended)

On platforms that support it (iOS 16+, macOS Ventura+, Chrome 108+), you can fire a `mediation: 'conditional'` call once at page load *and* set `autocomplete="username webauthn"` on the email input. The browser will offer the user's saved passkey directly inside the email field's autocomplete menu. The promise resolves only if the user picks one — if they type an email instead, the promise stays pending and you simply use the email path.

```js
if (await PublicKeyCredential.isConditionalMediationAvailable?.()) {
  navigator.credentials.get({
    publicKey: { challenge: <decoded>, rpId: field.rp_id, allowCredentials: [], userVerification: 'required' },
    mediation: 'conditional',
  }).then(cred => cred && submitFlowTick({ [field.submit_as]: pack(cred) })).catch(() => {});
}
```

```html
<input name="email" type="email" autocomplete="username webauthn">
```

### Passkey as second factor

When the user is identified (after submitting an email + password) and has at least one enrolled passkey, the OTP step emits the passkey field *alongside* the regular OTP code input. The client should present both options — pick whichever the user uses. The submission key is again whatever `field.submit_as` says (typically `passkey_response` in this context); the server detects it on the same flow tick and accepts the assertion in place of an OTP code.

### Failure modes

| Symptom | Likely cause | What to do |
|---|---|---|
| `error_passkey_expired` | Challenge TTL elapsed (120s) or already consumed | Re-fetch the form to mint a new challenge |
| `error_passkey_missing_field: …` | Client omitted one of the required keys | Verify all six keys (`challenge, id, authenticatorData, clientDataJSON, signature, userHandle`) are present |
| `invalid clientData origin` | Frontend is being served from a host other than `rp_id` | The page must be served from the same domain configured as the realm's main domain |
| `invalid rpIdHash` | Client passed a different `rpId` to the browser than the field's `rp_id` | Always pass `field.rp_id` verbatim |
| `user verification flag not set` | Authenticator returned `UV=0` (no biometric/PIN check) | Re-prompt with `userVerification: 'required'`; non-UV authenticators are rejected by policy |
| `no account matches this passkey` | The credential was created against a different realm, or the user was deleted | Treat as "no passkey" and fall back to email entry |

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

### Example Flow: Passkey Login

1. Client initiates login flow
2. Server returns initial fields including an email input *and* a passkey field
3. Client decodes `field.challenge`, calls `navigator.credentials.get(...)` with `allowCredentials: []` and `userVerification: 'required'`
4. The browser surfaces eligible passkeys; the user picks one and confirms with Touch ID / Face ID / PIN
5. Client packages the assertion (`id`, `authenticatorData`, `clientDataJSON`, `signature`, `userHandle`) plus the echoed `challenge` and POSTs `{ passkey: {...}, session: <token> }`
6. Server identifies the user via `userHandle`, verifies the signature against the stored credential, and accepts the assertion as both authentication and second factor
7. Server returns `complete: true` with user data
8. Client redirects to the designated page or application

### Example Flow: Passkey Enrollment (`add_passkey`)

1. Authenticated user initiates the `add_passkey` flow
2. Server validates the existing session and steps the user through a security OTP
3. Server returns a passkey field carrying the registration challenge and (where applicable) `pubKeyCredParams`, `authenticatorSelection`, `attestation` and `user` parameters
4. Client calls `navigator.credentials.create(...)` with those parameters; the user confirms enrollment with their authenticator
5. Client POSTs `{ passkey_create: { id, clientDataJSON, attestationObject }, session: <token> }`
6. Server verifies the attestation, stores the credential against the user, and returns `complete: true`

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
