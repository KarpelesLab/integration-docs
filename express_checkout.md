# Stripe Express Checkout Integration Guide

This document explains how to integrate Stripe Express Checkout (Apple Pay, Google Pay, etc.) into your Vue.js application, based on the implementation in StartPage.vue.

## Overview

The Stripe Express Checkout integration provides a streamlined payment experience by allowing users to pay with digital wallets like Apple Pay and Google Pay alongside standard credit card payments. This implementation:

1. **Offers Multiple Payment Options** - Both express checkout buttons and standard card form
2. **Collects Customer Information** - Gathers basic details while minimizing friction
3. **Creates Orders On-the-fly** - Creates orders at payment time rather than in advance
4. **Handles Payment Processing** - Manages the complete payment lifecycle

## Prerequisites

- A Vue.js application
- Access to the same backend API used in this project (which already handles the Stripe account configuration)
- No direct Stripe account management needed - all Stripe configuration is handled by the backend

## Implementation Steps

### 1. Load the Stripe.js Library

Dynamically load the Stripe.js library when the payment component is mounted:

```javascript
loadStripeScript() {
  // Create script element
  const script = document.createElement('script');
  script.src = 'https://js.stripe.com/v3/';
  script.async = true;
  script.onload = () => {
    console.log('Stripe.js loaded successfully');
    // Setup Stripe elements once the script is loaded
    this.$nextTick(() => {
      this.setupStripeElements();
    });
  };
  script.onerror = () => {
    console.error('Failed to load Stripe.js');
    this.stripeError = 'Failed to load payment processor. Please try again later.';
  };
  
  // Add the script to the document
  document.head.appendChild(script);
}
```

### 2. Fetch Stripe Configuration

Retrieve the Stripe payment method configuration from the backend:

```javascript
// Fetch Stripe payment method information
async fetchStripeInfo() {
  try {
    const response = await rest('Realm/PaymentMethod:methodInfo', 'GET', {
      method: 'Stripe'
    });
    
    if (response && response.data) {
      this.stripeMethod = response.data;
      
      // Setup stripe elements in the next tick to ensure DOM is ready
      this.$nextTick(() => {
        this.setupStripeElements();
      });
    }
  } catch (error) {
    console.error('Error fetching Stripe info:', error);
    this.error = 'Unable to initialize payment form. Please try again.';
  }
}
```

### 3. Initialize Stripe Elements with Express Checkout

Set up both standard payment element and express checkout element:

```javascript
setupStripeElements() {
  // Check if we already have Stripe elements or if Stripe method isn't loaded
  if (this.stripeCard || !this.stripeMethod) return;
  
  // Get Stripe key and options from method info
  if (!this.stripeMethod.Fields || !this.stripeMethod.Fields.cc_token || 
      !this.stripeMethod.Fields.cc_token.attributes || !this.stripeMethod.Fields.cc_token.attributes.key) {
    console.error('Missing Stripe configuration');
    this.stripeError = 'Payment configuration error. Please try again or contact support.';
    return;
  }
  
  const stripeKey = this.stripeMethod.Fields.cc_token.attributes.key;
  const stripeOptions = this.stripeMethod.Fields.cc_token.attributes.options || {};
  
  // Load Stripe.js dynamically if not already loaded
  if (!window.Stripe) {
    this.loadStripeScript();
    return;
  }
  
  // Initialize Stripe with the key and options
  const stripeInitOptions = {};
  if (stripeOptions.stripe_account) {
    stripeInitOptions.stripeAccount = stripeOptions.stripe_account;
  }
  
  this.stripe = window.Stripe(stripeKey, stripeInitOptions);
  
  // Create elements instance with deferred payment initialization using cart data
  const elementsOptions = {
    mode: 'payment',
    currency: this.productData.total.currency.toLowerCase(),
    amount: Math.round(parseFloat(this.productData.total_vat.value) * 100),
    setupFutureUsage: 'off_session' // Enable auto-renewal
  };
  
  // Copy other options from the backend
  Object.keys(stripeOptions).forEach(key => {
    if (key !== 'stripe_account') {
      elementsOptions[key] = stripeOptions[key];
    }
  });
  
  this.stripeElements = this.stripe.elements(elementsOptions);
  
  // Create and mount the Express Checkout Element (Apple Pay, Google Pay, etc.)
  if (this.$refs.expressCheckoutElement) {
    this.expressCheckoutElement = this.stripeElements.create('expressCheckout', {
      buttonType: {
        applePay: 'buy',
        googlePay: 'buy',
        paypal: 'checkout',
      },
      buttonTheme: {
        applePay: 'black',
        googlePay: 'black',
      },
      buttonHeight: 48,
      emailRequired: true,
      billingAddressRequired: true,
    });
    
    // Mount the Express Checkout Element
    this.expressCheckoutElement.mount(this.$refs.expressCheckoutElement);
    
    // Listen for payment events from Express Checkout
    this.expressCheckoutElement.on('confirm', async (event) => {
      try {
        // Handle the express checkout payment (see next step)
        await this.handleExpressCheckoutPayment(event);
      } catch (error) {
        console.error('Express checkout payment error:', error);
        this.paymentProcessing = false;
        this.stripeError = error.message || 'Payment processing failed. Please try again.';
      } finally {
        this.isSubmitting = false;
      }
    });
  }
  
  // Also create and mount the standard Payment Element
  this.stripeCard = this.stripeElements.create('payment', {
    fields: {
      billingDetails: {
        name: 'auto',
        email: 'never', // We collect email separately
        phone: 'never',
        address: {
          country: 'auto',
          postalCode: 'auto',
          line1: 'never',
          line2: 'never',
          city: 'never',
          state: 'never',
        }
      }
    }
  });
  
  // Mount the Payment Element
  this.stripeCard.mount(this.$refs.stripeElements);
}
```

