# src/api — Complete File and Function Reference

This document describes **every file and every function** in `pc-composable-storefront/src/api` in detail. The directory is the **BFF (Backend for Frontend)** layer: Express routes, SFCC/Commerce API integration, Emarsys, Paazl, caching, logging, and shared utilities.

---

## Directory structure

```
src/api
├── main.js                    # Registers routes and global error handler
├── middlewares/
│   ├── index.ts               # Re-exports middlewares
│   └── validate-request.middleware.ts
├── utils/
│   ├── index.js               # Re-exports crypto, parse, general, errors
│   ├── constants.ts           # Shared constants (not in index)
│   ├── crypto.js
│   ├── errors.ts
│   ├── general.ts
│   └── parse.js
├── types/
│   ├── index.ts               # Re-exports sfcc types
│   ├── common.ts
│   └── sfcc.ts
└── modules/
    ├── cache/cache-manager.ts
    ├── logger/
    │   ├── index.ts
    │   └── logger.service.ts
    ├── emarsys/               # Newsletter/subscriber integration
    │   ├── emarsys.constants.js
    │   ├── emarsys.api.service.js
    │   ├── emarsys.service.js
    │   ├── emarsys.utils.js
    │   ├── emarsys.subscriber.service.js
    │   ├── emarsys.controller.js
    │   └── emarsys.utils.test.js
    ├── paazl/                  # Shipping (Paazl) integration
    │   ├── index.ts
    │   ├── paazl.api.service.ts
    │   └── paazl.service.ts
    └── sfcc/                   # Salesforce Commerce Cloud
        ├── sfcc.controller.ts
        ├── sfcc-admin-auth.service.ts
        ├── sfcc-admin.service.js
        ├── admin-order.service.ts
        ├── shopper-baskets.service.ts
        ├── shopper-orders.service.ts
        ├── shipping-methods.service.ts
        ├── place-order.service.ts
        ├── cancel-order.service.ts
        ├── cancel-order.service.spec.ts
        ├── gift-card.service.ts
        ├── paazl-preferences.service.ts
        ├── paazl-delivery-info.service.ts
        ├── global-custom-preferences.service.ts
        └── global-custom-preferences.service.test.ts
```

---

## 1. main.js

**Role:** Entry point that mounts API routers and a global error handler.

### `registerCustomApiRoutes(app)`

- **Parameters:** `app` — Express application instance.
- **Behavior:**
  - Mounts `emarsysController` at `/api/emarsys`.
  - Mounts `sfccController` at `/api` (so SFCC routes are under `/api/...`).
  - Registers a final error middleware that:
    - Logs the error via `logger.error(err)`.
    - If `err` is `BadRequestError` → responds with `400` and `{ error: err.message }`.
    - If `err` is `EmarsysError` → responds with `400` and `{ error: err.message, type: err.errorType }`.
    - Otherwise → responds with `500` and `{ error: 'Internal server error' }`.
- **Usage:** Called from the PWA Kit runtime (e.g. `overrides`) when setting up the Express app.

---

## 2. middlewares/index.ts

Re-exports everything from `validate-request.middleware.ts` (so you can import from `../../middlewares`).

---

## 3. middlewares/validate-request.middleware.ts

Express middleware that runs **after** `express-validator` and returns 400 if validation failed.

### `validatorMiddleware(req, res, next)`

- **Parameters:** Express `Request`, `Response`, `NextFunction`.
- **Behavior:** Uses `validationResult(req)` from `express-validator`. If there are validation errors, responds with `400` and `{ errors: errors.array() }`. Otherwise calls `next()`.
- **Usage:** Used in SFCC controller on routes that have `body()`, `query()`, `param()`, `header()` validators.

---

## 4. utils/constants.ts

Shared constants used across the BFF (and sometimes from overrides). **Not** re-exported from `utils/index.js`; imported directly as `../../utils/constants` or similar.

