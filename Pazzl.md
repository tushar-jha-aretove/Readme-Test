# Payment Flows – Code Map & Behaviour

This document describes **where** and **why** each payment flow works the way it does, with direct references to the codebase.

---

## Table of contents

1. [Overview: payment types and entry points](#1-overview-payment-types-and-entry-points)
2. [Backend API routes (Adyen)](#2-backend-api-routes-adyen)
3. [Order ID behaviour: PayPal vs others](#3-order-id-behaviour-paypal-vs-others)
4. [Flow A: Normal checkout (card, PayPal, Apple Pay in Drop-in)](#4-flow-a-normal-checkout)
5. [Flow B: PayPal Express (from cart)](#5-flow-b-paypal-express-from-cart)
6. [Flow C: Apple Pay Express (from cart)](#6-flow-c-apple-pay-express-from-cart)
7. [Flow D: Paystack (South Africa)](#7-flow-d-paystack-south-africa)
8. [Flow E: Gift card only](#8-flow-e-gift-card-only)
9. [Flow F: Redirects and payment details](#9-flow-f-redirects-and-payment-details)
10. [Quick reference: file → responsibility](#10-quick-reference-file--responsibility)

---

## 1. Overview: payment types and entry points

| Flow | Where it starts | Order created when? | Main backend | Thank-you |
|------|-----------------|----------------------|--------------|-----------|
| **Normal checkout** | Checkout → Payment step | **Before** first Adyen call (`createOrder(basketId)`) | `POST /api/adyen/payments` (else branch) | Drop-in `onPaymentCompleted` → `/checkout/confirmation/:orderNo` |
| **PayPal Express** | Cart (express buttons) | **After** first Adyen call: order number first, then `createOrder()` in payment-details | `POST /api/adyen/payments` (PayPal branch) then `POST /api/adyen/payments/details` | `handleAdditionalDetails` → `navigateToThankYouPage(merchantReference)` |
| **Apple Pay Express** | Cart (express buttons) | **Before** Adyen call (`createOrder(basketId)` in payments) | `POST /api/adyen/payments` (else branch) | `processPaymentCompleted` → `navigateToThankYouPage(orderNo)` |
| **Paystack** | Checkout → Place order | On Place order: `createOrder(basketId)` then Paystack | Checkout `createOrder` + Paystack | `onSuccess` → thank-you |
| **Gift card only** | Checkout → Place order | `POST /api/orders` (no Adyen) | Orders API | `navigateToThankYouPage(orderNo)` |

**Important:**  
- **PayPal Express** is the only flow that needs an **order number before the order exists** (reserved first, order created later in payment-details).  
- **Normal checkout** and **Apple Pay Express** create the **real order from the basket first**, then use that `orderNo` in the payment request.

---

## 2. Backend API routes (Adyen)

All Adyen-related routes are registered in:

**File:** `overrides/app/adyen/api/routes/index.js`

| Method | Path | Controller | Purpose |
|--------|------|------------|---------|
| GET | `/api/adyen/environment` | EnvironmentController | Client key, environment (used by all Adyen UIs) |
| GET | `/api/adyen/paymentMethods` | PaymentMethodsController | Available methods (card, PayPal, Apple Pay, etc.); can be called with `location=checkout` or `location=cart` |
| POST | `/api/adyen/payments` | PaymentsController | First payment call: create/reserve order, call Adyen Payments API |
| POST | `/api/adyen/payments/details` | PaymentsDetailsController | Follow-up after redirect or 3DS; for PayPal Express also runs `createOrder()` |
| POST | `/api/adyen/paypal/order` | PaypalOrderController | PayPal Express only: update PayPal order (shipping, address) via SFCC `updatesOrderForPaypalExpressCheckout` |
| POST | `/api/adyen/shipping-methods` | ShippingMethodsController | Get/update shipping methods (used by express flows) |
| POST | `/api/adyen/shipping-address` | ShippingAddressController | Update shipping address (used by express) |

**Checkout redirect/confirmation (render only):**

- `*/checkout/redirect` – where user lands after PayPal/3DS redirect; `redirectResult` in query.
- `*/checkout/confirmation/:orderNo` – thank-you page for normal/Apple Pay flow.

---

## 3. Order ID behaviour: PayPal vs others

### PayPal Express: order number **before** order exists

1. **First request** (`POST /api/adyen/payments` with `paymentMethod.type === 'paypal'` and `subtype === 'express'`):
   - **File:** `overrides/app/adyen/api/controllers/payments.js` (around 347–356).
   - Calls **`customApiClient.createOrderNumber()`** (Custom API: generate and assign order number to basket).
   - Sends to Adyen: `reference: generatedOrderNo.orderNo`.
   - **No** `shopperOrdersService.createOrder(basketId)` here.
2. **Later** (when user returns from PayPal and payment details are submitted):
   - **File:** `overrides/app/adyen/api/controllers/payments-details.js` (around 27–31).
   - If `paymentType === 'paypalexpress'`: calls **`customApiClient.createOrder()`** to create the actual order (using the already-generated order number).
   - Then calls Adyen `paymentsDetails` and updates order (payment method, PSP reference, etc.).

**Custom API (order number vs order create):**

- **File:** `overrides/app/adyen/api/controllers/custom-api.js`
  - `createOrderNumber()` (lines 83–94): only generates/reserves an order number (e.g. custom SFCC endpoint `order-generate`).
  - `createOrder()` (lines 105–115): creates the order (e.g. custom SFCC endpoint `order-create`) using that number.

### Apple Pay & normal checkout: order **before** payment

1. **Single request** (`POST /api/adyen/payments`, **else** branch – not PayPal Express):
   - **File:** `overrides/app/adyen/api/controllers/payments.js` (around 408–416).
   - Calls **`shopperOrdersService.createOrder(req.headers.basketid)`** → real order is created.
   - Payment request uses `reference: order.orderNo` (around 474).
   - No `createOrderNumber()` or delayed `createOrder()`.

So: **PayPal Express = order number first, order created later in payment-details.**  
**Apple Pay / normal = order created first, then its ID used in payment.**

---

## 4. Flow A: Normal checkout

User goes through: **Shipping address → Shipping options → Payment → Place order.**

### 4.1 Checkout steps and state

- **File:** `overrides/app/pages/checkout/context/checkout-context.tsx`
  - Steps: `SHIPPING_ADDRESS`, `SHIPPING_OPTIONS`, `PAYMENT`, `REVIEW_ORDER` (see `CHECKOUT_STEPS_LIST`).
  - Step index and `goToStep` drive which partial is shown.

### 4.2 Where the payment step is rendered

- **File:** `overrides/app/pages/checkout/index.jsx`
  - Renders `ShippingAddressStep`, `ShippingOptionsStep`, and **`Payment`** based on `step`.
- **File:** `overrides/app/pages/checkout/partials/payment.jsx`
  - If `paymentProvider === PAYMENT_PROVIDERS.ADYEN`: wraps content in `AdyenCheckoutProvider` and renders **`AdyenPayment`** (around 282–299).
  - If Paystack: renders `PaystackPayment`.

### 4.3 Adyen payment component (Drop-in)

- **File:** `overrides/app/pages/checkout/partials/adyen-payment.jsx`
  - Renders gift card block (if applicable) and **`AdyenCheckout`** with basket, handlers, etc.
- **File:** `overrides/app/adyen/components/adyenCheckout.jsx`
  - Uses **Adyen Drop-in** with: Card, Klarna, Swish, **ApplePay**, **PayPal**, Bancontact, BcmcMobile, Blik (lines 69–77).
  - On load: `handleQueryParams` → if no `redirectResult`/`amazonCheckoutSessionId`/`adyenAction`, creates **`new Dropin(checkout, { paymentMethodComponents: [...], paymentMethodsConfiguration, ... })`** and mounts it (lines 64–86).
  - `paymentMethodsConfiguration` comes from **`overrides/app/adyen/components/paymentMethodsConfiguration.js`** (card, paypal, applepay, klarna, giftcard, etc.).

### 4.4 Where the first payment request is sent (normal flow)

- Drop-in’s **onSubmit** is wired via config from `getPaymentMethodsConfiguration` in **`overrides/app/adyen/contexts/adyen-checkout-context.jsx`** (e.g. around 77–95).
- Base submit behaviour is in **`overrides/app/adyen/components/helpers/baseConfig.js`**:
  - **`onSubmit`** (lines 34–66): calls **`AdyenPaymentsService.submitPayment(state.data, basketId, customerId, locale)`**.
- **File:** `overrides/app/adyen/services/payments.js`
  - `submitPayment` → **POST `/api/adyen/payments`** with `data`, headers `basketid`, `customerid`, query `locale` (and site from API client).

### 4.5 What the backend does (normal flow, not PayPal Express)

- **File:** `overrides/app/adyen/api/controllers/payments.js`
  - Validates request, gets basket (around 276–316).
  - If no Adyen payment instrument in basket, adds one (around 318–334).
  - **Not** PayPal Express → goes to **else** (around 408+):
    - **`order = await shopperOrdersService.createOrder(req.headers.basketid)`** (line 416).
    - Builds `paymentRequest` with `reference: order.orderNo`, amounts, addresses from **order** (around 467–493).
    - Calls **Adyen** `PaymentsApi.payments(paymentRequest)`.
    - On success: updates order (payment method, PSP ref) via **OrderApiClient** (around 528–537).
    - Gift card: may call `redeemStrategy.redeemLater(order.orderNo)` (around 545–550).
  - Response is shaped by **`createCheckoutResponse`** (e.g. `overrides/app/adyen/utils/createCheckoutResponse.js`).

### 4.6 Response handling and thank-you (normal flow)

- **baseConfig.js** `onSubmit`: after `submitPayment` returns, **`props?.setOrderNo?.(paymentsResponse.merchantReference)`** (line 62); then `actions.resolve(paymentsResponse)`.
- **adyenCheckout.jsx**: in checkout config, **`onPaymentCompleted: (result, component) => { props?.onPaymentCompleted?.(); legacyNavigator.navigateToThankYouPage(getOrderNo()) }`** (around 153–156).
- Thank-you URL: **`/checkout/thankyou?orderNo=...`** (see **`overrides/app/utils/navigator.js`** `legacyUrls.getThankYouPageUrl`).

### 4.7 3DS / redirect (normal flow)

- If Adyen returns a redirect (e.g. 3DS), user goes to **`/checkout/redirect?redirectResult=...`** (returnUrl in payment request, e.g. line 369 in payments.js).
- **File:** `overrides/app/pages/checkout-redirect/checkout-redirect-page.tsx` (and route in **routes.jsx**): page loads, can pass `redirectResult` into Adyen/checkout logic.
- **File:** `overrides/app/adyen/components/adyenCheckout.jsx`: when **`redirectResult`** is in URL params, **`checkout.submitDetails({ data: { details: { redirectResult } } })`** (lines 52–54) → that triggers a **payment-details** call and then completion/cancel handling.

### 4.8 Payment details (normal flow, after redirect)

- **File:** `overrides/app/adyen/components/helpers/baseConfig.js`
  - **`onAdditionalDetails`** (lines 67–78): calls **`AdyenPaymentsDetailsService.submitPaymentsDetails(state.data, customerId)`** (no `paymentType` set like PayPal).
- **File:** `overrides/app/adyen/services/payments-details.js`
  - **POST `/api/adyen/payments/details`** with `data` and `paymentType`.
- **File:** `overrides/app/adyen/api/controllers/payments-details.js`
  - For **non–PayPal Express** (`paymentType !== 'paypalexpress'`): **does not** call `createOrder()`; order already exists.
  - Calls Adyen **`paymentsDetails`**, then **OrderApiClient.updateOrder(merchantReference, ...)** (around 54–60).
  - Gift card redeem may run (around 79–87).
  - Response goes back to frontend; Drop-in completes and can trigger thank-you.

**Summary normal flow:**  
Checkout steps → Payment step → Drop-in → **onSubmit** → **POST /api/adyen/payments** → backend **createOrder(basketId)** → Adyen payments with `reference: order.orderNo` → success → **setOrderNo(merchantReference)** → **onPaymentCompleted** → thank-you. Redirect path: **redirect** page → **submitDetails(redirectResult)** → **POST /api/adyen/payments/details** (no `createOrder`) → complete.

---

## 5. Flow B: PayPal Express (from cart)

User is on **cart**; clicks **PayPal** button; address/shipping and payment happen in PayPal’s UI; then return to site and thank-you.

### 5.1 Where the button is shown

- **File:** `overrides/app/pages/cart/partials/cart-cta.tsx`
  - Renders **`ExpressPaymentContainer`** (around 43).
- **File:** `overrides/app/pages/cart/partials/cart-express-payment.tsx`
  - Uses **`AdyenExpressCheckoutProvider`** (cart context: basket, auth, **location: CART** for payment methods).
  - Renders **`PaypalExpress`** and **`ApplePayExpress`** (lines 54–55).

### 5.2 PayPal Express UI and config

- **File:** `overrides/app/adyen/components/paypal-express/paypal-express.jsx`
  - Uses **`useAdyenExpressCheckout()`** for: `adyenEnvironment`, `adyenPaymentMethods`, `basket`, `locale`, `site`, `authToken`, `shippingMethods`.
  - Creates **`PaypalExpressService`** with basketId, shippingMethodId, amounts, etc. (lines 55–68).
  - **AdyenCheckout** created with `paymentMethodsResponse: adyenPaymentMethods`, **amount: basket.c_productsTotalInCents** (no order yet).
  - **new PayPal(checkout, config)** with:
    - **onSubmit** → `paypalService.handleSubmit`
    - **onShippingAddressChange** → `paypalService.handleShippingAddressChange`
    - **onShippingOptionsChange** → `paypalService.handleShippingOptionsChange`
    - **onAuthorized** → `paypalService.handleAuthorized`
    - **onAdditionalDetails** → `paypalService.handleAdditionalDetails`

### 5.3 First payment call (PayPal Express)

- **File:** `overrides/app/adyen/components/paypal-express/paypal-express-service.js`
  - **`handleSubmit(state, component, actions)`** (lines 61–84):
    - Calls **`AdyenPaymentsService.submitPayment(state.data, basketId, customerId, locale)`** → **POST `/api/adyen/payments`**.
    - State includes payment method from Adyen (PayPal express); backend receives **`data.paymentMethod.type === 'paypal'`** and **`data.paymentMethod.subtype === 'express'`**.

**Backend (PayPal branch):**

- **File:** `overrides/app/adyen/api/controllers/payments.js` (lines 347–406)
  - **`customApiClient.createOrderNumber()`** → only order number generated and stored (e.g. on basket).
  - **No** `shopperOrdersService.createOrder(basketId)`.
  - Builds **paymentRequest** with **`reference: generatedOrderNo.orderNo`**, amount from **basket** (e.g. `basket.c_productsTotalInCents`).
  - **checkoutApi.PaymentsApi.payments(paymentRequest)** → Adyen may return redirect to PayPal.
  - Response with `orderNo: generatedOrderNo.orderNo` (in checkoutResponse) returned to frontend.

So: **order number exists; order record does not yet.**

### 5.4 Updating PayPal order (address/shipping in PayPal popup)

- When user changes address or shipping in PayPal UI, **onShippingAddressChange** / **onShippingOptionsChange** call **`this.#orderService.updatePayPalOrder(patch)`**.
- **File:** `overrides/app/adyen/components/paypal-express/paypal-express-service.js` (e.g. 86–113, 118–151).
- **File:** `overrides/app/adyen/services/paypal-order.ts`
  - **AdyenPaypalOrderService.updatePayPalOrder(orderData)** → **POST `/api/adyen/paypal/order`** with `siteId`, body `{ data: orderData }`.
- **File:** `overrides/app/adyen/api/controllers/paypal-order.js`
  - Calls SFCC **`checkoutApi.UtilityApi.updatesOrderForPaypalExpressCheckout(data, ...)`** to sync PayPal’s order (amounts, shipping, address).

### 5.5 After user approves in PayPal: authorized and additional details

- **handleAuthorized** (paypal-express-service.js 155–181): updates **basket** shipping/billing/customer from PayPal data via **ShopperBasketsService** (no order creation).
- When Adyen needs more details (e.g. after redirect back), **handleAdditionalDetails** (lines 183–199):
  - **AdyenPaymentsDetailsService.submitPaymentsDetails(state.data, customerId, PAYMENT_METHOD_TYPE)** with **`paymentType: 'paypalexpress'`**.
  - **File:** `overrides/app/adyen/services/payments-details.js`: POST **`/api/adyen/payments/details`** with `data` and **`paymentType: 'paypalexpress'`**.

**Backend payment-details (PayPal Express):**

- **File:** `overrides/app/adyen/api/controllers/payments-details.js` (lines 27–31)
  - If **`paymentType == 'paypalexpress'`**: **`await customApiClient.createOrder()`** → **actual order is created** (using the reserved order number).
  - Then **checkoutApi.PaymentsApi.paymentsDetails(data)**.
  - **OrderApiClient.updateOrder(response.merchantReference, ...)** (lines 54–60).
  - Gift card redeem if needed (79–87).
  - Response (with `merchantReference`) returned.

### 5.6 Thank-you (PayPal Express)

- **paypal-express-service.js** **handleAdditionalDetails**: after **submitPaymentsDetails** returns, **`legacyNavigator.navigateToThankYouPage(paymentsResponse.merchantReference)`** (line 195).

**Summary PayPal Express:**  
Cart → **PayPal** button → **onSubmit** → **POST /api/adyen/payments** (PayPal branch) → **createOrderNumber()** only → Adyen with `reference: orderNo` → user in PayPal (address/shipping updates via **POST /api/adyen/paypal/order**) → **onAuthorized** (basket updated) → **onAdditionalDetails** → **POST /api/adyen/payments/details** with **paymentType: 'paypalexpress'** → **createOrder()** → Adyen paymentDetails + order update → thank-you with **merchantReference**.

---

## 6. Flow C: Apple Pay Express (from cart)

User is on **cart**; clicks **Apple Pay**; address/shipping in Apple Pay sheet; payment; then thank-you.

### 6.1 Where the button is shown

- Same as PayPal: **cart-express-payment.tsx** inside **AdyenExpressCheckoutProvider**, rendering **`ApplePayExpress`** (line 55).

### 6.2 Apple Pay Express UI and config

- **File:** `overrides/app/adyen/components/apple-pay/apple-pay-express.jsx`
  - **useAdyenExpressCheckout()** for environment, payment methods, basket, shipping methods.
  - **getApplePayExpressCheckout** (apple-pay-express-checkout.js): creates Adyen Checkout instance for Apple Pay.
  - **getApplePayExpressConfig** (apple-pay-express-config.js): returns config with:
    - **onShippingContactSelected** → **ApplePayExpressService.processShippingContactSelected**
    - **onShippingMethodSelected** → **ApplePayExpressService.processShippingMethodSelected**
    - **onPaymentAuthorized** → **ApplePayExpressService.processAuthorized**
    - **onSubmit** → **ApplePayExpressService.processSubmit**
    - **onComplete** → **ApplePayExpressService.processPaymentCompleted**
  - **new ApplePay(checkout, config)** mounted on container.

### 6.3 Address/shipping in Apple Pay sheet

- **File:** `overrides/app/adyen/components/apple-pay/apple-pay-express-service.js`
  - **processShippingContactSelected** (59–84): validates country/postal (e.g. blocked postal codes); resolves/rejects with **newTotal**.
  - **processShippingMethodSelected** (98–122): finds shipping method, updates basket shipment via **ShopperBasketsService**, returns new total for Apple Pay.

### 6.4 Payment authorized and basket update

- **processAuthorized** (133–164): maps Apple Pay **billingContact** / **shippingContact** to addresses; updates **basket** shipping and billing and customer email via **ShopperBasketsService**; **actions.resolve()** (no order creation here).

### 6.5 Submit payment (first and only payment call)

- **processSubmit** (170–196):
  - **AdyenPaymentsService.submitPayment(state.data, basketId, customerId, locale)** → **POST `/api/adyen/payments`**.
  - Request does **not** send `paymentMethod.type === 'paypal'` with `subtype === 'express'`, so backend takes the **else** branch.

**Backend (else branch):**

- **payments.js** (408+): **`order = await shopperOrdersService.createOrder(req.headers.basketid)`** → **order created first**.
  - Builds paymentRequest with **`reference: order.orderNo`**, order addresses, etc.
  - **PaymentsApi.payments(paymentRequest)**.
  - Response includes **merchantReference** (= order.orderNo).

### 6.6 Thank-you (Apple Pay Express)

- **processSubmit** sets **`this.#orderNo = merchantReference`** (line 190).
- **processPaymentCompleted** (199–201): **`legacyNavigator.navigateToThankYouPage(this.#orderNo)`**.

**Summary Apple Pay Express:**  
Cart → **Apple Pay** → contact/shipping in sheet (basket updated) → **processAuthorized** (basket again) → **processSubmit** → **POST /api/adyen/payments** (else) → **createOrder(basketId)** → Adyen with `reference: order.orderNo` → **merchantReference** returned → **processPaymentCompleted** → thank-you. **No** payment-details step in this path; **no** `createOrderNumber()` or delayed `createOrder()`.

---

## 7. Flow D: Paystack (South Africa)

Used when **site uses Paystack** (e.g. `paulaschoice_sa` in **config/sites.js** with `payment.provider: 'paystack'`).

### 7.1 Where it runs

- **Checkout** → Payment step shows **PaystackPayment** (payment.jsx when `paymentProvider !== PAYMENT_PROVIDERS.ADYEN`).
- **Place order** in **overrides/app/pages/checkout/index.jsx** (lines 195–274): if **Paystack**, then:
  - Add/update Paystack payment instrument on basket.
  - **createOrder({ body: { basketId } })** → order created via SLAS/Orders API (not Adyen).
  - Initialize Paystack with **reference: order.orderNo** (line 272).
  - **onSuccess** / **onClose** handle success/failure; success typically navigates to thank-you with **order.orderNo**.

No Adyen payments or payment-details; order is created at place-order time.

---

## 8. Flow E: Gift card only

When **order remaining is 0** (e.g. full gift card) and user places order:

- **File:** `overrides/app/pages/checkout/index.jsx` (lines 275–316)
  - **if (isNoOrderRemaining)**:
    - **axios.post('/api/orders', { basketId }, { params: { siteId, locale }, headers: { authorization } })** (lines 308–314).
    - **legacyNavigator.navigateToThankYouPage(response.data.orderNo)** (line 311).
  - No Adyen; no payment step submit.

---

## 9. Flow F: Redirects and payment details

### 9.1 Return URL and redirect page

- Payment requests (payments.js) set **returnUrl** to **`${origin}/checkout/redirect`** (e.g. 369, 487).
- Route: **`*/checkout/redirect`** (adyen api/routes/index.js 72–76) → **runtime.render** (checkout redirect page).
- **File:** `overrides/app/pages/checkout-redirect/checkout-redirect-page.tsx`: reads **`redirectResult`** from query; can pass it into checkout/component that calls **submitDetails({ details: { redirectResult } })** so that **POST /api/adyen/payments/details** is triggered.

### 9.2 When payment-details is used

- **Normal flow:** After 3DS or other redirect, frontend calls **submitDetails** → **onAdditionalDetails** in baseConfig → **POST /api/adyen/payments/details** (no `paymentType` or non-express). Backend does **not** call `createOrder()`; order already exists.
- **PayPal Express:** After user returns from PayPal, Adyen may require **additionalDetails**. Frontend calls **handleAdditionalDetails** with **paymentType: 'paypalexpress'** → **POST /api/adyen/payments/details** → backend runs **createOrder()** then **paymentsDetails** and order update.

### 9.3 Confirmation/thank-you URL

- **navigator.js**: **getThankYouPageUrl(orderNo)** → **`/checkout/thankyou?orderNo=${orderNo}`** (legacy URL).
- Routes in **routes.jsx**: **`/checkout/confirmation/:orderNo`** for confirmation view (e.g. **overrides/app/pages/checkout/confirmation.jsx**).

---

## 10. Quick reference: file → responsibility

| File | Responsibility |
|------|----------------|
| **overrides/app/adyen/api/controllers/payments.js** | Main payment handler: PayPal branch = createOrderNumber + Adyen; else = createOrder(basketId) + Adyen. |
| **overrides/app/adyen/api/controllers/payments-details.js** | Payment details + for PayPal Express only: createOrder() then paymentsDetails and order update. |
| **overrides/app/adyen/api/controllers/custom-api.js** | createOrderNumber() (reserve ID), createOrder() (create order using it). |
| **overrides/app/adyen/api/controllers/paypal-order.js** | Proxy to SFCC updatesOrderForPaypalExpressCheckout (address/shipping in PayPal). |
| **overrides/app/adyen/api/routes/index.js** | Registers all Adyen and checkout redirect/confirmation routes. |
| **overrides/app/adyen/contexts/adyen-checkout-context.jsx** | Checkout payment step: env, payment methods (location CHECKOUT), getPaymentMethodsConfiguration. |
| **overrides/app/adyen/contexts/adyen-express-checkout-context.jsx** | Cart express: env, payment methods (location CART), shipping methods for PayPal/Apple Pay. |
| **overrides/app/adyen/components/adyenCheckout.jsx** | Drop-in mount, handleQueryParams (redirectResult, etc.), onPaymentCompleted → thank-you. |
| **overrides/app/adyen/components/helpers/baseConfig.js** | onSubmit → submitPayment, setOrderNo; onAdditionalDetails → submitPaymentsDetails. |
| **overrides/app/adyen/components/paymentMethodsConfiguration.js** | Per-method config (card, paypal, applepay, klarna, giftcard, …). |
| **overrides/app/adyen/components/paypal-express/paypal-express.jsx** | Mounts PayPal component; wires handleSubmit, handleAuthorized, handleAdditionalDetails, etc. |
| **overrides/app/adyen/components/paypal-express/paypal-express-service.js** | handleSubmit → POST payments; handleAdditionalDetails → POST payments/details (paypalexpress) → thank-you; updatePayPalOrder for address/shipping. |
| **overrides/app/adyen/components/apple-pay/apple-pay-express.jsx** | Mounts Apple Pay; uses ApplePayExpressService for contact/shipping/submit/complete. |
| **overrides/app/adyen/components/apple-pay/apple-pay-express-service.js** | processAuthorized (basket update); processSubmit → POST payments (order created in backend) → merchantReference → processPaymentCompleted → thank-you. |
| **overrides/app/adyen/services/payments.js** | Frontend: submitPayment → POST /api/adyen/payments. |
| **overrides/app/adyen/services/payments-details.js** | Frontend: submitPaymentsDetails → POST /api/adyen/payments/details (body: data, paymentType). |
| **overrides/app/adyen/services/paypal-order.ts** | Frontend: updatePayPalOrder → POST /api/adyen/paypal/order. |
| **overrides/app/pages/checkout/partials/payment.jsx** | Renders AdyenPayment (Adyen) or PaystackPayment; step visibility. |
| **overrides/app/pages/checkout/partials/adyen-payment.jsx** | Wraps AdyenCheckout with gift card; passes handlers to AdyenCheckout. |
| **overrides/app/pages/cart/partials/cart-express-payment.tsx** | Renders PaypalExpress and ApplePayExpress inside AdyenExpressCheckoutProvider. |
| **overrides/app/pages/checkout/index.jsx** | Checkout steps; submitOrder: Paystack vs Adyen (paymentRef.handleAdyenSubmit) vs gift-card-only (POST /api/orders). |
| **overrides/app/utils/navigator.js** | legacyNavigator.navigateToThankYouPage(orderNo) → /checkout/thankyou?orderNo=... |

This should give you a single place to see **why** each flow behaves as it does and **where** in the code each step is implemented. For order ID behaviour, the crucial distinction remains: **PayPal Express = order number first, order created in payment-details; all others (normal + Apple Pay Express) = order created in payments controller before sending reference to Adyen.**
