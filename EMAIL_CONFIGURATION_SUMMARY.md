# Email Configuration Summary

## Overview
This document summarizes when and how emails are sent in the rental booking system.

## Email Types

### 1. Booking Payment Email (to Owner)
**When:** When a payment succeeds (M-Pesa or Stripe)
**Recipient:** Vehicle owner
**Purpose:** Notify owner that a booking has been paid and requires confirmation

**Trigger Points:**
- ✅ M-Pesa webhook callback (`handleSTKPushCallback` in `mpesaWebhooks.js`)
- ✅ Stripe webhook callback (`handlePaymentIntentSucceeded` in `webhooks.js`)
- ✅ Fallback in `queryPaymentStatus` (if webhook missed)

**Prevention of Duplicates:**
- Uses `payment.ownerEmailSent` flag to prevent duplicate sends
- Flag is set to `true` after successful email send
- Flag is checked before sending email

**Configuration:**
- File: `services/mailSerives.js` → `SendRentalBookingPaymentEmail`
- Click tracking: **DISABLED** (to prevent URL wrapping)
- Frontend URL: Uses `buildFrontendUrl("/rental-history")` from `utils/frontendUrl.js`

---

### 2. Booking Declined Email (to Renter)
**When:** When owner declines a pending booking
**Recipient:** Renter (customer)
**Purpose:** Notify renter that booking was declined and refund processed

**Trigger Points:**
- ✅ Owner declines booking via `declineBooking` endpoint
- ✅ Refund is processed (if payment was made)
- ✅ Email sent with refund details

**Configuration:**
- File: `services/mailSerives.js` → `SendRentalBookingDeclinedEmail`
- Click tracking: **DISABLED** (to prevent URL wrapping)
- Frontend URL: Uses `buildFrontendUrl("/rental-history")` from `utils/frontendUrl.js`

**Email Content Includes:**
- Booking details (vehicle, dates, duration)
- Refund amount and method (if refunded)
- Decline reason
- Link to browse other vehicles

---

### 3. Booking Cancelled Email (to Renter)
**When:** When owner cancels a booking (after confirmation)
**Recipient:** Renter (customer)
**Purpose:** Notify renter that booking was cancelled and refund processed

**Trigger Points:**
- ✅ Owner cancels booking via `cancelBooking` endpoint
- ✅ Refund is processed (if payment was made)
- ✅ Email sent with refund details

**Configuration:**
- Uses same function as decline email: `SendRentalBookingDeclinedEmail`
- Same configuration as decline email

---

## Email Sending Flow

### Payment Success Flow:
```
1. Payment succeeds (M-Pesa/Stripe)
   ↓
2. Webhook callback received
   ↓
3. Payment record updated (status = 'succeeded')
   ↓
4. Check if payment.ownerEmailSent === false
   ↓
5. Find booking via payment.rentalBookingId
   ↓
6. Populate booking with owner, renter, vehicle
   ↓
7. Get owner email (from populated data or direct lookup)
   ↓
8. Send email via SendGrid
   ↓
9. Set payment.ownerEmailSent = true
   ↓
10. Update booking.paymentStatus = 'paid'
```

### Decline/Cancel Flow:
```
1. Owner declines/cancels booking
   ↓
2. Process refund (if payment was made)
   ↓
3. Update booking status to 'cancelled'
   ↓
4. Populate booking with renter, vehicle
   ↓
5. Get renter email (from populated data or direct lookup)
   ↓
6. Send email via SendGrid with refund details
   ↓
7. Return success response
```

---

## Email Configuration Settings

### SendGrid Configuration:
- **From Email:** `info@blackpanthertkn.org`
- **From Name:** `Black Panther DAO`
- **Reply To:** `info@blackpanthertkn.org`

### Tracking Settings (All Emails):
```javascript
trackingSettings: {
  clickTracking: { enable: false, enableText: false },
  openTracking: { enable: false },
  subscriptionTracking: { enable: false },
  ganalytics: { enable: false }
}
```

### Frontend URL Configuration:
- Uses `buildFrontendUrl()` from `utils/frontendUrl.js`
- Priority:
  1. `FRONTEND_URL` environment variable
  2. Auto-detect from `Environment` or `NODE_ENV`
  3. Default: `http://localhost:5173` (development)

---

## Testing

### Reset Email Flag for Testing:
```bash
# Reset ownerEmailSent flag for a payment
node scripts/reset-payment-email-flag.js <paymentId>
```

### Manually Send Owner Email:
```bash
# Manually trigger owner email for a payment/booking
node scripts/send-owner-email.js <paymentId> [bookingId]
```

### Test Email Sending:
```bash
# Test email sending functionality
node scripts/test-email-sending.js <email>
```

---

## Troubleshooting

### Email Not Received:
1. **Check SendGrid Dashboard:** https://app.sendgrid.com/activity
2. **Check Server Logs:** Look for `📧` emoji logs
3. **Verify Email Flag:** Check if `ownerEmailSent` is `true` (prevents duplicates)
4. **Check Spam Folder:** Emails might be filtered
5. **Verify SendGrid API Key:** Check `.env` file

### Email Flag Preventing Sends:
- This is **intentional** to prevent duplicate emails
- Use `reset-payment-email-flag.js` script to reset for testing
- In production, flag should remain `true` after first send

### URL Wrapping Issue:
- Click tracking is **disabled** in all emails
- If URLs are still wrapped, check SendGrid account-level settings
- Disable click tracking at account level in SendGrid dashboard

---

## Files Involved

### Email Service:
- `services/mailSerives.js` - Email sending functions

### Controllers:
- `controllers/mpesaWebhooks.js` - M-Pesa payment webhook handler
- `controllers/webhooks.js` - Stripe payment webhook handler
- `controllers/rentals.js` - Booking decline/cancel handlers
- `controllers/mpesa.js` - Payment status query (fallback email)

### Utilities:
- `utils/frontendUrl.js` - Frontend URL builder
- `utils/email.js` - Base email configuration (not used for rental emails)

### Scripts:
- `scripts/send-owner-email.js` - Manually send owner email
- `scripts/reset-payment-email-flag.js` - Reset email flag for testing
- `scripts/test-email-sending.js` - Test email functionality

---

## Status: ✅ All Emails Configured

- ✅ Booking payment email (to owner) - Configured
- ✅ Booking declined email (to renter) - Configured
- ✅ Booking cancelled email (to renter) - Configured
- ✅ Click tracking disabled - Configured
- ✅ Frontend URL generation - Configured
- ✅ Duplicate prevention - Configured
- ✅ Fallback mechanisms - Configured