- **`CUSTOM_PREFERENCES_SCOPE`** — `{ PAAZL: 'Paazl' }`. Used for SFCC custom preference group IDs.
- **`COUNTRY_CODES`** (enum) — `DE`, `ES`. Used for Emarsys country-specific logic (e.g. opt-in).
- **`EMARSYS_ERROR_TYPES`** — `{ ACCOUNT_EXISTS: 'AccountExists' }`. Returned in Emarsys error responses.
- **`PAYMENT_LOCATIONS`** (enum) — `CHECKOUT = 'checkout'`, `CART = 'cart'`. Used in global preferences (e.g. PayPal/Apple Pay per location).

---

## 5. utils/errors.ts

Custom error class for 400 responses.

### `BadRequestError`

- **Class** extending `Error`. Sets `name = 'BadRequestError'`.
- **Usage:** Thrown when basket is invalid, not found, or gift card redeem fails; caught in `main.js` and returned as 400.

---

## 6. utils/general.ts

Generic object-path helper.

### `getIn(obj, path)`

- **Parameters:** `obj` — any value; `path` — dot-separated string (e.g. `'ApplePay.applepayconfiguration'`).
- **Returns:** Value at that path, or `undefined` if `obj` is not an object, path is not a string, or path is invalid.
- **Implementation:** Splits `path` by `'.'` and reduces over keys. Used in global preferences to read nested config.

---

## 7. utils/parse.js

Safe JSON parsing with fallback and optional logging.

### `tryParseJSON(value, fallback, logger)`

- **Parameters:** `value` — string to parse; `fallback` — default `null`; `logger` — optional object with `logger.error`.
- **Returns:** Parsed object or `fallback` on parse error. Logs error if `logger` is provided.
- **Usage:** Used for env/config that is JSON (e.g. Power Reviews config, Paazl options, Emarsys, feature toggles).

---

## 8. utils/crypto.js

Crypto helpers for auth headers.

### `getBase64Sha1(str)`

- **Parameters:** `str` — string to hash.
- **Returns:** SHA-1 hash of `str` as Base64 (hex digest converted to Base64).
- **Usage:** Emarsys WSSE header digest: `getBase64Sha1(nonce + timestamp + EMARSYS_SECRET)`.

### `createAuthBasicHeader(username, password)`

- **Parameters:** `username`, `password` — strings.
- **Returns:** `'Basic ' + Base64(username:password)`.
- **Usage:** SFCC admin OAuth client credentials (Basic auth for token endpoint).

---

## 9. utils/index.js

Re-exports: `crypto` (getBase64Sha1, createAuthBasicHeader), `parse` (tryParseJSON), `general` (getIn), `errors` (BadRequestError). Does **not** export `constants`.

---

## 10. types/common.ts

Generic type helper.

### `ObjectValues<T>`

- **Type:** `T[keyof T]` — union of all values of object type `T`. Used in `sfcc.ts` to derive union types from const objects.

---

## 11. types/sfcc.ts

SFCC/order and payment types and constants.

- **`SfccServiceFactoryParams`** — `{ siteId, locale, shopperToken }`. Used when creating Shopper Baskets/Orders services.
- **`PAYMENT_METHOD_IDS`** — `CASH`, `GIFT_CERTIFICATE`. Used for adding cash payment when order is fully discounted and for filtering gift certificates on cancel.
- **`PaymentMethodId`** — union from `PAYMENT_METHOD_IDS`.
- **`PAYMENT_STATUS`** — `PAID`, `PART_PAID`, `NOT_PAID`.
- **`EXPORT_STATUS`** — `READY`, `FAILED`, `EXPORTED`, `NOT_EXPORTED`.
- **`CONFIRMATION_STATUS`** — `CONFIRMED`, `NOT_CONFIRMED`.
- **`ORDER_STATUS`** — `CREATED`, `NEW`, `COMPLETED`, `CANCELLED`, `FAILED`, `FAILED_REOPEN`.
- Corresponding `*Status` / `PaymentMethodId` types are `ObjectValues<typeof ...>`.

---

## 12. types/index.ts

Re-exports everything from `./sfcc` only (common is used by sfcc, not exported from api types).

---

## 13. modules/logger/logger.service.ts

Simple logger that prefixes all messages with `[COMPOSABLE][LEVEL]:` and forwards to `console`.

### Class `LoggerService`

