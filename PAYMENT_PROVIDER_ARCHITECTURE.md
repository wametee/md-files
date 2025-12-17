# Payment Provider Architecture

## Overview

The payment system is designed to be **provider-agnostic** and easily extensible for future payment providers. The architecture uses a unified `Payment` model that can handle any payment provider through generic fields and flexible metadata.

## Key Design Principles

1. **Unified Payment Model**: All payments (Stripe, M-Pesa, PayPal, Flutterwave, etc.) are stored in the same `Payment` model
2. **Bidirectional References**: `RentalBooking.paymentId` ↔ `Payment.rentalBookingId` for quick lookups
3. **Generic Fields**: `providerTransactionId` and `providerMetadata` work for any provider
4. **Backward Compatibility**: Provider-specific fields (e.g., `paymentIntentId`, `mpesaCheckoutRequestId`) are maintained for existing code

## Payment Model Structure

### Core Fields (Provider-Agnostic)

```javascript
{
  paymentMethod: 'stripe' | 'mpesa' | 'paypal' | 'flutterwave' | 'pesapal' | 'other',
  providerTransactionId: String,  // Generic transaction ID (works for any provider)
  providerMetadata: {              // Flexible object for provider-specific data
    // Can store any provider-specific fields
  },
  rentalBookingId: ObjectId,       // Reference to booking
  customerId: ObjectId,            // Who paid
  vehicleOwnerId: ObjectId,        // Who receives payment
  amount: Number,                  // Amount in cents
  status: String,                  // pending, succeeded, failed, etc.
  currency: String,
}
```

### Provider-Specific Fields (Backward Compatibility)

- **Stripe**: `paymentIntentId`, `chargeId`, `refundId`
- **M-Pesa**: `mpesaCheckoutRequestId`, `mpesaTransactionId`, `mpesaMerchantRequestId`, `mpesaPhoneNumber`

## RentalBooking Model

```javascript
{
  paymentId: ObjectId,  // Direct reference to Payment (for quick lookup)
  // ... other fields
}
```

## How It Works

### 1. Creating a Payment

**For M-Pesa:**
```javascript
const payment = new Payment({
  paymentMethod: 'mpesa',
  providerTransactionId: checkoutRequestId,  // Generic field
  providerMetadata: {                       // Flexible metadata
    checkoutRequestId: checkoutRequestId,
    merchantRequestId: merchantRequestId,
    phoneNumber: phoneNumber,
  },
  // Provider-specific fields (backward compatibility)
  mpesaCheckoutRequestId: checkoutRequestId,
  mpesaMerchantRequestId: merchantRequestId,
  // ... other fields
});
```

**For Stripe:**
```javascript
const payment = new Payment({
  paymentMethod: 'stripe',
  providerTransactionId: paymentIntentId,  // Generic field
  providerMetadata: {                       // Flexible metadata
    paymentIntentId: paymentIntentId,
    chargeId: chargeId,
  },
  // Provider-specific fields (backward compatibility)
  paymentIntentId: paymentIntentId,
  // ... other fields
});
```

**For Future Provider (e.g., PayPal):**
```javascript
const payment = new Payment({
  paymentMethod: 'paypal',
  providerTransactionId: paypalTransactionId,  // Generic field
  providerMetadata: {                         // All PayPal-specific data
    transactionId: paypalTransactionId,
    payerId: payerId,
    orderId: orderId,
    // ... any other PayPal fields
  },
  // No need for provider-specific fields - use providerMetadata
  // ... other fields
});
```

### 2. Linking Payment to Booking

**Bidirectional linking:**
```javascript
// When payment is created
payment.rentalBookingId = booking._id;
booking.paymentId = payment._id;
await payment.save();
await booking.save();
```

### 3. Finding Payment for Cancellation

**Quick lookup using direct reference:**
```javascript
// Fastest method - direct reference
if (booking.paymentId) {
  payment = await Payment.findById(booking.paymentId);
}

// Fallback methods if paymentId not set
if (!payment) {
  payment = await Payment.findOne({ rentalBookingId: booking._id });
}
```