### 4. Handle Express Checkout Confirmation

Process the express checkout payment when a digital wallet payment is confirmed:

```javascript
async handleExpressCheckoutPayment(event) {
  // Extract email from the Express Checkout event
  const userEmail = event.billingDetails?.email || this.orderData.email;
  
  // Check for required fields
  if (!userEmail) {
    throw new Error('Email address is required');
  }
  
  this.isSubmitting = true;
  this.paymentProcessing = true;
  this.processingMessage = 'Processing express checkout...';
  
  // Get billing details from the event
  const billingDetails = event.billingDetails || {};
  const billingAddress = billingDetails.address || {};
  
  // Extract name from billing details
  let firstName = this.orderData.firstName;
  let lastName = this.orderData.lastName;
  
  if (billingDetails.name) {
    const lastSpaceIndex = billingDetails.name.lastIndexOf(' ');
    if (lastSpaceIndex !== -1) {
      firstName = billingDetails.name.substring(0, lastSpaceIndex);
      lastName = billingDetails.name.substring(lastSpaceIndex + 1);
    } else {
      // If no space found, use the whole name as lastName
      lastName = billingDetails.name;
    }
  }
  
  // Update orderData with the extracted information
  this.orderData.firstName = firstName;
  this.orderData.lastName = lastName;
  
  // Also update country and zip if available
  if (billingAddress.country) {
    this.orderData.country = billingAddress.country;
  }
  
  if (billingAddress.postal_code) {
    this.orderData.zip = billingAddress.postal_code;
  }
  
  // Create order and initialize payment
  const { orderId, session, clientSecret } = await this.createOrderAndInitPayment({
    email: userEmail,
    billingAddress: {
      country: billingAddress.country || 'US',
      zip: billingAddress.postal_code
    }
  });
  
  // Process the payment with the express checkout
  this.processingMessage = 'Confirming payment...';
  
  // Confirm the payment using the Stripe API
  const { error: confirmError } = await this.stripe.confirmPayment({
    elements: this.stripeElements,
    clientSecret,
    confirmParams: {
      return_url: window.location.origin + this.$router.resolve({ 
        path: `/order/${orderId}` 
      }).href,
      payment_method_data: {
        billing_details: {
          email: userEmail,
          name: `${firstName} ${lastName}`,
        }
      },
    },
    redirect: 'if_required'
  });
  
  if (confirmError) {
    throw new Error(confirmError.message);
  }
  
  // Complete the order processing
  await this.completeOrderProcessing(orderId, session);
}
```