- **`info(payload)`** — `console.info` with composed message.
- **`warn(payload)`** — `console.warn` with composed message.
- **`error(payload)`** — `console.error` with composed message.
- **`debug(payload)`** — `console.debug` with composed message.

**Payload:** Can be `string`, `Error`, or `Record<string, unknown>`. For `Error`, it serializes `message`, `stack`, and optional `response.data`, `context`, `metadata`. For objects, JSON.stringify. Private helpers: `composeLog`, `getLogMessage`, `trySerialize`.

---

## 14. modules/logger/index.ts

Exports `LoggerService` and a singleton **`logger`** instance (`new LoggerService()`). The rest of the BFF imports `logger` from here.

---

## 15. modules/cache/cache-manager.ts

Singleton in-memory cache (NodeCache) with 30-minute TTL.

### `getCacheManager()`

- **Returns:** A single `NodeCache` instance (`stdTTL: 60 * 30`). Creates it on first call.
- **Usage:** SFCC admin token caching (`sfcc-admin-auth.service.ts`) and global custom preferences caching (`global-custom-preferences.service.ts`).

---

## 16. modules/emarsys/emarsys.constants.js

Emarsys API and domain constants.

- **`EMARSYS_API_URL`** — `'https://suite11.emarsys.net/api/v2/'`.
- **`EMARSYS_AUTH_HEADER_NAME`** — `'X-WSSE'`.
- **`EMARSYS_FIELD_IDS`** — Map of Emarsys field names to numeric IDs (firstName, lastName, email, optIn, countryCode, languageTag, etc.).
- **`EMARSYS_COUNTRY_CODES`** — Map of country codes to Emarsys IDs: `US: 185`, `UK: 184`, `DE: 65`, `FR: 61`, `ZA: 161`.
- **`EMARSYS_ERROR_CODES`** — `CONTACT_NOT_FOUND: 2008`.
- **`EMARSYS_FORM_NAMES`** — `CHECKOUT`, `ACCOUNT`, `Footer`.
- **`EMARSYS_OPT_IN_VALUE`** — `EMPTY: null`, `TRUE: '1'`, `FALSE: '2'`.
- **`EMARSYS_OPT_IN_STATE`** — string enum `EMPTY`, `TRUE`, `FALSE` for internal logic.

---

## 17. modules/emarsys/emarsys.api.service.js

Low-level HTTP client for Emarsys API with WSSE authentication.

### Class `EmarsysApiService`

- **Constructor:** Creates axios instance with `baseURL: EMARSYS_API_URL`.
- **`get(url)`** — GET with WSSE header.
- **`post(url, data)`** — POST with WSSE header.
- **`put(url, data, options)`** — PUT with WSSE header (default `options = {}`).
- **`#getHeaders()`** — Returns `{ 'X-WSSE': this.#getWsseHeader() }`.
- **`#getWsseHeader()`** — Builds WSSE UsernameToken: random nonce, ISO timestamp, digest = Base64(SHA1(nonce + timestamp + EMARSYS_SECRET)). Uses `process.env.EMARSYS_USER` and `process.env.EMARSYS_SECRET`.

---

## 18. modules/emarsys/emarsys.service.js

Business logic for contacts and forms.

### Class `EmarsysService`

- **Constructor:** Creates internal `EmarsysApiService`.
- **`getContact(email)`** — POSTs to `contact/getdata` with keyId=email, keyValues=[email], fields=[email, optIn]. If errors contain `CONTACT_NOT_FOUND`, returns `null`. Otherwise throws. Returns first result row (contact data keyed by field ID).
- **`createOrUpdateAccount(contactData)`** — PUTs to `contact/` with `create_if_not_exists: 1`, key_id=email. Used to create or update contact.
- **`getForms()`** — GETs `form`, returns `response.data.data` (list of forms).
- **`submitForm(formId, email)`** — POSTs to `form/{formId}/trigger` with key_id=email, external_id=email. Used to trigger form (e.g. checkout/newsletter).

---

## 19. modules/emarsys/emarsys.utils.js

Opt-in logic and contact payload building.

### `EmarsysError`

