# Paazl — Knowledge Transfer (KT) Document

This document helps anyone new to the project understand **Paazl**: what it is, how it’s used, where it lives in the codebase, and how to work with it. Use it for onboarding, debugging, or making changes.

---

## 1. What is Paazl?

**Paazl** is a **shipping and delivery provider** integrated into the PWA. It lets customers:

- Choose **delivery options** (e.g. home delivery with time windows, carrier, price).
- Choose **pickup locations** (PUDO — Pick Up Drop Off) when available.

The integration uses:

| Component | Role |
|-----------|------|
| **Paazl REST API** (external) | Token, checkout state, shipping options, pickup locations |
| **BFF** (PWA `src/api` + overrides) | Proxies and orchestrates calls to Paazl and SFCC |
| **Paazl Checkout Widget** (JavaScript) | Third-party UI: Delivery/Pickup tabs, option selection, pickup point selection |
| **SFCC / Commerce API** | Shipping method `paazl_{CURRENCY}`, basket/shipment custom attributes |

---

## 2. Why We Use Paazl

- **Multi-carrier and PUDO**: One integration for multiple carriers and pickup networks.
- **Country-specific**: Enabled per site/country via SFCC site preferences.
- **Consistent UX**: Same widget and flow across sites that have Paazl enabled.

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PWA Frontend (React)                                                        │
│  • Cart: country from locale/URL (or optional selector), default shipping, │
│    refresh Paazl on cart change                                              │
│  • Checkout: Paazl widget (Delivery/Pickup), "Save and continue"            │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  BFF (Node)                                                                  │
│  • src/api: GET .../shipping-methods/default, GET .../paazl/delivery-info   │
│  • overrides/app/api/paazl: paazl-config, paazl-token, paazl-options,       │
│    paazl-checkout, shipping-options, pickup-locations                      │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
┌───────────────────┐   ┌─────────────────────┐   ┌─────────────────────────┐
│  Commerce API     │   │  Paazl API           │   │  SCAPI                  │
│  (SFCC)           │   │  (external)          │   │  Site custom preferences │
│  Basket, shipment │   │  Token, checkout,   │   │  Paazl group:            │
│  c_paazlAPIToken  │   │  shippingoptions,   │   │  paazlEnabled,           │
│  c_paazlDelivery  │   │  pickuplocations    │   │  paazlWidgetEndpoint,    │
│  Info             │   │                     │   │  paazlDefaultShipping…   │
└───────────────────┘   └─────────────────────┘   └─────────────────────────┘
```

- **Cart flows** (default shipping, refresh delivery info) use **src/api** (ShippingMethodsService, PaazlService, PaazlDeliveryInfoService).
- **Checkout flow** (widget config, token, tabs, submit) uses **overrides** Paazl API handlers.

---

## 4. Key Concepts

### 4.1 Paazl checkout token

- Obtained from Paazl: `POST {PAAZL_URL}/checkout/token` with `{ reference: basketId }`.
- Tied to a basket; used for all Paazl calls for that basket.
- We store it on the basket as **`c_paazlAPIToken`** so we can reuse it (e.g. when cart items change).

### 4.2 Delivery info

- JSON string describing the selected option: carrier, cost, identifier, name, delivery dates; for pickup, also pickup location (name, address, opening times, code).
- We store it on the **shipment** as **`c_paazlDeliveryInfo`** (string).
- Used to show the chosen method in cart/checkout summary and for order processing.

### 4.3 Shipping method ID

- In SFCC, Paazl is represented as a shipping method with ID **`paazl_{CURRENCY}`** (e.g. `paazl_EUR`, `paazl_GBP`).
- This ID is set on the shipment when the user has a Paazl option selected.

### 4.4 Country / locale (cart)

- **Default:** The shipping country used for default shipping and Paazl is **not** from a dropdown. It is derived from the **locale/URL** via **useExtendedMultiSite()** — the site’s default country for the current locale (e.g. `/uk-en` → UK). See `use-extended-multi-site.js` (countryCode from `site.l10n.defaultCountry`).
- **Optional:** If cart content has **countrySelector.enabled** (Contentstack), a **country selector** is shown in the order summary; the user can change country and that value is used (and stored in localStorage). Logic: **useCountryCode** returns `defaultCountry` (from locale) when selector is disabled, or the dropdown value when enabled.

### 4.5 Paazl preferences (SFCC)

- Stored in **Site Custom Preferences**, group **`Paazl`** (see `CUSTOM_PREFERENCES_SCOPE.PAAZL` in `src/api/utils/constants.ts`).
- Examples: `paazlEnabled`, `paazlWidgetEndpoint`, `paazlDefaultShippingOption` (JSON per country: `postalCode`, `shippingOptionIdentifier`), widget tabs/style/limits.

---

## 5. End-to-End Flows

### 5.1 Flow 1: Cart — Default shipping by country (from locale/URL or optional selector)

**Goal:** Use the current shipping country to set default shipping method; if Paazl is enabled for that country, get a default Paazl option and store token + delivery info.

**Where country comes from:** By default, **country is derived from the locale/URL** (via `useExtendedMultiSite()` — site’s default country for the current locale). There is **no** country selector in that case. When cart content has **country selector enabled** (`countrySelector.enabled` from Contentstack), a dropdown is shown and the user can change country; that value is then used (and stored in localStorage).

1. On cart load (or when user changes country in the optional selector), **selectedCountryCode** is set: either from **useExtendedMultiSite()** (locale/URL) or from the dropdown.
2. Frontend calls **`GET /api/basket/:basketId/shipping-methods/default?siteId=...&locale=...&countryCode=...`** (with Bearer token), where **countryCode** = selectedCountryCode.
3. BFF (`sfcc.controller.ts` → `ShippingMethodsService.getDefaultShippingInfo()`):
   - Gets/updates basket shipping address to match country.
   - Gets shipping methods; reads Paazl preferences for site + country.
   - If `paazlEnabled` and `postalCode` are set: calls `PaazlService.getDefaultDeliveryInfo()` (token + shipping options), picks default by `shippingOptionIdentifier` or first option.
   - Returns either `{ type: 'paazl', shippingMethod, paazl: { checkoutToken, deliveryInfo } }` or `{ type: 'standard', shippingMethod }`.
4. Frontend applies result:
   - If Paazl: **updateBasket** with `c_paazlAPIToken`, **updateShipmentForBasket** with `shippingMethod.id` and `c_paazlDeliveryInfo`.

**Key files:**  
`overrides/app/pages/cart/partials/cart-country-selector.tsx`, `overrides/app/hooks/use-select-default-shipping-method.ts`, `src/api/modules/sfcc/sfcc.controller.ts`, `src/api/modules/sfcc/shipping-methods.service.ts`, `src/api/modules/paazl/paazl.service.ts`, `src/api/modules/paazl/paazl.api.service.ts`.

---

### 5.2 Flow 2: Cart — Basket already has Paazl; items change

**Goal:** Keep Paazl delivery info in sync when the user adds/removes items or changes quantity.

1. Cart page has a **useEffect** that runs when `basket?.productTotal` (or equivalent) changes.
2. If current shipping method is Paazl (`id.startsWith('paazl_')` — see `overrides/app/utils/shipping-utils.ts`: `isPaazlShippingMethod`) and a country is selected, frontend calls **`GET /api/basket/:basketId/paazl/delivery-info?siteId=...&locale=...&countryCode=...`**.
3. BFF (`PaazlDeliveryInfoService.getPaazlDeliveryInfo()`):
   - Requires basket to have `c_paazlAPIToken`.
   - Uses preferences; if Paazl enabled and postal code set, calls `PaazlService.getDefaultDeliveryInfo()` with existing token.
   - Returns new `deliveryInfo` string.
4. Frontend updates shipment with **`c_paazlDeliveryInfo`** via Commerce API.

**Key files:**  
`overrides/app/pages/cart/index.jsx` (useEffect), `overrides/app/hooks/use-update-paazl-delivery-info.ts`, `src/api/modules/sfcc/paazl-delivery-info.service.ts`, `src/api/modules/paazl/paazl.service.ts`.

---

### 5.3 Flow 3: Checkout — Shipping step and Paazl widget

**Goal:** Show Paazl widget on shipping step; when user clicks “Save and continue”, read selection from Paazl and save to basket.

**3a. Load config and token**

- Checkout context uses **`usePaazlConfig(basket)`**.
- Frontend calls **`GET /api/paazl-config?basketId=...&siteId=...`**.
- BFF (overrides): gets SCAPI admin token; in parallel fetches SFCC site Paazl preferences and Paazl token (`POST .../checkout/token`). Returns merged config (e.g. `paazlWidgetEndpoint`, `paazlEnabled`, widget tabs/style/limits) + token.
- Frontend derives **isPaazlAvailable** (e.g. `paazlEnabled` and current country in `paazlDefaultShippingOption`). If true, shipping step renders **PaazlShippingOptions** (and thus the widget).

**3b. Widget mount and init**

- **PaazlShippingOptions** renders **PaazlWidget** with basket, config, widget endpoint.
- **PaazlWidget** (`overrides/app/components/paazl-widget/index.jsx`):
  - Uses **usePaazlToken(basket)** → `GET /api/paazl-token?basketId=...` for token.
  - Uses **usePaazlOptionsConfig(basket, paazlToken)** → `GET /api/paazl-options?postalCode=...&countryCode=...&paazlToken=...` to get **availableTabs** (DELIVERY, PICKUP).
  - Loads script from `paazlWidgetEndpoint`, then calls **`window.PaazlCheckout.init(...)`** with token, address, shipment parameters, config, mount element `#paazl-checkout`.