### 4. Getting Transaction ID (Any Provider)

**Unified method:**
```javascript
// Works for any provider
const transactionId = payment.getProviderTransactionId();

// Or use generic field directly
const transactionId = payment.providerTransactionId;
```

## Adding a New Payment Provider

### Step 1: Update Payment Model Enum

```javascript
paymentMethod: {
  type: String,
  enum: ['stripe', 'mpesa', 'paypal', 'flutterwave', 'pesapal', 'your_new_provider'],
  // ...
}
```

### Step 2: Create Payment Record

```javascript
const payment = new Payment({
  paymentMethod: 'your_new_provider',
  providerTransactionId: providerTransactionId,  // Use generic field
  providerMetadata: {                           // Store all provider-specific data
    transactionId: providerTransactionId,
    // ... any other provider-specific fields
  },
  // ... standard fields (amount, customerId, etc.)
});
```

### Step 3: Update Helper Method (Optional)

If you want `getProviderTransactionId()` to work for your provider:

```javascript
// In Payment model
paymentSchema.methods.getProviderTransactionId = function() {
  if (this.providerTransactionId) {
    return this.providerTransactionId;
  }
  
  switch (this.paymentMethod) {
    case 'stripe':
      return this.paymentIntentId || this.chargeId;
    case 'mpesa':
      return this.mpesaTransactionId || this.mpesaCheckoutRequestId;
    case 'your_new_provider':
      return this.providerMetadata?.transactionId;
    // ...
  }
};
```

### Step 4: Use in Controllers

```javascript
// Cancel booking - works for any provider
if (booking.paymentId) {
  payment = await Payment.findById(booking.paymentId);
  
  // Process refund based on provider
  if (payment.paymentMethod === 'stripe') {
    // Stripe refund logic
  } else if (payment.paymentMethod === 'mpesa') {
    // M-Pesa refund logic
  } else if (payment.paymentMethod === 'your_new_provider') {
    // Your new provider refund logic
  }
}
```

## Benefits

1. **Single Reference Point**: `booking.paymentId` works for any provider
2. **No Schema Changes**: Adding new providers doesn't require schema changes (use `providerMetadata`)
3. **Backward Compatible**: Existing provider-specific fields still work
4. **Flexible**: `providerMetadata` can store any provider-specific data
5. **Quick Lookups**: Direct reference is faster than searching
6. **Unified Interface**: `getProviderTransactionId()` works for all providers

## Migration Path

For existing payments:
- `providerTransactionId` can be populated from existing provider-specific fields
- `providerMetadata` can be populated with existing provider data
- No breaking changes - old code continues to work

## Example: Adding PayPal

```javascript
// 1. Create payment
const payment = new Payment({
  paymentMethod: 'paypal',
  providerTransactionId: paypalOrderId,
  providerMetadata: {
    orderId: paypalOrderId,
    payerId: payerId,
    captureId: captureId,
    // ... any PayPal-specific data
  },
  customerId: userId,
  vehicleOwnerId: ownerId,
  amount: amountInCents,
  // ... other fields
});

// 2. Link to booking
booking.paymentId = payment._id;
payment.rentalBookingId = booking._id;

// 3. Cancel booking (works the same way)
if (booking.paymentId) {
  const payment = await Payment.findById(booking.paymentId);
  
  if (payment.paymentMethod === 'paypal') {
    // PayPal refund logic
    const refund = await paypalService.refund(payment.providerTransactionId);
    payment.status = 'refunded';
    payment.providerMetadata.refundId = refund.id;
  }
}
```

## Best Practices

1. **Always use `providerTransactionId`** for new providers
2. **Store provider-specific data in `providerMetadata`** instead of adding new schema fields
3. **Use `booking.paymentId`** for quick lookups instead of searching
4. **Keep provider-specific fields** for backward compatibility only
5. **Use `getProviderTransactionId()`** method for unified transaction ID access