- **Class** extending Error. `name = 'EmarsysError'`, `errorType` set in constructor. Used for “already subscribed” (ACCOUNT_EXISTS).

### `getOptInState({ accountExists, optIn, countryCode, checkboxState })`

- **Returns:** One of `EMARSYS_OPT_IN_STATE.TRUE`, `FALSE`, `EMPTY`, or `null` depending on:
  - Whether contact exists, checkbox state, and country (e.g. DE gets EMPTY in some cases for double opt-in).
- **Logic (summary):** Existing account + checkbox checked → TRUE (or for DE with optIn FALSE → EMPTY). New account + checkbox unchecked → FALSE; checked → TRUE or EMPTY for DE. Otherwise null.

### `getContactData({ address, accountExists, optIn, checkboxState })`

- **Returns:** Object keyed by Emarsys field IDs, built from `address`. Maps address keys to `EMARSYS_FIELD_IDS`; converts countryCode to `EMARSYS_COUNTRY_CODES`. Adds optIn field using `getOptInState`. Only includes non-null, non-empty values.

---

## 20. modules/emarsys/emarsys.subscriber.service.js

Orchestrates “subscribe” flow: get contact, validate, update contact, submit form.

### Class `EmarsysSubscriberService`

- **Constructor:** Creates internal `EmarsysService`.
- **`createOrUpdateSubscriber(data)`**  
  - Gets contact by `data.email`.  
  - If form is Footer and contact exists with optIn TRUE → throws `EmarsysError` with `ACCOUNT_EXISTS`.  
  - Builds `contactData` via `getContactData({ address, accountExists, optIn, checkboxState })`.  
  - Calls `createOrUpdateAccount(contactData)`.  
  - Fetches forms, finds form by name (from `data.formName`; code checks for `EMARSYS_FORM_NAMES.CHECKOUT` in error message but form name is passed in — typically Checkout, Account, or Footer).  
  - Calls `submitForm(checkoutFormId, data.email)`.  
  - No return; caller expects success.

---

## 21. modules/emarsys/emarsys.controller.js

Express router for Emarsys.

- **POST `/subscribers`** — Body: subscriber data (email, firstName, lastName, countryCode, checkboxState, formName). Creates `EmarsysSubscriberService`, calls `createOrUpdateSubscriber(req.body)`. On success: `200` and `{ success: true }`. Errors passed to `next(e)` (handled in main.js).

---

## 22. modules/emarsys/emarsys.utils.test.js

Unit tests for `getOptInState` only: account exists/does not exist, checkbox checked/unchecked, DE vs ES, expected `EMARSYS_OPT_IN_STATE` or null.

---

## 23. modules/paazl/paazl.api.service.ts

HTTP client for Paazl API (shipping).

### Class `PaazlApiService`

- **Constructor:** Axios client with `baseURL: process.env.PAAZL_URL`, auth header from `#getAuthHeader()` (Bearer `PAAZL_API_KEY:PAAZL_API_SECRET`).
- **`getCheckoutToken(reference)`** — POST `/checkout/token`, body `{ reference }`. Returns `data.token`.
- **`getCheckout(reference)`** — GET `/checkout`, params `{ reference }`. Returns full `data`.
- **`getShippingOptions(params)`** — POST `/shippingoptions`, returns `data.shippingOptions ?? []`.
- **`#getAuthHeader()`** — `Bearer ${PAAZL_API_KEY}:${PAAZL_API_SECRET}`.

---

## 24. modules/paazl/paazl.service.ts

Paazl delivery info for a basket (token + default shipping option).

### Class `PaazlService`

- **Constructor:** Creates `PaazlApiService`.
- **`getDefaultDeliveryInfo({ basket, postalCode, countryCode, shippingOptionIdentifier, paazlAPIToken })`**  
  - Gets checkout token: uses `paazlAPIToken` if provided, else `getCheckoutToken(basket.basketId)`.  
  - Calls `getShippingOptions` with consigneeCountryCode, consigneePostalCode, token, and `getShipmentParameters(basket)` (weight, price, quantity per product).  
  - Picks default option by `shippingOptionIdentifier` or first option.  
  - Returns `{ checkoutToken, deliveryInfo }` where `deliveryInfo` is JSON string with carrierName, carrierDescription, cost, identifier, name, deliveryDates.
