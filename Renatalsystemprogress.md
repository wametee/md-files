# Rental Service System — Remaining Work

This document outlines the remaining work required to complete and productionize the rental service system. While core infrastructure is in place, several critical integrations, optimizations, and end-to-end flows are still pending.

---

## 1. M-Pesa B2C Payments for Rental Services (Partially Done)

### What Exists
- B2C service implementation: `mpesaService.initiateB2CPayment`
- B2C callback handler: `handleB2CCallback`
- Automatic B2C payout triggered in M-Pesa webhook on successful payment  
  *(mpesaWebhooks.js, lines 276–320)*
- Manual payout endpoint:  
  `POST /api/mpesa/driver-payout`

### What’s Missing

#### A. Automatic Payout on Rental Completion
- **Current behavior:**  
  B2C payout triggers only when payment succeeds (via webhook).
- **Missing:**
  - Automatic B2C payout when a rental is completed or checked out
  - Payout trigger if payment was already completed earlier
  - Payout handling for rentals paid via **Stripe** (currently only M-Pesa webhook triggers B2C)

#### B. M-Pesa B2C Refunds
- TODOs present in `penaltyService.js` (lines 69–72, 201–204)
- Refunds are marked in the database but **not sent via B2C**
- Required refund scenarios:
  - Owner cancellation
  - Renter no-show (partial refund)
  - Vehicle unavailable (full refund)

#### C. Payout Status Tracking
- No frontend UI for owners to see payout status
- No retry mechanism for failed B2C payouts
- No notification when payout completes or fails

---

## 2. Vehicle Location Tracking (Partially Done)

### What Exists
- Backend models: `LocationTracking`
- Backend services: `locationTrackingService`
- Backend controllers: `locationTracking.js`
- Frontend component: `RentalLocationTracker.jsx`
- Basic browser geolocation tracking

### What’s Missing

#### A. Real-time Socket.IO Integration
- Owners currently poll location every 10 seconds (not real-time)
- Missing:
  - Socket.IO events for location updates
  - Owners subscribing to `rental:${rentalId}` rooms

#### B. Location Update Optimization
- **Current:** Updates on every GPS change
- **Missing:**
  - Distance-based throttling (send only if moved >10m)
  - Time-based throttling (max once every 5 seconds)
  - Accuracy filtering (ignore readings with accuracy >50m)

#### C. Frontend Address Geocoding
- Backend performs reverse geocoding (slow)
- Missing:
  - Google Maps Geocoding API on frontend for instant address display

#### D. Battery Optimization
- Missing:
  - Adaptive update intervals (5s when moving, 30s when stationary)
  - Pause tracking when app is backgrounded

#### E. Offline Support
- Missing:
  - Queue location updates when offline
  - Sync queued updates when connection is restored

---

## 3. Additional Missing Features

### A. Rental Completion Payout Flow
- Automatic payout trigger on `checkoutBooking`
- Payout for completed rentals (not only on payment)
- Payout reconciliation (verify payout success before marking rental complete)

### B. M-Pesa Refund Implementation
- B2C refund for owner cancellations
- B2C refund for renter no-shows (90% refund)
- B2C refund for vehicle unavailable (100% refund)
- Refund status tracking and notifications

### C. Location Tracking Enhancements
- Real-time map updates for owners
- Location history analytics (distance traveled, speed, duration)
- Geofencing alerts (vehicle leaves allowed area)
- Location accuracy warnings

### D. Rental Workflow Gaps
- Automatic status transitions:
  - `pending → confirmed → active → completed`
- Reminder notifications:
  - Pickup time approaching
  - Return time approaching
- Late return penalty calculation
- Vehicle condition check-in / checkout photos

### E. Payment Reconciliation
- Automatic retry for failed B2C payouts
- Payout failure notifications to owners
- Payout history dashboard for owners
- Manual payout trigger from rental history page

### F. Integration Gaps
- Socket.IO events for rental status changes
- Real-time notifications for booking confirmations
- Real-time location sharing between renter and owner

---

## Priority Implementation Order

### Phase 1: Critical (Complete Rental Flow)
- Automatic B2C payout on rental completion
- M-Pesa B2C refund implementation
- Real-time Socket.IO location updates

### Phase 2: Important (User Experience)
- Location update optimization (throttling + accuracy filtering)
- Frontend address geocoding
- Payout status tracking and UI

### Phase 3: Enhancements (Polish & Scale)
- Battery optimization
- Offline location queue
- Geofencing alerts
- Rental status automation

---

## Summary

- **M-Pesa C2B:** ✅ Done  
- **M-Pesa B2C:** ⚠️ Partially done  
  - Missing completion flow integration and refunds
- **Vehicle Location Tracking:** ⚠️ Partially done  
  - Missing real-time updates and optimization

**Conclusion:**  
The core infrastructure exists. The remaining work focuses on integration, optimization, and completing end-to-end rental, payment, and tracking flows to achieve a production-ready system.
