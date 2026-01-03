# Frontend HTMX Integration Guide

This guide explains how to build server-rendered frontends using HTMX with the platform's templating system.

## Table of Contents

- [Getting Started](#getting-started)
- [Template Directives](#template-directives)
- [REST API Integration](#rest-api-integration)
- [HTMX Response Control](#htmx-response-control)
- [Routing](#routing)
- [Security](#security)
- [Examples](#examples)

---

## Getting Started

### Enabling HTMX Mode

Add to your `etc/registry.ini`:

```ini
Web_Engine=htmx
```

Your HTML files should be placed in the `dist/` directory (or configure via `Javascript_Dist_Dir`).

### Basic Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My App</title>
  <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
  <div id="content">
    <!-- Your content here -->
  </div>
</body>
</html>
```

### The FW Object

A global `FW` JavaScript object is automatically injected containing:

- `FW.token` - Session token for API authentication
- `FW.i18n` - Translation data
- `FW.Registry` - Configuration values
- `FW.mode` - Client mode (e.g., "client", "client-admin")

HTMX requests automatically include the session token via the `Authorization` header.

---

## Template Directives

### Conditional Rendering: `klb-if`

Conditionally render elements based on expressions.

```html
<div klb-if="user.is_admin">
  Admin Panel
</div>

<div klb-if="!user.is_admin">
  Regular User View
</div>

<span klb-if="items.length > 0">
  {items.length} items found
</span>
```

### Loops: `klb-loop` and `klb-target`

Iterate over arrays.

```html
<ul>
  <li klb-loop="users" klb-target="user">
    {user.name} - {user.email}
  </li>
</ul>
```

Available loop variables:
- `{item}` - Current item (or custom name via `klb-target`)
- `{item_key}` - Current index (0-based)
- `{item_first}` - `true` if first item
- `{item_last}` - `true` if last item

```html
<div klb-loop="products" klb-target="product">
  <span klb-if="product_first" class="badge">First!</span>
  #{product_key}: {product.name}
  <hr klb-if="!product_last">
</div>
```

### Variable Interpolation

Use `{variable}` syntax in text and attributes:

```html
<div id="user-{user.User__}">
  <img src="{user.avatar_url}" alt="{user.name}">
  <a href="/users/{user.User__}">{user.name}</a>
</div>
```

Nested properties are supported:

```html
{user.Profile.Display_Name}
{order.items[0].product.name}
```

### Internationalization: `klb-i18n`

Translate text using i18n tokens.

```html
<klb-i18n token="welcome_message">Welcome!</klb-i18n>

<button>
  <klb-i18n token="btn_submit">Submit</klb-i18n>
</button>
```

The text inside the element serves as the fallback if the token is not found.

### Container Elements

#### `klb-block`

A container that renders its children without outputting itself:

```html
<klb-block klb-if="show_section">
  <h2>Section Title</h2>
  <p>Section content</p>
</klb-block>
```

#### `klb-head`

Move content to the `<head>` element:

```html
<klb-head>
  <link rel="stylesheet" href="/css/page-specific.css">
  <script src="/js/page-specific.js"></script>
</klb-head>
```

---

## REST API Integration

### Basic Usage: `klb-rest`

Call REST APIs during server-side rendering.

```html
<klb-rest path="User:get" User__="usr-123abc" klb-target="user">
  <h1>Hello, {user.Profile.Display_Name}!</h1>
  <p>Email: {user.Email}</p>
</klb-rest>
```

### Attributes

| Attribute | Description | Default |
|-----------|-------------|---------|
| `path` | API endpoint (e.g., `User:get`, `Order:list`) | Required |
| `method` | HTTP method: `GET`, `POST`, `PATCH`, `DELETE` | `GET` |
| `klb-target` | Variable name for the result | Spreads to context |
| `onerror` | Set to `ignore` to handle errors gracefully | Fails on error |
| Other attributes | Become request parameters | - |

### HTTP Methods

```html
<!-- GET (default) -->
<klb-rest path="User:get" User__="{id}" klb-target="user">
  {user.name}
</klb-rest>

<!-- POST -->
<klb-rest path="Order:create" method="POST"
          Product__="{product.id}" quantity="1"
          klb-target="order">
  Order #{order.Order__} created!
</klb-rest>

<!-- PATCH -->
<klb-rest path="User/@:update" method="PATCH"
          Display_Name="{newName}" klb-target="updated">
  Profile updated!
</klb-rest>

<!-- DELETE -->
<klb-rest path="Cart/Item:delete" method="DELETE"
          Cart_Item__="{item.id}" onerror="ignore">
  <span klb-if="_success">Removed!</span>
</klb-rest>
```

### Error Handling

With `onerror="ignore"`, failed requests set special variables:

- `_success` - `true` if request succeeded, `false` on error
- `_error` - Error message string (only on failure)

```html
<klb-rest path="User:create" method="POST"
          Email="{form.email}" Password="{form.password}"
          klb-target="result" onerror="ignore">

  <div klb-if="_success" class="success">
    Account created! Welcome, {result.Email}
  </div>

  <div klb-if="!_success" class="error">
    Failed: {_error}
  </div>
</klb-rest>
```

### Dynamic Parameters

Parameters support variable interpolation:

```html
<klb-rest path="Product:search"
          category="{selected_category}"
          min_price="{filters.min}"
          max_price="{filters.max}"
          klb-target="products">
  <div klb-loop="products.data" klb-target="product">
    {product.name} - ${product.price}
  </div>
</klb-rest>
```

---

## HTMX Response Control

Control HTMX behavior from your templates using `klb-hx-*` directives.

### Client-Side Redirect

```html
<klb-hx-redirect url="/dashboard"/>
```

Redirect after a successful action:

```html
<klb-rest path="User:login" method="POST" ... onerror="ignore">
  <klb-hx-redirect url="/dashboard" klb-if="_success"/>
  <div klb-if="!_success">Login failed</div>
</klb-rest>
```

### URL History

```html
<!-- Push new URL to history -->
<klb-hx-push-url url="/users/{user.User__}"/>

<!-- Replace current URL -->
<klb-hx-replace-url url="/users/{user.User__}/edit"/>
```

### Trigger Client Events

Trigger custom events that JavaScript can listen to:

```html
<klb-hx-trigger event="userUpdated"/>

<!-- With timing -->
<klb-hx-trigger event="showNotification" after="settle"/>
<klb-hx-trigger event="refreshList" after="swap"/>
```

JavaScript listener:

```javascript
document.body.addEventListener('userUpdated', function(e) {
  console.log('User was updated!');
});
```

### Response Targeting

Change where the response content goes:

```html
<!-- Retarget to a different element -->
<klb-hx-retarget selector="#notification-area"/>

<!-- Change swap method -->
<klb-hx-reswap method="beforeend"/>
```

Swap methods: `innerHTML`, `outerHTML`, `beforebegin`, `afterbegin`, `beforeend`, `afterend`, `delete`, `none`

### Out-of-Band Updates

Update multiple elements with a single response:

```html
<!-- Mark elements to include in response -->
<klb-hx-oob id="cart-count"/>
<klb-hx-oob id="notifications"/>
```

The elements with those IDs will be included with `hx-swap-oob="true"`.

### Full Page Refresh

Force a complete page reload:

```html
<klb-hx-refresh/>
```

---

## Routing

### Defining Routes

Use `<klb-route>` to define URL patterns:

```html
<!-- In dist/user.html -->
<klb-route path="/users/:id"/>
<klb-route path="/profile/:username"/>

<klb-rest path="User:get" User__="{id}" klb-target="user">
  <h1>{user.name}</h1>
</klb-rest>
```

Route variables (`:id`, `:username`) are available in the template context.

### Index Routes

`index.html` is automatically mapped to `/`.

### Static Files

Non-HTML files in `dist/` are served as static assets.

---

## Security

### CSP Nonce

All `<script>`, `<style>`, and `<link rel="stylesheet">` elements automatically receive a CSP nonce attribute.

### Session Token

HTMX requests automatically include the session token:

```
Authorization: Session <token>
```

This is handled automatically via the `htmx:configRequest` event listener.

### CSRF Protection

The session token serves as CSRF protection. For manual fetch requests:

```javascript
fetch('/api/endpoint', {
  headers: {
    'Authorization': 'Session ' + FW.token
  }
});
```

---

## Examples

### User Profile Page

```html
<!-- dist/profile.html -->
<klb-route path="/users/:user_id"/>

<!DOCTYPE html>
<html>
<head>
  <title>User Profile</title>
  <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
  <klb-rest path="User:get" User__="{user_id}" klb-target="user" onerror="ignore">
    <div klb-if="_success" class="profile">
      <img src="{user.Profile.Avatar_Url}" alt="{user.Profile.Display_Name}">
      <h1>{user.Profile.Display_Name}</h1>
      <p>{user.Email}</p>

      <div klb-if="user.is_self">
        <a href="/settings" hx-get="/settings" hx-target="#content">
          Edit Profile
        </a>
      </div>
    </div>

    <div klb-if="!_success" class="error">
      <h1>User Not Found</h1>
      <p>{_error}</p>
    </div>
  </klb-rest>
</body>
</html>
```

### Product Listing with Filters

```html
<!-- dist/products.html -->
<klb-route path="/products"/>

<div id="product-list">
  <klb-rest path="Product:list" category="{category}" klb-target="result">
    <div class="products" klb-loop="result.data" klb-target="product">
      <div class="product" id="product-{product.Product__}">
        <img src="{product.image_url}" alt="{product.name}">
        <h3>{product.name}</h3>
        <p class="price">${product.price}</p>

        <button hx-post="/api/cart/add"
                hx-vals='{"product": "{product.Product__}"}'
                hx-target="#cart-count"
                hx-swap="innerHTML">
          <klb-i18n token="btn_add_to_cart">Add to Cart</klb-i18n>
        </button>
      </div>
    </div>

    <div klb-if="result.paging.next">
      <button hx-get="/products?page={result.paging.next}"
              hx-target="#product-list"
              hx-swap="innerHTML">
        Load More
      </button>
    </div>
  </klb-rest>
</div>

<div id="cart-count">
  <klb-rest path="Cart/@:info" klb-target="cart">
    {cart.item_count} items
  </klb-rest>
</div>
```

### Form with Validation

```html
<form hx-post="/api/register" hx-target="#form-result" hx-swap="innerHTML">
  <div>
    <label><klb-i18n token="label_email">Email</klb-i18n></label>
    <input type="email" name="Email" required>
  </div>

  <div>
    <label><klb-i18n token="label_password">Password</klb-i18n></label>
    <input type="password" name="Password" required minlength="8">
  </div>

  <button type="submit">
    <klb-i18n token="btn_register">Register</klb-i18n>
  </button>
</form>

<div id="form-result"></div>
```

Server response fragment:

```html
<!-- Returned by POST /api/register -->
<klb-rest path="User:create" method="POST"
          Email="{POST.Email}" Password="{POST.Password}"
          klb-target="result" onerror="ignore">

  <klb-hx-trigger event="userRegistered" klb-if="_success"/>
  <klb-hx-redirect url="/welcome" klb-if="_success"/>

  <div klb-if="!_success" class="error">
    <klb-i18n token="error_registration_failed">Registration failed</klb-i18n>: {_error}
  </div>
</klb-rest>
```

### Modal Dialog

```html
<button hx-get="/modals/confirm-delete?id={item.id}"
        hx-target="#modal-container"
        hx-swap="innerHTML">
  Delete
</button>

<div id="modal-container"></div>
```

Modal template (`dist/modals/confirm-delete.html`):

```html
<klb-route path="/modals/confirm-delete"/>

<div class="modal-backdrop">
  <div class="modal">
    <h2><klb-i18n token="confirm_delete">Confirm Delete</klb-i18n></h2>
    <p><klb-i18n token="confirm_delete_message">Are you sure?</klb-i18n></p>

    <button hx-delete="/api/items/{id}"
            hx-target="#item-{id}"
            hx-swap="delete"
            hx-on::after-request="document.getElementById('modal-container').innerHTML=''">
      <klb-i18n token="btn_delete">Delete</klb-i18n>
    </button>

    <button onclick="document.getElementById('modal-container').innerHTML=''">
      <klb-i18n token="btn_cancel">Cancel</klb-i18n>
    </button>
  </div>
</div>
```

---

## Quick Reference

### Template Elements

| Element | Purpose |
|---------|---------|
| `<klb-rest>` | Call REST API |
| `<klb-block>` | Container (no output) |
| `<klb-head>` | Add to `<head>` |
| `<klb-i18n>` | Translate text |
| `<klb-route>` | Define URL route |

### Template Attributes

| Attribute | Purpose |
|-----------|---------|
| `klb-if` | Conditional rendering |
| `klb-loop` | Iterate array |
| `klb-target` | Loop variable name |

### HTMX Response Directives

| Element | Purpose |
|---------|---------|
| `<klb-hx-redirect>` | Client redirect |
| `<klb-hx-push-url>` | Push URL to history |
| `<klb-hx-replace-url>` | Replace URL in history |
| `<klb-hx-refresh>` | Full page refresh |
| `<klb-hx-trigger>` | Trigger client event |
| `<klb-hx-retarget>` | Change response target |
| `<klb-hx-reswap>` | Change swap method |
| `<klb-hx-oob>` | Out-of-band update |