- **`getShipmentParameters(basket)`** (private) — Builds Paazl shipment payload from basket product items (totalWeight, totalPrice, numberOfGoods, goods array). Default weight 1 kg per item.
- **`getShippingOptionDeliveryInfo(shippingOption)`** (private) — Serializes one shipping option to the deliveryInfo shape above.

---

## 25. modules/paazl/index.ts

Re-exports `PaazlApiService` and `PaazlService`.

---

## 26. modules/sfcc/sfcc.controller.ts

Express router for SFCC-related endpoints. All require query `siteId`, `locale`, and header `authorization` (shopper token) where relevant.

### GET `/basket/:basketId/shipping-methods/default`

- **Query:** `siteId`, `locale`, `countryCode`. **Params:** `basketId`. **Header:** `authorization`.
- **Handler:** Creates `ShippingMethodsService` with siteId, locale, countryCode, basketId, shopperToken. Calls `getDefaultShippingInfo()`. Returns `200` with JSON result (either standard shipping method or Paazl type with shippingMethod + paazl.checkoutToken + paazl.deliveryInfo).

### GET `/basket/:basketId/paazl/delivery-info`

- **Query:** `siteId`, `locale`, `countryCode`. **Params:** `basketId`. **Header:** `authorization`.
- **Handler:** Creates `PaazlDeliveryInfoService`, calls `getPaazlDeliveryInfo()`. If result exists, `200` and `{ deliveryInfo }`; else `404`.

### POST `/orders`

- **Body:** `basketId` (string, required). **Query:** `siteId`, `locale`. **Header:** `authorization`. Uses `express-validator` and `validatorMiddleware`.
- **Handler:** `matchedData` for basketId, siteId, locale, authorization. Creates `PlaceOrderService.create({ siteId, locale, shopperToken })`, calls `placeOrder(basketId)`. Returns `200` and `{ orderNo }`.

### POST `/orders/:orderNo/cancel`

- **Params:** `orderNo`. **Query:** `siteId`, `locale`. **Header:** `authorization`. Uses validation and `validatorMiddleware`.
- **Handler:** Creates `CancelOrderService`, calls `cancelOrder(orderNo)`. On success `204`. On error, attaches `metadata: { orderNo }` and `context: 'CANCEL_ORDER'` to error and calls `next(e)`.

---

## 27. modules/sfcc/sfcc-admin-auth.service.ts

Obtains and caches SFCC **admin** (client credentials) access token for Commerce API.

### Class `SfccAdminAuthenticator`

- **Constructor:** Axios client for `https://account.demandware.com` with Basic auth (client id/secret from env). Uses `getCacheManager()` for token cache.
- **`getAccessToken()`** — If cache has key `sfcc-admin-access-token`, returns it. Otherwise POSTs to `/dwsso/oauth2/access_token` with grant_type client_credentials, scope `SALESFORCE_COMMERCE_API:{realmId}_{instanceId} {scopes}` (from `SFCC_REALM_ID`, `SFCC_INSTANCE_ID`, `SFCC_OAUTH_SCOPES`). Caches `access_token` with TTL `expires_in - 60` seconds. Returns token.
- **`#getAuthHeader()`** — Basic header from `SFCC_ADMIN_CLIENT_ID` and `SFCC_ADMIN_CLIENT_SECRET`.

---

## 28. modules/sfcc/sfcc-admin.service.js

Commerce API calls that require **admin** token (site and global custom preferences).

### Class `SfccAdminService`

- **Constructor:** Creates `SfccAdminAuthenticator` and axios client with base URL from `SHORT_CODE.api.commercecloud.salesforce.com`.
- **`getSiteCustomPreferences(siteId)`** — GET `/configuration/preferences/v1/organizations/{ORGANIZATION_ID}/site-custom-preferences?siteId=`. Uses Bearer from authenticator. Returns `#getMappedPreferences(data.data)`.
- **`getGlobalCustomPreferences()`** — GET same path but `global-custom-preferences`. Returns same mapped shape.
- **`#getAuthHeader()`** — Bearer token from authenticator.
- **`#getMappedPreferences(preferences)`** — Reduces array of preference items to object: `{ [groupId]: { [id]: value } }`.