- Paazl’s script renders Delivery/Pickup UI inside `#paazl-checkout`. We only provide the “Save and continue” button; all option/pickup selection is inside the widget.

**3c. User clicks “Save and continue”**

- **PaazlShippingOptions** `handleSubmit`:
  1. Calls **`GET /api/paazl-checkout?basketId=...&currencyCode=...`**.
  2. BFF (overrides) calls Paazl **`GET {PAAZL_URL}/checkout?reference=basketId`**, then **parseResponse** to build delivery info (ID, carrier, cost, identifier, name, deliveryDates; for pickup, pickupLocation with name, address, openingTimes, code).
  3. Frontend receives delivery info and calls **onSubmit({ shippingMethodId: deliveryInfo.ID, c_paazlDeliveryInfo: JSON.stringify(deliveryInfo) })**.
  4. Shipping step **updateShipmentForBasket** with `shippingMethod: { id }` and `c_paazlDeliveryInfo`, then continues to next step.

**Key files:**  
`overrides/app/pages/checkout/context/checkout-context.tsx`, `overrides/app/pages/checkout/partials/shipping-options/shipping-options-step.jsx`, `paazl-shipping-options.jsx`, `overrides/app/components/paazl-widget/index.jsx`, `overrides/app/hooks/use-paazl-config.jsx`, `use-paazl-token.jsx`, `use-paazl-options-config.jsx`, `overrides/app/api/paazl/*` (paazl-config.js, paazl-token.js, paazl-tabs-config.js, paazl-checkout.js, helper.js).

