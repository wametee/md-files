# Rental Payment System Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Payment Methods](#payment-methods)
4. [Payment Flow (C2B - Customer to Business)](#payment-flow-c2b---customer-to-business)
5. [B2C Flow (Business to Customer - Payouts)](#b2c-flow-business-to-customer---payouts)
6. [Escrow System](#escrow-system)
7. [Booking-Payment Linking](#booking-payment-linking)
8. [Email Notifications](#email-notifications)
9. [Refunds](#refunds)
10. [Penalties](#penalties)
11. [API Endpoints](#api-endpoints)
12. [Webhooks & Callbacks](#webhooks--callbacks)
13. [Error Handling](#error-handling)
14. [Configuration](#configuration)

---

## Overview

The Panther Ride rental payment system is a comprehensive solution that handles:
- **C2B Payments**: Customer payments via M-Pesa STK Push and Stripe
- **B2C Payouts**: Business-to-Customer payouts to vehicle owners via M-Pesa
- **Escrow Management**: Secure fund holding until rental completion
- **Automatic Linking**: Intelligent payment-booking reconciliation
- **Email Notifications**: Automated transactional emails
- **Refunds**: Full and partial refund processing
- **Penalties**: Automated penalty system for no-shows and vehicle unavailability

### Key Features
- **Payment-First Flow**: Payment can be processed before booking creation
- **Bidirectional Linking**: Payment ↔ Booking references for data integrity
- **Multi-Provider Support**: M-Pesa, Stripe, and extensible for future providers
- **Real-time Status Updates**: Webhook-based payment status updates
- **Currency Conversion**: Automatic USD to KES conversion for M-Pesa payments
- **Idempotent Operations**: Safe retry mechanisms for all payment operations

---

## Architecture

### System Components

```
┌─────────────────┐
│   Frontend      │
│  (React/Vite)   │
└────────┬────────┘
         │
         │ HTTP/REST API
         │
┌────────▼────────────────────────────────────────┐
│           Backend API Server                    │
│  ┌──────────────────────────────────────────┐ │
│  │  Payment Controllers                      │ │
│  │  - mpesa.js (STK Push, B2C)               │ │
│  │  - payments.js (Stripe)                  │ │
│  │  - webhooks.js (Stripe webhooks)         │ │
│  │  - mpesaWebhooks.js (M-Pesa callbacks)   │ │
│  └──────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────┐ │
│  │  Services                                │ │
│  │  - mpesaService.js                       │ │
│  │  - stripeService.js                      │ │
│  │  - pricingService.js                    │ │
│  │  - penaltyService.js                    │ │
│  │  - mailSerives.js                       │ │
│  └──────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────┐ │
│  │  Models                                  │ │
│  │  - Payment                               │ │
│  │  - RentalBooking                        │ │
│  │  - Escrow                               │ │
│  └──────────────────────────────────────────┘ │
└────────┬────────────────────────────────────────┘
         │
         ├──────────────┬──────────────┐
         │              │              │
    ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
    │  M-Pesa │   │ Stripe  │   │ MongoDB │
    │  Daraja │   │   API   │   │         │
    │   API   │   │         │   │         │
    └─────────┘   └─────────┘   └─────────┘
```

### Data Flow

1. **Payment Initiation**: Frontend → Backend → Payment Provider
2. **Payment Processing**: Payment Provider processes payment
3. **Webhook/Callback**: Payment Provider → Backend webhook endpoint
4. **Status Update**: Backend updates Payment and Booking records
5. **Email Notification**: Backend sends email to vehicle owner
6. **Escrow Management**: Funds held until rental completion
7. **Payout**: B2C payment to vehicle owner (if applicable)

---

## Payment Methods

### 1. M-Pesa STK Push (Lipa Na M-Pesa Online)

**Type**: C2B (Customer to Business)  
**Provider**: Safaricom M-Pesa Daraja API  
**Supported Currencies**: KES (Kenyan Shillings)  
**Minimum Amount**: 1 KES  
**Maximum Amount**: 70,000 KES per transaction

#### How It Works
1. Customer enters phone number (254XXXXXXXXX or 07XXXXXXXX)
2. Backend initiates STK Push via Daraja API
3. Customer receives prompt on their phone
4. Customer enters M-Pesa PIN to authorize
5. M-Pesa processes payment and sends callback
6. Backend updates payment status and sends email

#### Phone Number Format
- **Accepted Formats**: `254712345678` or `0712345678`
- **Normalized Format**: Always stored as `254712345678`
- **Validation**: Must be 12 digits starting with `254`

### 2. Stripe

**Type**: C2B (Customer to Business)  
**Provider**: Stripe Payment Intents API  
**Supported Currencies**: USD, EUR, GBP, and others  
**Payment Methods**: Credit/Debit cards, Apple Pay, Google Pay

#### How It Works
1. Frontend creates Payment Intent via Stripe.js
2. Customer enters card details
3. Stripe processes payment
4. Webhook (`payment_intent.succeeded`) updates backend
5. Backend links payment to booking and sends email

### 3. Future Providers

The system is designed to support additional payment providers:
- PayPal
- Flutterwave
- Pesapal
- Bank transfers
---

## Payment Flow (C2B - Customer to Business)

### Payment-First Flow (Recommended)

This flow allows payment to be processed before booking creation, providing better user experience and reducing failed bookings.

```
┌──────────┐
│ Customer │
└────┬─────┘
     │ 1. Select vehicle & dates
     │
┌────▼─────────────────────────────────────┐
│  Frontend: VehicleBookingDialog          │
│  - Calculate pricing                      │
│  - Show payment modal                     │
└────┬─────────────────────────────────────┘
     │ 2. Initiate Payment
     │
┌────▼─────────────────────────────────────┐
│  POST /api/mpesa/initiate                 │
│  Body: {                                  │
│    amount, currency, phoneNumber,        │
│    vehicleOwnerId, ...                    │
│  }                                        │
└────┬─────────────────────────────────────┘
     │ 3. Create Payment Record
     │
┌────▼─────────────────────────────────────┐
│  Payment Model                            │
│  - status: "pending"                      │
│  - mpesaCheckoutRequestId                 │
│  - customerId, vehicleOwnerId             │
└────┬─────────────────────────────────────┘
     │ 4. Initiate STK Push
     │
┌────▼─────────────────────────────────────┐
│  M-Pesa Daraja API                       │
│  - Send STK Push to customer's phone      │
└────┬─────────────────────────────────────┘
     │ 5. Customer authorizes payment
     │
┌────▼─────────────────────────────────────┐
│  M-Pesa Callback                         │
│  POST /api/mpesa/callback                 │
│  - ResultCode: 0 (success)               │
└────┬─────────────────────────────────────┘
     │ 6. Update Payment Status
     │
┌────▼─────────────────────────────────────┐
│  Payment Model                            │
│  - status: "succeeded"                    │
│  - mpesaTransactionId                    │
│  - succeededAt                            │
└────┬─────────────────────────────────────┘
     │ 7. Create Booking
     │
┌────▼─────────────────────────────────────┐
│  POST /api/rentals/rental                 │
│  Body: {                                  │
│    vehicleId, startAt, endAt,            │
│    mpesaCheckoutRequestId, ...           │
│  }                                        │
└────┬─────────────────────────────────────┘
     │ 8. Auto-Link Payment to Booking
     │
┌────▼─────────────────────────────────────┐
│  createBooking Controller                 │
│  - Find payment by checkoutRequestId     │
│  - Link: payment.rentalBookingId          │
│  - Link: booking.paymentId               │
│  - Update booking.paymentStatus: "paid"  │
└────┬─────────────────────────────────────┘
     │ 9. Create Escrow
     │
┌────▼─────────────────────────────────────┐
│  POST /api/escrow/hold                    │
│  Body: {                                  │
│    referenceId: bookingId,               │
│    transactionReference: checkoutRequestId│
│  }                                        │
└────┬─────────────────────────────────────┘
     │ 10. Send Email to Owner
     │
┌────▼─────────────────────────────────────┐
│  mailSerives.SendRentalBookingPaymentEmail│
│  - Booking details                        │
│  - Renter information                     │
│  - Payment breakdown                      │
└──────────────────────────────────────────┘
```

### Booking-First Flow (Legacy)

In this flow, booking is created first, then payment is processed. This is still supported but not recommended.

```
1. Create Booking → 2. Initiate Payment → 3. Process Payment → 4. Link Payment → 5. Create Escrow
```

---

## B2C Flow (Business to Customer - Payouts)

B2C payments are used to send money from the platform to vehicle owners (payouts).

### Use Cases
- **Owner Payouts**: Send owner earnings after rental completion
- **Driver Payments**: Pay drivers for their service
- **Refunds**: Refund customers (if B2C refund is implemented)

### B2C Payment Flow

```
┌─────────────────┐
│ Vehicle Owner   │
│ (Payee)         │
└────────┬────────┘
         │
         │ 1. Rental completed
         │
┌────────▼────────────────────────────────────┐
│  POST /api/mpesa/driver-payout               │
│  Body: { paymentId }                         │
└────────┬────────────────────────────────────┘
         │ 2. Validate Payment
         │
┌────────▼────────────────────────────────────┐
│  Payment Model                               │
│  - status: "succeeded"                       │
│  - payoutStatus: "pending"                   │
│  - ownerEarnings: 8500 (KES cents)          │
└────────┬────────────────────────────────────┘
         │ 3. Format Phone Number
         │
┌────────▼────────────────────────────────────┐
│  mpesaService.formatPhoneNumber()            │
│  - Input: "0712345678" or "254712345678"    │
│  - Output: "254712345678"                   │
└────────┬────────────────────────────────────┘
         │ 4. Initiate B2C Payment
         │
┌────────▼────────────────────────────────────┐
│  M-Pesa Daraja B2C API                      │
│  POST /mpesa/b2c/v1/paymentrequest          │
│  Body: {                                     │
│    InitiatorName, SecurityCredential,       │
│    CommandID: "BusinessPayment",            │
│    Amount: 85,                              │
│    PartyA: shortcode,                      │
│    PartyB: "254712345678",                 │
│    Remarks, QueueTimeOutURL, ResultURL     │
│  }                                          │
└────────┬────────────────────────────────────┘
         │ 5. Update Payment Record
         │
┌────────▼────────────────────────────────────┐
│  Payment Model                               │
│  - payoutStatus: "processing"               │
│  - payoutDetails: {                         │
│      conversationId,                        │
│      originatorConversationId,             │
│      phoneNumber, amount, initiatedAt      │
│    }                                        │
└────────┬────────────────────────────────────┘
         │ 6. M-Pesa Processes Payment
         │
┌────────▼────────────────────────────────────┐
│  M-Pesa B2C Callback                        │
│  POST /api/mpesa/callback                   │
│  (ResultCode: 0 = success)                 │
└────────┬────────────────────────────────────┘
         │ 7. Update Payout Status
         │
┌────────▼────────────────────────────────────┐
│  Payment Model                               │
│  - payoutStatus: "completed"               │
│  - payoutDetails.completedAt               │
└─────────────────────────────────────────────┘
```

### B2C Configuration

**Required Environment Variables**:
```env
MPESA_INITIATOR_NAME=your_initiator_name
MPESA_SECURITY_CREDENTIAL=encrypted_security_credential
MPESA_QUEUE_TIMEOUT_URL=https://your-domain.com/api/mpesa/timeout
```

**Security Credential**:
- Must be encrypted using the M-Pesa public key
- Used to authenticate B2C payment requests
- Different from STK Push credentials

**Limits**:
- **Minimum**: 10 KES
- **Maximum**: 150,000 KES per transaction
- **Daily Limit**: Varies by account tier

### B2C Command IDs

- `BusinessPayment`: Standard business payment (used for owner payouts)
- `SalaryPayment`: Salary payments
- `PromotionPayment`: Promotional payments

---

## Escrow System

The escrow system holds funds securely until rental completion or cancellation.

### Escrow States

```
pending → held → released
              ↓
          refunded
```

- **pending**: Cash payment - no funds held, payment due on completion
- **held**: Digital payment - funds held in escrow
- **released**: Funds released to owner (rental completed)
- **refunded**: Funds refunded to renter (rental cancelled)

### Escrow Lifecycle

#### 1. Hold Funds

**Endpoint**: `POST /api/escrow/hold`

**Request Body**:
```json
{
  "referenceId": "booking_id",
  "referenceType": "rental",
  "payerId": "renter_user_id",
  "payeeId": "owner_user_id",
  "amount": 100.00,
  "currency": "USD",
  "paymentMethod": "mpesa",
  "transactionReference": "ws_CO_16122025120424969705049364"
}
```

**Response**:
```json
{
  "success": true,
  "escrow": {
    "_id": "...",
    "reference_id": "booking_id",
    "reference_type": "rental",
    "status": "held",
    "amount": 100.00,
    "currency": "USD",
    "transaction_reference": "ws_CO_...",
    "held_date": "2025-12-16T08:48:42.938Z"
  }
}
```

#### 2. Release Funds

**Endpoint**: `POST /api/escrow/release`

**When**: Vehicle is picked up (check-in)

**Request Body**:
```json
{
  "referenceId": "booking_id",
  "referenceType": "rental"
}
```

**Response**:
```json
{
  "success": true,
  "escrow": {
    "status": "released",
    "released_date": "2025-12-17T10:00:00.000Z"
  }
}
```

#### 3. Refund Funds

**Endpoint**: `POST /api/escrow/refund`

**When**: Booking is cancelled or declined

**Request Body**:
```json
{
  "referenceId": "booking_id",
  "referenceType": "rental",
  "reason": "Owner declined booking"
}
```

**Response**:
```json
{
  "success": true,
  "escrow": {
    "status": "refunded",
    "refunded_date": "2025-12-16T09:00:00.000Z"
  }
}
```

### Escrow Model

```javascript
{
  reference_id: String,        // Booking ID
  reference_type: String,      // "rental" or "ride"
  payer_id: String,           // Renter user ID
  payee_id: String,           // Owner user ID
  amount: Number,             // Amount in base currency
  currency: String,           // "USD", "KES", etc.
  payment_method: String,     // "mpesa", "stripe", "cash"
  status: String,            // "pending", "held", "released", "refunded"
  transaction_reference: String, // M-Pesa checkoutRequestId or Stripe paymentIntentId
  held_date: Date,
  released_date: Date,
  refunded_date: Date,
  notes: String
}
```

---

## Booking-Payment Linking

The system automatically links payments to bookings using multiple fallback methods.

### Linking Methods (Priority Order)

1. **Direct Link**: `mpesaCheckoutRequestId` passed when creating booking
2. **Escrow Transaction Reference**: Find payment via escrow's `transaction_reference`
3. **Customer/Owner Match**: Find payment by matching `customerId` and `vehicleOwnerId` within 5 minutes

### Automatic Linking in `createBooking`

```javascript
// Method 1: Direct link via checkoutRequestId
if (mpesaCheckoutRequestId) {
  payment = await Payment.findOne({ mpesaCheckoutRequestId });
}

// Method 2: Via escrow transaction_reference
if (!payment) {
  escrow = await Escrow.findOne({
    payer_id: userId,
    payee_id: ownerId,
    reference_type: 'rental',
    createdAt: { $gte: new Date(Date.now() - 60000) }
  });
  if (escrow?.transaction_reference) {
    payment = await Payment.findOne({ 
      mpesaCheckoutRequestId: escrow.transaction_reference 
    });
  }
}

// Method 3: Customer/Owner match
if (!payment) {
  payment = await Payment.findOne({
    customerId: userId,
    vehicleOwnerId: ownerId,
    status: { $in: ['succeeded', 'paid'] },
    rentalBookingId: null,
    createdAt: { $gte: new Date(Date.now() - 300000) }
  });
}

// Bidirectional linking
if (payment) {
  payment.rentalBookingId = booking._id;
  booking.paymentId = payment._id;
  await payment.save();
  await booking.save();
}
```

### Bidirectional References

- **Payment → Booking**: `payment.rentalBookingId`
- **Booking → Payment**: `booking.paymentId`

This ensures data integrity and allows querying from either direction.

---

## Email Notifications

### 1. Booking Payment Email (to Owner)

**Trigger**: Payment succeeds and booking is linked

**Recipient**: Vehicle owner

**Content**:
- Booking details (vehicle, dates, duration)
- Renter information (name, email, phone)
- Payment breakdown (total, owner earnings, platform fee)
- CTA: "View Booking & Confirm"

**Template**: `mailSerives.SendRentalBookingPaymentEmail()`

### 2. Booking Declined Email (to Renter)

**Trigger**: Owner declines booking

**Recipient**: Renter

**Content**:
- Booking declined notification
- Refund information (if payment was made)
- Vehicle details
- CTA: "View Other Available Vehicles"

**Template**: `mailSerives.SendRentalBookingDeclinedEmail()`

### Email Sending Flow

```
Payment Succeeds
    ↓
Webhook/Callback Received
    ↓
Payment Status Updated
    ↓
Booking Linked (if exists)
    ↓
Check: payment.ownerEmailSent === false
    ↓
Send Email
    ↓
Update: payment.ownerEmailSent = true
```

### Email Tracking

The `Payment` model includes:
- `ownerEmailSent`: Boolean flag
- `ownerEmailSentAt`: Timestamp

This prevents duplicate emails if webhooks are received multiple times.

---

## Refunds

### Refund Scenarios

1. **Owner Declines Booking**: Full refund to renter
2. **Owner Cancels Booking**: Full refund to renter
3. **Renter No-Show**: 90% refund to renter, 10% penalty
4. **Vehicle Unavailable**: 100% refund to renter, 10% penalty to owner

### Refund Flow

#### Stripe Refunds

```javascript
// Full refund
const refund = await stripeService.createRefund(
  payment.paymentIntentId,
  null, // Full refund
  "requested_by_customer"
);

// Update payment record
payment.status = "refunded";
payment.refundId = refund.id;
payment.refundAmount = payment.amount;
payment.refundedAt = new Date();
await payment.save();
```

#### M-Pesa Refunds

Currently, M-Pesa refunds are marked in the system but actual B2C refunds need to be implemented:

```javascript
// Mark as refunded
payment.status = "refunded";
payment.refundAmount = payment.amount;
payment.refundReason = "Owner declined booking";
payment.refundedAt = new Date();
await payment.save();

// TODO: Implement M-Pesa B2C refund
// await mpesaService.initiateB2CPayment(
//   renterPhoneNumber,
//   refundAmount,
//   "Refund for booking cancellation"
// );
```

### Refund Endpoints

- **Owner Decline**: `POST /api/rentals/rental/:id/decline`
- **Owner Cancel**: `POST /api/rentals/rental/:id/cancel`
- **Renter Cancel**: `POST /api/rentals/rental/:id/cancel` (with renter validation)

---

## Penalties

### Penalty Types

#### 1. Owner Penalty (Vehicle Unavailable)

**Trigger**: Owner reports vehicle unavailable at pickup time

**Penalty**:
- Owner: 10% of total amount
- Renter: 100% refund

**Implementation**: `penaltyService.applyOwnerPenalty()`

#### 2. Renter No-Show Penalty

**Trigger**: Pickup time passes and renter hasn't shown up

**Penalty**:
- Renter: 10% penalty (90% refund)
- Owner: No penalty

**Implementation**: `penaltyService.applyRenterNoShowPenalty()`

### Penalty Calculation

```javascript
// Owner penalty
const penaltyAmount = totalAmount * 0.10;
const refundAmount = totalAmount; // 100% refund

// Renter no-show penalty
const penaltyAmount = totalAmount * 0.10;
const refundAmount = totalAmount * 0.90; // 90% refund
```

### Penalty Tracking

Stored in `RentalBooking.penaltyApplied`:
```javascript
{
  type: "owner_unavailable" | "renter_no_show",
  amount: Number,
  percentage: Number,
  refundedAmount: Number,
  penalizedParty: "owner" | "renter",
  reason: String,
  at: Date
}
```

---

## API Endpoints

### M-Pesa Endpoints

#### Initiate Payment
```
POST /api/mpesa/initiate
Authorization: Bearer <token>
Body: {
  amount: Number,
  currency: String,
  phoneNumber: String,
  vehicleOwnerId: String,
  callbackUrl?: String
}
Response: {
  success: true,
  checkoutRequestId: String,
  merchantRequestId: String
}
```

#### Query Payment Status
```
GET /api/mpesa/payment-status/:checkoutRequestId
Authorization: Bearer <token>
Response: {
  checkoutRequestId: String,
  resultCode: String,
  resultDesc: String,
  paymentStatus: "pending" | "succeeded" | "failed" | "cancelled",
  failureReason?: String
}
```

#### Initiate B2C Payout
```
POST /api/mpesa/driver-payout
Authorization: Bearer <token>
Body: {
  paymentId: String
}
Response: {
  success: true,
  conversationId: String,
  amount: Number,
  currency: "KES",
  phoneNumber: String,
  status: "processing"
}
```

### Escrow Endpoints

#### Hold Funds
```
POST /api/escrow/hold
Body: {
  referenceId: String,
  referenceType: "rental" | "ride",
  payerId: String,
  payeeId: String,
  amount: Number,
  currency: String,
  paymentMethod: String,
  transactionReference?: String
}
```

#### Release Funds
```
POST /api/escrow/release
Body: {
  referenceId: String,
  referenceType: "rental" | "ride"
}
```

#### Refund Escrow
```
POST /api/escrow/refund
Body: {
  referenceId: String,
  referenceType: "rental" | "ride",
  reason?: String
}
```

#### Get Escrow Status
```
GET /api/escrow/status?referenceId=xxx&referenceType=rental
```

### Rental Endpoints

#### Create Booking
```
POST /api/rentals/rental
Authorization: Bearer <token>
Body: {
  vehicleId: String,
  startAt: ISO8601,
  endAt: ISO8601,
  price: Number,
  pickupLocation: Object,
  dropoffLocation: Object,
  paymentId?: String,
  mpesaCheckoutRequestId?: String
}
```

#### Decline Booking
```
POST /api/rentals/rental/:id/decline
Authorization: Bearer <token>
Body: {
  reason?: String
}
```

#### Cancel Booking
```
POST /api/rentals/rental/:id/cancel
Authorization: Bearer <token>
Body: {
  reason?: String
}
```

---

## Webhooks & Callbacks

### M-Pesa Callbacks

#### STK Push Callback
```
POST /api/mpesa/callback
Content-Type: application/json
Body: {
  Body: {
    stkCallback: {
      MerchantRequestID: String,
      CheckoutRequestID: String,
      ResultCode: Number,
      ResultDesc: String,
      CallbackMetadata: {
        Item: [
          { Name: "Amount", Value: Number },
          { Name: "MpesaReceiptNumber", Value: String },
          { Name: "TransactionDate", Value: String },
          { Name: "PhoneNumber", Value: String }
        ]
      }
    }
  }
}
```

**Result Codes**:
- `0`: Success
- `1`: Insufficient funds
- `1032`: User cancelled
- `1037`: Timeout
- `2001`: Invalid PIN
- `2002`: Invalid phone number

#### B2C Callback
```
POST /api/mpesa/callback
Content-Type: application/json
Body: {
  Result: {
    ResultType: Number,
    ResultCode: Number,
    ResultDesc: String,
    OriginatorConversationID: String,
    ConversationID: String,
    TransactionID: String,
    ResultParameters: {
      ResultParameter: [
        { Key: "TransactionReceipt", Value: String },
        { Key: "TransactionAmount", Value: Number },
        { Key: "B2CWorkingAccountAvailableFunds", Value: Number },
        { Key: "B2CUtilityAccountAvailableFunds", Value: Number },
        { Key: "TransactionCompletedDateTime", Value: String },
        { Key: "ReceiverPartyPublicName", Value: String },
        { Key: "B2CChargesPaidAccountAvailableFunds", Value: Number },
        { Key: "B2CRecipientIsRegisteredCustomer", Value: String }
      ]
    }
  }
}
```

### Stripe Webhooks

#### Payment Intent Succeeded
```
POST /api/webhooks/stripe
Headers: {
  "stripe-signature": String
}
Body: {
  type: "payment_intent.succeeded",
  data: {
    object: {
      id: String,
      amount: Number,
      currency: String,
      metadata: {
        rentalBookingId?: String,
        vehicleId: String,
        renterId: String,
        ownerId: String
      }
    }
  }
}
```

---

## Error Handling

### Common M-Pesa Errors

| Result Code | Description | User-Friendly Message |
|------------|-------------|----------------------|
| 0 | Success | Payment successful |
| 1 | Insufficient funds | Insufficient funds. Please top up and try again. |
| 1032 | User cancelled | Payment was cancelled. Please try again. |
| 1037 | Timeout | Payment request timed out. Please try again. |
| 2001 | Invalid PIN | You entered incorrect M-Pesa PIN. Please try again. |
| 2002 | Invalid phone number | Phone number not registered on M-Pesa. Please register and try again. |

### Error Handling Strategy

1. **Webhook Failures**: Log error, payment status remains "pending"
2. **Email Failures**: Log error, don't fail payment processing
3. **Linking Failures**: Log error, continue with booking creation
4. **Retry Logic**: Frontend polls payment status until resolved

### Error Recovery

- **Missing Callback**: Frontend polling will detect success and trigger email
- **Booking Not Linked**: `createBooking` will auto-link payment
- **Email Not Sent**: `queryPaymentStatus` fallback will send email

---

## Configuration

### Environment Variables

#### M-Pesa Configuration
```env
# STK Push (C2B)
MPESA_CONSUMER_KEY=your_consumer_key
MPESA_CONSUMER_SECRET=your_consumer_secret
MPESA_SHORTCODE=174379
MPESA_PASSKEY=your_passkey
MPESA_CALLBACK_URL=https://your-domain.com/api/mpesa/callback

# B2C (Payouts)
MPESA_INITIATOR_NAME=your_initiator_name
MPESA_SECURITY_CREDENTIAL=encrypted_security_credential
MPESA_QUEUE_TIMEOUT_URL=https://your-domain.com/api/mpesa/timeout

# Environment
MPESA_ENVIRONMENT=sandbox  # or "production"
```

#### Stripe Configuration
```env
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

#### Email Configuration
```env
SENDGRID_API_KEY=SG.xxx...
FRONTEND_URL=http://localhost:5173
```

#### Currency Conversion
```env
USD_TO_KES_RATE=129  # Fallback rate if API fails
```

### M-Pesa Passkey Handling

**Important**: M-Pesa requires exactly 32 characters for STK Push password generation. If Daraja provides a 64-character passkey, only the first 32 characters are used:

```javascript
const passkeyToUse = MPESA_CONFIG.passkey.substring(0, 32);
const passwordString = `${shortcode}${passkeyToUse}${timestamp}`;
const password = Buffer.from(passwordString).toString('base64');
```

---

## Best Practices

### 1. Payment-First Flow
Always use payment-first flow for better user experience and reliability.

### 2. Idempotency
All payment operations are idempotent. Safe to retry if network fails.

### 3. Webhook Verification
Always verify webhook signatures (Stripe) and validate callback data (M-Pesa).

### 4. Error Logging
Log all payment errors with full context for debugging.

### 5. Email Tracking
Use `ownerEmailSent` flag to prevent duplicate emails.

### 6. Currency Handling
Always store amounts in cents (smallest currency unit) for precision.

### 7. Phone Number Normalization
Always normalize phone numbers to `254XXXXXXXXX` format.

### 8. Escrow Management
Always create escrow after payment succeeds and booking is created.

---

## Testing

### Test M-Pesa Payments

1. **Sandbox Test Numbers**: Use Safaricom test numbers
2. **Test Amounts**: Use small amounts (1-10 KES)
3. **Callback Testing**: Use ngrok for local development
4. **Status Queries**: Use `/api/mpesa/payment-status/:checkoutRequestId`

### Test Stripe Payments

1. **Test Cards**: Use Stripe test card numbers
2. **Webhook Testing**: Use Stripe CLI for local webhook testing
3. **Test Mode**: Ensure `STRIPE_SECRET_KEY` starts with `sk_test_`

---

## Troubleshooting

### Payment Not Received
1. Check M-Pesa callback URL is accessible
2. Verify phone number format
3. Check payment status via query endpoint
4. Review webhook logs

### Email Not Sent
1. Check `payment.ownerEmailSent` flag
2. Verify SendGrid API key
3. Check email logs
4. Use fallback email sending in `queryPaymentStatus`

### Booking Not Linked
1. Check `mpesaCheckoutRequestId` is passed
2. Verify escrow `transaction_reference` matches
3. Check customer/owner IDs match
4. Review linking logs in `createBooking`

---

## Future Enhancements

1. **M-Pesa B2C Refunds**: Implement actual B2C refunds for M-Pesa
2. **Partial Refunds**: Support partial refunds for penalties
3. **Multi-Currency**: Full support for multiple currencies
4. **Payment Plans**: Installment payment options
5. **Automated Payouts**: Automatic B2C payouts on rental completion
6. **Payment Analytics**: Dashboard for payment metrics
7. **Fraud Detection**: Implement fraud detection mechanisms

---

## Support

For issues or questions:
- Review logs in `PanthR-Backend/logs/`
- Check M-Pesa Daraja API documentation: https://developer.safaricom.co.ke/
- Check Stripe documentation: https://stripe.com/docs

---

**Last Updated**: December 2025  
**Version**: 1.0.0