---

## 29. modules/sfcc/admin-order.service.ts

Checkout Orders API (admin) for a given site: update order status, payment status, export status, confirmation status.

### Class `AdminOrderService`

- **Constructor:** `(siteId)`. Creates authenticator and axios client for `.../checkout/orders/v1/organizations/{org}/orders`, with `siteId` as query param.
- **`updatePaymentStatus(orderNo, paymentStatus)`** — PUT `/orders/{orderNo}/payment-status`, body `{ status }`. Uses `PaymentStatus` from types.
- **`updateExportStatus(orderNo, exportStatus)`** — PUT `/orders/{orderNo}/export-status`, body `{ status }`.
- **`updateStatus(orderNo, status)`** — PUT `/orders/{orderNo}/status`, body `{ status }`. Returns response (used by cancel-order to read Location header).
- **`updateConfirmationStatus(orderNo, confirmationStatus)`** — PUT `/orders/{orderNo}/confirmation-status`, body `{ status }`.

---

## 30. modules/sfcc/shopper-baskets.service.ts

Shopper Baskets API wrapper (Commerce SDK).

### Class `ShopperBasketsService`

- **Constructor:** Takes Commerce SDK `ShopperBaskets` client.
- **`getBasket(basketId)`** — Get basket by ID.
- **`getShippingMethods(basketId)`** — Get shipping methods for shipment id `'me'`.
- **`updateShippingAddressForShipment(basketId, address)`** — Update shipping address for shipment `'me'`.
- **`addPaymentInstrument(basketId, data)`** — Add payment instrument; `data.paymentMethodId` typed as `PaymentMethodId` (e.g. CASH, GIFT_CERTIFICATE).
- **`removePaymentInstrumentFromBasket(basketId, paymentInstrumentId)`** — Remove one instrument.
- **`static create({ siteId, locale, shopperToken })`** — Builds SDK client from app config and creates service. Used by place-order, cancel-order, shipping-methods, paazl-delivery-info.

---

## 31. modules/sfcc/shopper-orders.service.ts

Shopper Orders API wrapper.

### Class `ShopperOrdersService`

- **Constructor:** Takes Commerce SDK `ShopperOrders` client.
- **`getOrder(orderNo)`** — Get order by number.
- **`createOrder(basketId)`** — Create order from basket.
- **`static create({ siteId, locale, shopperToken })`** — Same pattern as baskets; used by place-order and cancel-order.

---

## 32. modules/sfcc/shipping-methods.service.ts

Returns default shipping info for a basket: either **standard** (SFCC methods) or **Paazl** (token + delivery info).

### Class `ShippingMethodsService`

- **Constructor:** `({ siteId, locale, countryCode, basketId, shopperToken })`. Creates `ShopperBasketsService`, `PaazlService`, `PaazlPreferencesService`.
- **`getDefaultShippingInfo()`** — Gets basket; if shipment country !== countryCode, updates shipping address. Gets applicable shipping methods and default id. Gets Paazl preferences for site/country (paazlEnabled, postalCode, shippingOptionIdentifier). If Paazl enabled and postalCode set, returns Paazl result (type `'paazl'`, shippingMethod, paazl.checkoutToken, paazl.deliveryInfo). Else returns standard (type `'standard'`, single shipping method).
- **`getPaazlShippingInfo(...)`** (private) — Finds Paazl shipping method by currency (`paazl_{CURRENCY}`), calls `paazlService.getDefaultDeliveryInfo`, returns combined object.
- **`getStandardShippingInfo(shippingMethods, defaultShippingMethodId)`** (private) — Returns default or first method.
- **`getPaazlShippingMethodId(currency)`** (private) — `paazl_${currency.toUpperCase()}`.

---

## 33. modules/sfcc/place-order.service.ts

Creates an order from a basket and updates order/export/confirmation status (or redeems gift card).

### Class `PlaceOrderService`