---

## 6. Code Map — Where to Find What

### 6.1 Frontend (overrides)

| Area | Path | Purpose |
|------|------|---------|
| Country source | `overrides/app/hooks/use-country-code.ts`, `use-extended-multi-site.js` | selectedCountryCode: from locale/URL when selector disabled, from dropdown when enabled |
| Optional country selector | `overrides/app/pages/cart/partials/cart-country-selector.tsx` | Shown only when countrySelector.enabled in cart content; country change → default shipping |
| Cart context | `overrides/app/pages/cart/context/cart-provider.js` | selectedCountryCode, setSelectedCountryCode, updatePaazlDeliveryInfo, isBasketUpdatingForPaazl |
| Default shipping hook | `overrides/app/hooks/use-select-default-shipping-method.ts` | Calls default shipping API, applies Paazl token + delivery info to basket/shipment |
| Paazl refresh hook | `overrides/app/hooks/use-update-paazl-delivery-info.ts` | Calls paazl/delivery-info, updates shipment c_paazlDeliveryInfo |
| Cart trigger | `overrides/app/pages/cart/index.jsx` | useEffect on basket.productTotal to refresh Paazl when items change |
| Shipping utils | `overrides/app/utils/shipping-utils.ts` | `isPaazlShippingMethod(id)` |
| Checkout context | `overrides/app/pages/checkout/context/checkout-context.tsx` | usePaazlConfig(basket) |
| Shipping step | `overrides/app/pages/checkout/partials/shipping-options/shipping-options-step.jsx` | isPaazlAvailable, onSubmit (updateShipment), summary label |
| Paazl vs standard | `overrides/app/pages/checkout/partials/shipping-options/shipping-options.jsx` | Renders PaazlShippingOptions or standard options |
| Paazl shipping UI | `overrides/app/pages/checkout/partials/shipping-options/paazl-shipping-options.jsx` | Widget + “Save and continue”, getPaazlDeliveryInfo, onSubmit |
| Paazl widget | `overrides/app/components/paazl-widget/index.jsx` | Script load, PaazlCheckout.init, #paazl-checkout, styles |
| Widget helper | `overrides/app/components/paazl-widget/helper.js` | getShipmentParameters(basket) |
| Paazl hooks | `overrides/app/hooks/use-paazl-config.jsx`, `use-paazl-token.jsx`, `use-paazl-options-config.jsx` | Config, token, available tabs |

