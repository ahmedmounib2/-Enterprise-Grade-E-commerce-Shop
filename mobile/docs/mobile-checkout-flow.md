# Mobile Checkout Flow Wiring Guide

This document explains how the mobile checkout experience is wired so users can tap **Proceed to
checkout** on the cart screen, choose cash on delivery (COD) or card payments, and receive a success
or cancel outcome just like on the website. It also highlights the key screens, navigation hooks,
backend integration points, and translation updates that keep the experience consistent with the
existing UI theme.

## Overview of Screens

| Screen                | File                                        | Purpose                                                                                                                                           |
| --------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CartPage`            | `mobile/src/screens/CartPage.js`            | Hosts the **Proceed to checkout** button. Ensures the shopper is signed in and that the cart is not empty before continuing to the checkout flow. |
| `CheckoutPage`        | `mobile/src/screens/CheckoutPage.js`        | Collects shipping/billing details, lets the shopper choose between COD and card, and initiates the corresponding backend call.                    |
| `PurchaseSuccessPage` | `mobile/src/screens/PurchaseSuccessPage.js` | Confirms the order, fetches the latest order status (COD or Stripe), and clears the cart.                                                         |
| `PurchaseCancelPage`  | `mobile/src/screens/PurchaseCancelPage.js`  | Provides graceful cancellation messaging and routes the user back to their cart.                                                                  |

These screens all use the same theme palette (`useThemePalette`) and translation strings
(`react-i18next`) that the rest of the mobile app relies on, so the UX remains consistent across
light/dark modes and locales.

## Navigation Wiring

1. **Register the routes** in `mobile/src/navigation/AppNavigator.js`. The stack navigator exports
   routes named `Checkout`, `PurchaseSuccess`, and `PurchaseCancel` so the cart and deep links can
   reach them.
2. **Gate access from the cart** in `CartPage`:
   - If the user is not authenticated, an alert prompts them to sign in and sends them to the login
     screen.
   - If the cart is empty, an alert explains that checkout requires products.
   - Otherwise the screen calls `navigation.navigate('Checkout')`.
3. **Handle deep links** with the helper `extractCheckoutParams` inside `AppNavigator`. Stripe
   redirects back to `eshop://checkout-success?session_id=...` while COD orders can send
   `eshop://checkout-success?orderId=...&cod=true`. Cancellation uses `eshop://checkout-cancel`. The
   navigator queues navigation events until the container is ready, so tapping links outside the app
   still lands on the correct screen.

## Checkout Flow Details

### Card (Stripe) Checkout

1. `CheckoutPage` posts to `/payments/create-checkout-session` with calculated totals and the return
   deep-link URLs (`eshop://checkout-success` and `eshop://checkout-cancel`).
2. The backend (`backend/src/controllers/payment.controller.js`) validates the supplied URLs, passes
   them to Stripe, and returns the hosted checkout URL and the deep-link safe payload.
3. The app opens the `stripeSession.url` in the device browser. Stripe redirects to the supplied
   success/cancel URLs on completion.
4. `PurchaseSuccessPage` fetches the order via `/payments/checkout-success?session_id=...`, displays
   the summary, and clears the cart. If the query fails, the screen shows an error state with retry
   controls.

### Cash on Delivery Checkout

1. The shopper toggles COD in `CheckoutPage`. Instead of contacting Stripe, the screen posts to
   `/payments/cod-checkout`.
2. The backend creates the COD order and responds with an order identifier plus a success deep link
   payload.
3. The screen immediately navigates to `PurchaseSuccess` with `isCod: true` so the success page
   shows COD-specific messaging (`purchaseSuccess.codSteps`).

### Cancellation Handling

- Any failure that occurs before redirecting to Stripe surfaces as an error alert on `CheckoutPage`.
- Stripe cancellations or manual cancellations redirect to `eshop://checkout-cancel`, which
  `AppNavigator` maps to `PurchaseCancel` with friendly messaging and a button that routes back to
  the cart.

## Translation & Theme Notes

- All new copy lives under the `orderSummary`, `checkout`, `purchaseSuccess`, and `purchaseCancel`
  namespaces inside `shared/locales/*.json`. This ensures parity across EN/ES/FR/AR locales.
- Every new screen obtains colors from `useThemePalette`, so any future theme adjustments propagate
  automatically.

## Extensibility Checklist

When adding new payment methods or modifying the flow:

- Reuse the same navigation route names so deep-link handling stays intact.
- Extend translations in all locale files when adding new text.
- Update the backend to sanitize any new return URLs before forwarding them to Stripe or other
  processors.
- Keep address and contact validation logic in `CheckoutPage` in sync with the web checkout
  experience.