- **Constructor:** Takes shopperBaskets, shopperOrders, giftCardService, adminOrderService.
- **`placeOrder(basketId)`** — Gets basket. If no basket/invalid, throws `BadRequestError('Basket not found or invalid')`. If no payment instruments and order is not fully covered by promotion, throws `BadRequestError('Invalid basket')`. If no payment instruments but fully covered, adds CASH instrument with amount 0. Creates order via `shopperOrders.createOrder(basketId)`. Then: if basket has gift card, calls `giftCardService.redeem(orderNo)` and validates redeemed total; else updates order status NEW, payment PAID, export READY, confirmation CONFIRMED via admin API. On any error after order creation, sets order status to FAILED_REOPEN then rethrows. Returns order.
- **`isOrderFullyCoveredByPromotion(basket)`** (private) — True when orderTotal, shippingTotal, c_amountOpen are 0 and there are coupon items and order price adjustments.
- **`static create({ siteId, locale, shopperToken })`** — Builds ShopperBasketsService, ShopperOrdersService, GiftCardService(siteId, shopperToken), AdminOrderService(siteId), returns new PlaceOrderService.

---

## 34. modules/sfcc/cancel-order.service.ts

Cancels an order (FAILED_REOPEN), restores basket from Location header, removes non–gift-card payment instruments from restored basket.

### Class `CancelOrderService`

- **Constructor:** Takes adminOrderService, shopperBaskets, shopperOrders.
- **`cancelOrder(orderNo)`** — Calls `shopperOrders.getOrder(orderNo)` (validates ownership). Calls `adminOrderService.updateStatus(orderNo, FAILED_REOPEN)`. Reads restored basket ID from response `Location` header (last path segment). If missing, throws. Gets restored basket, then calls `removePaymentInstrumentsFromBasket(basket)`.
- **`getRestoredBasketId(response)`** (private) — `response.headers?.location?.split('/')?.at(-1) ?? null`.
- **`removePaymentInstrumentsFromBasket(basket)`** (private) — Filters out GIFT_CERTIFICATE, then removes each remaining payment instrument from basket via shopperBaskets.
- **`static create({ siteId, locale, shopperToken })`** — Creates AdminOrderService(siteId), ShopperBasketsService, ShopperOrdersService, returns new CancelOrderService.

---

## 35. modules/sfcc/cancel-order.service.spec.ts

Jest unit tests for `CancelOrderService`: cancelOrder success, missing location header, null location, no payment instruments, multiple instruments, undefined paymentInstruments, unauthorized (user doesn’t own order), and `create()` factory.

---

## 36. modules/sfcc/gift-card.service.ts

Gift certificate redeem via custom Commerce API endpoint.

### Constants / types

- **`PAYMENT_METHODS`** — Includes `GIFT_CARD: 'GIFT_CERTIFICATE'`.
- **`GiftCardRedeemResponse`** — success, redeemed_total, remainingAmount, redeem_info array.

### Class `GiftCardService`

- **Constructor:** `(siteId, authorization)` — shopper token for the redeem request.
- **`redeem(orderNo)`** — POST to custom endpoint `.../custom/gift-certificate/v1/organizations/{org}/redeem?siteId={siteId}` with body `{ orderId: orderNo }`, Authorization header. Returns response data.
- **`canRedeem(paymentInstruments)`** — True if any instrument has `paymentMethodId === GIFT_CERTIFICATE`.
- **`getRedeemUrl()`** (private) — Builds URL from COMMERCE_API_SHORT_CODE, COMMERCE_API_ORG_ID, siteId.

---

## 37. modules/sfcc/paazl-preferences.service.ts

Reads Paazl-related site custom preferences from SFCC.

### Class `PaazlPreferencesService`

- **Constructor:** Creates `SfccAdminService`.
- **`getPreferences(siteId, countryCode)`** — Gets site custom preferences, reads group `CUSTOM_PREFERENCES_SCOPE.PAAZL` ('Paazl'). Parses `paazlDefaultShippingOption` as JSON. Returns `{ paazlEnabled, postalCode, shippingOptionIdentifier }` for the given countryCode from that JSON.

---

## 38. modules/sfcc/paazl-delivery-info.service.ts