### 6.2 BFF — src/api (Cart flows)

| Area | Path | Purpose |
|------|------|---------|
| Routes | `src/api/modules/sfcc/sfcc.controller.ts` | GET .../shipping-methods/default, GET .../paazl/delivery-info |
| Default shipping logic | `src/api/modules/sfcc/shipping-methods.service.ts` | getDefaultShippingInfo, Paazl vs standard, getPaazlShippingMethodId |
| Paazl preferences | `src/api/modules/sfcc/paazl-preferences.service.ts` | getPreferences(siteId, countryCode) from SCAPI Paazl group |
| Paazl delivery info | `src/api/modules/sfcc/paazl-delivery-info.service.ts` | getPaazlDeliveryInfo (uses basket token + PaazlService) |
| Paazl business logic | `src/api/modules/paazl/paazl.service.ts` | getDefaultDeliveryInfo, getShipmentParameters, getShippingOptionDeliveryInfo |
| Paazl API client | `src/api/modules/paazl/paazl.api.service.ts` | getCheckoutToken, getCheckout, getShippingOptions (axios, PAAZL_URL, Bearer auth) |
| Constants | `src/api/utils/constants.ts` | CUSTOM_PREFERENCES_SCOPE.PAAZL = 'Paazl' |

### 6.3 BFF — overrides (Checkout flows)

| Area | Path | Purpose |
|------|------|---------|
| Paazl routes | `overrides/app/api/paazl/index.js` | registerPaazlEndpoints: paazl-config, paazl-checkout, paazl-options, paazl-token |
| Config | `overrides/app/api/paazl/paazl-config.js` | handlePaazlConfigRequest, fetchPaazlConfig (SCAPI), fetchPaazlToken |
| Token | `overrides/app/api/paazl/paazl-token.js` | handlePaazlTokenRequest, POST .../checkout/token |
| Tabs | `overrides/app/api/paazl/paazl-tabs-config.js` | handlePaazlOptionsConfigRequest, shippingoptions + pickuplocations → availableTabs |
| Checkout (submit) | `overrides/app/api/paazl/paazl-checkout.js` | handlePaazlCheckoutRequest, GET Paazl /checkout, parseResponse |
| Helpers | `overrides/app/api/paazl/helper.js` | filterData (config keys), parseResponse, formatShippingAddress, getShippingMethodID |
| Shipping options | `overrides/app/api/paazl/shipping-options.js` | POST .../shippingoptions (used from paazl-tabs-config) |
| Pickup locations | `overrides/app/api/paazl/pickup-locations.js` | POST .../pickuplocations (used from paazl-tabs-config) |

---

## 7. Configuration

### 7.1 Environment variables

Set in `.env` (see `.env.example`):

| Variable | Purpose |
|----------|---------|
| `PAAZL_URL` | Paazl API base URL (e.g. `https://api-acc.paazl.com/v1`) |
| `PAAZL_API_KEY` | Paazl API key |
| `PAAZL_API_SECRET` | Paazl API secret |

Auth header used in `paazl.api.service.ts` and overrides: `Bearer ${PAAZL_API_KEY}:${PAAZL_API_SECRET}`.

### 7.2 SFCC site custom preferences (Paazl group)