### 5. Create Order and Initialize Payment

Connect to the backend to create an order and get a payment intent client secret:

```javascript
async createOrderAndInitPayment({ email, billingAddress = {} }) {
  try {
    // Create order with product and customer details
    const orderResponse = await rest('Order', 'POST', {
      request: this.productId,
      Email: email,
      Flags: 'autorenew_record', // Enable auto-renewal flag
      Billing: {
        First_Name: this.orderData.firstName || '',
        Last_Name: this.orderData.lastName || '',
        Country__: billingAddress.country || this.orderData.country,
        ...(billingAddress.zip && { Zip: billingAddress.zip })
      }
    });
    
    if (!orderResponse || !orderResponse.data) {
      throw new Error('Failed to create order');
    }
    
    const orderId = orderResponse.data.Order__;
    
    // Initialize the payment process
    this.processingMessage = 'Creating secure payment...';
    
    // First call to process to get payment methods
    const initialProcessResponse = await rest(`Order/${orderId}:process`, 'POST');
    
    if (!initialProcessResponse || !initialProcessResponse.data || !initialProcessResponse.data.methods) {
      throw new Error('Failed to initialize payment');
    }
    
    // Get Stripe method info
    const stripeMethod = initialProcessResponse.data.methods.Stripe;
    
    if (!stripeMethod || !stripeMethod.fields || !stripeMethod.fields.stripe_intent) {
      throw new Error('Stripe payment not available');
    }
    
    // Get the client secret and session
    const clientSecret = stripeMethod.fields.stripe_intent.attributes.client_secret;
    const session = stripeMethod.session;
    
    return {
      orderId,
      session,
      clientSecret
    };
    
  } catch (error) {
    console.error('Order creation error:', error);
    throw error;
  }
}
```

### 6. Complete Order Processing

Finalize the order after payment is confirmed:

```javascript
async completeOrderProcessing(orderId, session) {
  try {
    this.processingMessage = 'Finalizing your order...';
    
    // Second call to process with method, session and Stripe intent
    await rest(`Order/${orderId}:process`, 'POST', {
      method: 'Stripe',
      stripe_intent: '1', // Required when using Stripe payment method
      session,
      cc_remember: true // Enable auto-renewal
    });
    
    // Redirect to order page with order ID
    this.$router.push({
      path: `/order/${orderId}`
    });
    
  } catch (error) {
    console.error('Order completion error:', error);
    throw error;
  }
}
```

### 7. HTML Template Structure

Create a template that includes both express checkout buttons and standard payment form:

```html
<template>
  <div class="checkout-page">
    <!-- Product summary section -->
    <div class="product-summary">
      <!-- Product details -->
    </div>
    
    <!-- Payment section -->
    <div class="payment-section">
      <h2>Payment Details</h2>
      
      <!-- Express Checkout (Apple Pay, Google Pay, etc.) -->
      <div class="express-checkout">
        <div id="express-checkout-element" ref="expressCheckoutElement"></div>
        <div class="divider">- or -</div>
      </div>
      
      <!-- Customer information form -->
      <div class="customer-info">
        <!-- Name, email, etc. fields -->
      </div>
      
      <!-- Standard payment element -->
      <div id="stripe-elements" ref="stripeElements"></div>
      <div v-if="stripeError" class="error-message">{{ stripeError }}</div>
      
      <button @click="handleStripePayment" :disabled="isSubmitting">
        {{ isSubmitting ? 'Processing...' : 'Pay Now' }}
      </button>
    </div>
    
    <!-- Processing state -->
    <div v-if="paymentProcessing" class="processing-overlay">
      <div class="spinner"></div>
      <p>{{ processingMessage }}</p>
    </div>
  </div>
</template>
```

## Styling the Express Checkout Buttons

Apply these styles to create a professional-looking payment form with express checkout buttons:

```css
/* Express checkout section */
.express-checkout {
  margin-bottom: 24px;
}

/* Divider between express checkout and standard payment */
.divider {
  text-align: center;
  margin: 16px 0;
  color: #6b7280;
  font-size: 14px;
  position: relative;
}

.divider:before,
.divider:after {
  content: "";
  position: absolute;
  top: 50%;
  width: 45%;
  height: 1px;
  background-color: #e5e7eb;
}

.divider:before {
  left: 0;
}

.divider:after {
  right: 0;
}

/* Standard payment element */
#stripe-elements {
  padding: 16px;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  margin-bottom: 20px;
}

.error-message {
  color: #ef4444;
  margin-bottom: 16px;
  font-size: 14px;
}

/* Processing overlay */
.processing-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(255, 255, 255, 0.9);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 50;
}

.spinner {
  width: 48px;
  height: 48px;
  border: 4px solid #e5e7eb;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
```

## Handling Browser Redirects

Some express checkout flows may require redirects, so handle the return properly:

```javascript
// In your component's created or mounted hook
created() {
  // Check if this is a redirect back from Stripe
  const query = new URLSearchParams(window.location.search);
  const paymentIntentId = query.get('payment_intent');
  const paymentIntentClientSecret = query.get('payment_intent_client_secret');
  
  if (paymentIntentId && paymentIntentClientSecret) {
    // Handle the redirect return
    this.handleRedirectReturn(paymentIntentId, paymentIntentClientSecret);
  } else {
    // Normal initialization
    this.fetchProductData();
  }
},

// Method to handle redirect return
async handleRedirectReturn(paymentIntentId, clientSecret) {
  try {
    this.isLoading = true;
    
    // Retrieve payment intent status
    const { error, paymentIntent } = await this.stripe.retrievePaymentIntent(clientSecret);
    
    if (error) {
      throw new Error(error.message);
    }
    
    if (paymentIntent.status === 'succeeded') {
      // Payment was successful, redirect to order page
      // Extract order ID from the return_url
      const orderId = this.extractOrderIdFromReturnUrl(paymentIntent.return_url);
      if (orderId) {
        this.$router.push({
          path: `/order/${orderId}`
        });
      } else {
        // Fallback to dashboard if we can't extract the order ID
        this.$router.push('/dashboard');
      }
    } else {
      // Payment failed or still pending
      this.error = 'Payment was not completed. Please try again.';
    }
  } catch (error) {
    console.error('Error handling redirect:', error);
    this.error = error.message || 'An error occurred processing your payment.';
  } finally {
    this.isLoading = false;
  }
},

// Helper to extract order ID from return URL
extractOrderIdFromReturnUrl(returnUrl) {
  if (!returnUrl) return null;
  
  try {
    const url = new URL(returnUrl);
    const path = url.pathname + url.hash;
    const orderMatch = path.match(/\/order\/([^\/\?#]+)/i);
    return orderMatch ? orderMatch[1] : null;
  } catch (e) {
    return null;
  }
}
```

## Testing Express Checkout

For testing express checkout buttons:

1. **Apple Pay**: 
   - Only works on Safari on macOS or iOS
   - Requires a valid Apple Pay certificate on the Stripe account
   - The domain must be properly registered with Apple Pay

2. **Google Pay**:
   - Works on Chrome and other browsers supporting Google Pay
   - Can be tested in development mode

3. **Testing in Development**:
   - For local development, use Stripe's test mode
   - The express checkout buttons will show as "Test Mode" buttons

## Complete Express Checkout Flow

1. User selects product and lands on the payment page
2. Page displays both express checkout buttons and standard payment form
3. User chooses to pay with Apple Pay, Google Pay, or other express checkout option
4. User authorizes payment in the digital wallet popup/sheet
5. Backend creates order and initializes payment intent
6. Stripe processes the payment through the selected payment method
7. Order is completed and user is redirected to the order confirmation page

## Conclusion

This implementation provides a modern, streamlined payment experience by integrating Stripe's Express Checkout buttons alongside standard credit card payment options. The approach reduces checkout friction and offers users their preferred payment methods, potentially increasing conversion rates.

For more information, refer to:
- [Stripe Express Checkout Documentation](https://stripe.com/docs/payments/express-checkout)
- [Stripe Payment Element Documentation](https://stripe.com/docs/payments/payment-element)
- [Stripe Apple Pay Documentation](https://stripe.com/docs/apple-pay)
- [Stripe Google Pay Documentation](https://stripe.com/docs/google-pay)