Returns Paazl delivery info for a basket when Paazl is enabled and postal code is configured.

### Class `PaazlDeliveryInfoService`

- **Constructor:** Same pattern as ShippingMethodsService (siteId, locale, countryCode, basketId, shopperToken). Creates ShopperBasketsService, PaazlService, PaazlPreferencesService.
- **`getPaazlDeliveryInfo()`** — Gets basket; requires `basket.c_paazlAPIToken`, else throws. Gets Paazl preferences for site/country. If paazlEnabled and postalCode set, calls `paazlService.getDefaultDeliveryInfo` (with paazlAPIToken) and returns `deliveryInfo`. Otherwise returns null.

---

## 39. modules/sfcc/global-custom-preferences.service.ts

Fetches and caches **global** custom preferences; exposes Apple Pay, PayPal, and Gift Card enabled flags by country/locale/location.

### Class `GlobalCustomPreferencesService`

- **Constructor:** Creates `SfccAdminService` and gets `getCacheManager()`.
- **Static paths:** `APPLE_PAY_CONFIG_PATH`, `PAYPAL_CONFIG_PATH`, `GIFT_CARD_CONFIG_PATH` for dot paths into preferences.
- **`getPaymentPreferences(countryCode, locale, location)`** — Loads (cached) preferences for those three paths. Returns `{ isApplePayEnabled, isPayPalEnabled, isGiftCardEnabled }` using private helpers.
- **`getPreferencesCached(configPath)`** (private) — Cache key is path (or joined array). If not in cache, fetches via `getPreferencesByPath`, sets cache. Returns cached value.
- **`getPreferencesByPath(configPath)`** (private) — Gets global custom preferences from SFCC. If configPath is array, reduces to object of parsed JSON per path; else single path, returns tryParseJSON of value. Uses `getIn` for nested keys.
- **`getApplePayEnabled(countryCode, locale, location, config)`** (private) — Resolves full locale (e.g. en_SE), then looks up config[fullLocale] or config[locale], then config[location].
- **`getPayPalEnabled(...)`** (private) — For non-cart location always true; for cart uses fullLocale/countryCode/locale from config.
- **`getGiftCardEnabled(countryCode, locale, config)`** (private) — countryCode or fullLocale; default true.
- **`getFullLocale(locale, countryCode)`** (private) — Converts e.g. `en-SE` to `en_SE`, or uses countryCode if no country in locale.

---

## 40. modules/sfcc/global-custom-preferences.service.test.ts

Jest tests for GlobalCustomPreferencesService: getPreferencesCached (miss/hit), getFullLocale, getApplePayEnabled (various configs), getGiftCardEnabled (countryCode, fullLocale, defaults, edge cases). Mocks SfccAdminService and cache-manager.

---

## Summary table

| Area        | Main entry              | Purpose |
|------------|--------------------------|--------|
| **Entry**  | main.js                  | Mount Emarsys + SFCC routes; global error handler |
| **Middleware** | validate-request.middleware | 400 on express-validator errors |
| **Utils**  | constants, errors, general, parse, crypto | Shared BFF helpers |
| **Types**  | sfcc, common             | Order/payment/status types and factory params |
| **Logger** | logger.service           | [COMPOSABLE] console logger |
| **Cache**  | cache-manager            | NodeCache singleton (30 min TTL) |
| **Emarsys**| controller → subscriber → service → api | Newsletter/subscribe; WSSE auth; create/update contact + form trigger |
| **Paazl**  | paazl.service + api      | Checkout token, shipping options, default delivery info |
| **SFCC**   | sfcc.controller          | Shipping methods (default + Paazl), place order, cancel order |
| **SFCC admin** | sfcc-admin-auth, sfcc-admin, admin-order | Token cache; site/global preferences; order status updates |
| **SFCC shopper** | shopper-baskets, shopper-orders | Basket and order SDK wrappers |
| **SFCC flows** | place-order, cancel-order, gift-card | Order creation, gift redeem, cancel + restore basket |
| **SFCC preferences** | paazl-preferences, global-custom-preferences | Paazl config; Apple Pay / PayPal / Gift Card flags |

This is the complete reference for every file and function in `src/api`.