- **paazlEnabled** — Whether Paazl is on for the site.
- **paazlDefaultShippingOption** — JSON per countryCode: `{ postalCode, shippingOptionIdentifier }` (used for default option on cart).
- **paazlWidgetEndpoint** — URL of Paazl Checkout Widget script.
- **paazlWidgetDeliveryRangeFormat**, **paazlWidgetAvailableTabs**, **paazlWidgetDefaultTabs**, **paazlWidgetPredefinedStyle**, **paazlWidgetShippingOptionsLimit**, **paazlWidgetPickupLocationsPageLimit**, **paazlWidgetPickupLocationsLimit** — Widget behaviour and styling.

---

## 8. Data Stored on Basket / Shipment

| Location | Attribute | Type | Description |
|----------|-----------|------|-------------|
| Basket | `c_paazlAPIToken` | string | Paazl checkout token for this basket |
| Shipment | `shippingMethod.id` | string | e.g. `paazl_EUR` when Paazl is selected |
| Shipment | `c_paazlDeliveryInfo` | string | JSON string of selected option (carrier, cost, identifier, name, deliveryDates; for pickup, pickupLocation) |

These custom attributes must exist in your SFCC basket/shipment type definitions.

---

## 9. API Reference (Quick)

### 9.1 Our BFF APIs (used by frontend)

| Method + path | Used in | Purpose |
|---------------|---------|---------|
| GET `/api/basket/:basketId/shipping-methods/default` | Cart (country change) | Default shipping; when Paazl enabled returns token + deliveryInfo |
| GET `/api/basket/:basketId/paazl/delivery-info` | Cart (items change) | Refresh c_paazlDeliveryInfo |
| GET `/api/paazl-config` | Checkout | Widget config + Paazl token |
| GET `/api/paazl-token` | Checkout (widget) | Paazl token for widget init |
| GET `/api/paazl-options` | Checkout (widget) | availableTabs (DELIVERY/PICKUP) for postal code |
| GET `/api/paazl-checkout` | Checkout (Save and continue) | Current Paazl selection → deliveryInfo for shipment |

### 9.2 Paazl external API (used by BFF)

| Method + path | Purpose |
|---------------|---------|
| POST `{PAAZL_URL}/checkout/token` | Get checkout token (body: `{ reference: basketId }`) |
| GET `{PAAZL_URL}/checkout?reference=basketId` | Get current checkout state (deliveryType, shippingOption, pickupLocation) |
| POST `{PAAZL_URL}/shippingoptions` | Get delivery options for address |
| POST `{PAAZL_URL}/pickuplocations` | Get pickup locations for address |

---

## 10. Troubleshooting

- **Paazl not showing on cart/checkout**  
  Check: (1) `paazlEnabled` and country in `paazlDefaultShippingOption` in site preferences, (2) `postalCode` set for that country for default option, (3) env `PAAZL_URL`, `PAAZL_API_KEY`, `PAAZL_API_SECRET` set and correct.

- **“c_paazlAPIToken is missed in the basket”**  
  Delivery-info refresh (cart items change) requires the basket to already have a token. Ensure Flow 1 (country change → default shipping) runs first when Paazl is enabled, or that checkout has requested a token so basket was updated.

- **Widget not loading**  
  Check `paazlWidgetEndpoint` in site preferences and that the script URL is reachable. Check browser console for script/init errors; ensure `window.PaazlCheckout` exists after script load.

- **Wrong default option on cart**  
  Check `paazlDefaultShippingOption` for the country: `shippingOptionIdentifier` must match an option returned by Paazl for that postal code.

- **Submit returns 500 or empty**  
  Ensure user has selected an option/pickup in the widget before “Save and continue”. Check BFF logs for Paazl API errors (e.g. invalid token, wrong reference).

---

## 11. Related Documentation

- **Paazl_Flow_Complete_Story.md** — Detailed user story with APIs, clicks, and code locations for each flow.
- **PWA_Integrations_And_Features_Guide.md** — High-level integrations overview.
- **Src_API_Complete_Reference.md** — BFF `src/api` structure and modules.
- **Commerce_API_Usage_In_The_Project.md** — How we use Commerce API (basket, shipment).
- SFCC-side: `sfcc-demandware/paazl_documentation/` and `sfcc-demandware/cartridges/int_paazl_core/` — SFCC cartridge and Business Manager setup (if you need backend/OCAPI context).

---

*Last updated to align with the codebase and Paazl_Flow_Complete_Story.md. For flow-level detail and exact code references, use Paazl_Flow_Complete_Story.md.*
