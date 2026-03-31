# Checkout shipping address: end-to-end flow

This document describes how a **shipping address** is resolved, validated, transformed, and persisted in **pc-composable-storefront**, from the first page load through the Commerce Cloud basket APIs. It reflects the current `overrides/` implementation.

---

## 1. Where `siteId` comes from

1. On app bootstrap, **`AppConfig.restore`** runs (SSR and client). It parses the current URL, resolves **site** and **locale** via `resolveSiteFromUrl` / `resolveLocaleFromUrl` in `overrides/app/utils/site-utils.js`, and sets:
   - `apiConfig.parameters.siteId = site.id`
   - `locals.site`, `locals.locale`, `locals.buildUrl`

2. **`CommerceApiProvider`** (`overrides/app/components/_app-config/index.jsx`) receives `siteId={locals.site?.id}` and `locale={locals.locale?.id}`. All **Shopper API** calls from `@salesforce/commerce-sdk-react` (basket, customer, etc.) use this **site** and **locale** context.

So: the **site id is not passed manually into every hook**; it is bound once at the provider from the URL-resolved site.

---

## 2. Opening checkout: page shell and data

1. **Route container**: `overrides/app/pages/checkout/index.jsx` exports `CheckoutContainer`.
   - Waits for basket (`useCurrentBasket`), Contentstack checkout content (`useCheckoutContent`), and `customerId`.
   - Wraps the main UI in **`CheckoutProvider`** (`overrides/app/pages/checkout/context/checkout-context.tsx`).

2. **`CheckoutProvider`** loads checkout **Contentstack** entry via `useCheckoutContent`, computes whether the **shipping options** step should show (e.g. hidden for gift-certificate-only baskets), and tracks the **checkout step** index (`SHIPPING_ADDRESS` → `SHIPPING_OPTIONS` → `PAYMENT` → `REVIEW_ORDER`).

3. The main **`Checkout`** component renders **`ShippingAddressStep`** first (always), then **`ShippingOptionsStep`** when applicable, then payment.

Until Contentstack content has a usable `uid`, checkout shows **`CheckoutSkeleton`** (loading gate in `index.jsx`).

---

## 3. Shipping step component chain

| Layer | File | Role |
|--------|------|------|
| Step wrapper | `overrides/app/pages/checkout/partials/shipping-address/shipping-address-step.tsx` | Merges basket + customer into form defaults; on submit calls Shopper Baskets + customer address APIs. |
| Address UI | `overrides/app/pages/checkout/partials/shipping-address/shipping-address-selection.tsx` | Saved-address radios, add/edit flows, `react-hook-form` submit. |
| Form fields | `overrides/app/pages/checkout/partials/shipping-address/shipping-address-edit-form.tsx` | Renders inputs driven by Contentstack field config. |
| Summary | `overrides/app/pages/checkout/partials/shipping-address/address-info.tsx` | Read-only display when the step is collapsed. |

The step is wrapped in **`ToggleCard`** so users can collapse/expand the shipping section and see **`AddressInfo`** from **`basket.shipments[0].shippingAddress`** (and billing if different).

---

## 4. How the “initial” shipping address object is built

In **`shipping-address-step.tsx`**:

1. **`useContentStackShippingFields({ content })`** (`overrides/app/hooks/use-contentstack-shipping-fields.jsx`) reads checkout **`page_blocks`** from Contentstack and builds an array of field descriptors (labels, types, `country_set`, required flags, validation groups, billing checkbox, etc.). Each field maps to a **Salesforce field name** (e.g. `firstName`, `countryCode`, custom `c_*`).

2. **`getShippingAddressFromBasket(basket, customer, addressFields)`** (`overrides/app/pages/checkout/util/helpers.js`):
   - Starts from **`basket.shipments[0].shippingAddress`** (or `{}`).
   - Fills **`firstName` / `lastName`** from shipment address, else from **`customer`** (`getCustomerNames`).
   - Sets **`c_email`** from `customer.email` or `basket.customerInfo.email`.
   - Picks **`countryCode`**: if the current code is in the Contentstack **`country_set`** for the country field, keep it; otherwise default to the set’s first country (`getDefaultCountryCode`).

3. **`getBillingAddressFromBasket`** prefixes billing keys with `billing_` and applies the same country defaulting for **`billing_countryCode`**.

4. The step merges shipping + billing into **`selectedShippingAddress`** for the form’s default shape when **billing same as shipping** is toggled.

---

## 5. Site-specific country lists (taxonomy)

In **`shipping-address-selection.tsx`**:

- **`useMultiSite()`** provides **`site`** (PWA Kit multi-site).
- **`mapCountrySelector(site)`** (`helpers.js`) returns allowed country codes:
  - Prefer **`site.l10n.availableCountries`** from config when present.
  - Else use a **hardcoded map** keyed by `site.id` (e.g. `paulaschoice_de` → `['DE','AT']`, `paulaschoice_eu` → long EU list).

**`getAvailableCountries(content)`** (`overrides/app/pages/checkout/util/address/address-validation.js`) reads the **Contentstack** `country` block’s **`country_set`** for the checkout page. Saved customer addresses are **filtered** to those in `availableCountries`.

On submit, if **`address.countryCode`** is not in the site **taxonomy**, the code **forces** `countryCode` to **`taxonomy[0]`** before continuing.

---

## 6. Form setup and client-side validation

1. **`useShippingAddressForm`** (`use-shipping-address-form.ts`) chooses which Contentstack fields to show:
   - If **billing same as shipping**, it **drops** fields whose names include `billing_`.
   - Otherwise shows full shipping + billing fields.

2. **`useFormWithValidationSchema`** (`overrides/app/hooks/use-form-with-validation-schema.jsx`):
   - Builds **`formFields`** via **`useFormFields`** from Contentstack definitions.
   - **`generateFormValidationSchema`** (`overrides/app/utils/form-validation-schema-utils.js`) produces a **Yup** schema via **`FormValidationSchemaBuilder`**.
   - **`yupResolver`** is wired when feature flag **`newShippingAddressFlowEnabled`** is true; otherwise a legacy path uses `useForm` without resolver (see TODO in hook).

3. **`getExcludedFieldNames`** (`address-validation.js`) removes validation for certain fields in context:
   - Registered customers: e.g. **`c_email`**, **`signUpNewsletter`**.
   - When **FR** is in the available country list: **`c_houseNumber`** and **`billing_c_houseNumber`** (house number handled differently for France).

4. **`checkFormValidity`** uses **`validateForm`** (sync Yup) and pushes errors into **`react-hook-form`** — used when **auto-selecting** a saved address to decide if the user must edit.

5. **Extra rules** in schema utils:
   - **`phoneValidationRules`** / **`postalCodeValidationRules`** per country (`phone-validation-rules.js`, `postal-code-validation-rules.js`).
   - **`blockedPostalCodes`** — block specific postal values per country.

6. The shipping form uses **`noValidate`** on the `<form>`; validation is **programmatic** (Yup + RHF).

---

## 7. Submit path: from form to basket

**`shipping-address-selection.tsx`** → **`submitForm`**:

1. Ensures **`countryCode`** is in site taxonomy (else reset to first taxonomy country).
2. If editing a saved address, attaches **`addressId`**.
3. **EU site** (`SITE_IDS.EU`): strips spaces, dots, and hyphens from **`phone`**.
4. **`filterAddressFields(address)`** (`filter-address-fields.js`): for **FR**, drops **`c_houseNumber`** / **`billing_c_houseNumber`** from the payload (country-specific strip).
5. Calls parent **`onSubmit`** → **`submitAndContinue`** in **`shipping-address-step.tsx`**.

**`submitAndContinue`** then:

1. Optionally updates **basket customer email** via **`updateCustomerForBasket`** if it changed (registered flow rules).
2. **Guest registration**: if password present, **`AuthHelpers.Register`**; on failure, sets form error and stops.
3. **`extractShippingAddress(address)`** (`address-transformers.ts`): builds the body for the shipment API by **removing** `addressId`, `email`, metadata, **`password`**, and any key starting with **`billing_`**.
4. **`updateShippingAddressForShipment`** (Shopper Baskets mutation) with **`shipmentId: 'me'`**, **`useAsBilling: false`**, **`body: shippingAddress`** — this is the **authoritative** shipping address on the **basket** in Salesforce.
5. For **logged-in** customers, **`createOrUpdateCustomerAddress`**: **`createCustomerAddress`** or **`updateCustomerAddress`** with a generated **`addressId`** if new (`use-create-or-update-customer-address.ts` + `address-id-helper`).
6. **`onBillingSubmit`**: either copies shipping as billing or strips `billing_` prefix and calls **`updateBillingAddressForBasket`**.
7. If **shipping country** changed vs previous basket shipment, **`resetShippingMethodConfirmation`** (checkout context) so shipping method must be reconfirmed.
8. **`goToNextStep()`** advances the wizard.

On API failure, user sees a toast with **`SHIPPING_FORM_ERROR_MESSAGE`** (`localizations`).

---

## 8. Alternate path: Adyen shipping-address API

**`POST /api/adyen/shipping-address`** (`overrides/app/adyen/api/routes/index.js` → `controllers/shipping-address.js`) maps **Adyen-shaped** payload (`deliveryAddress`, `profile`) into SFCC fields and calls **`ShopperBaskets.updateShippingAddressForShipment`** with **`siteId` from query** and **`basketId` from header**.

This is a **server-side** bridge for the Adyen flow, not the main React checkout form path.

---

## 9. Structure summary

| Concern | Source of truth |
|--------|------------------|
| Site / locale for APIs | URL → `AppConfig.restore` → `CommerceApiProvider` |
| Field labels, required flags, country sets in UI | Contentstack checkout `page_blocks` |
| Allowed countries for saved addresses | Contentstack `country` block + `getAvailableCountries` |
| Allowed countries for taxonomy correction | `mapCountrySelector(site)` |
| Current shipping address on order | `basket.shipments[0].shippingAddress` after mutation |
| Saved addresses (registered) | `customer.addresses` via Shopper Customers API |

---

## 10. Validation checklist (quick reference)

- **Yup schema** from Contentstack field config (+ floating fields), minus **`excludedFieldNames`**.
- **Phone / postal** patterns and **blocked postal codes** by country (`form-validation-schema-utils`).
- **Saved address** must pass **`checkFormValidity`** when auto-selected; otherwise UI forces edit.
- **Submit**: country forced into site taxonomy; **EU** phone normalized; **FR** house-number fields stripped from payload via **`filterAddressFields`**.
- **Radio** `addressId` required when not in “add new” edit mode (`Controller` rules in selection component).
- **Server**: Shopper Baskets API enforces SFCC schema and business rules beyond the storefront.

---

## 11. Files to read first

1. `overrides/app/pages/checkout/index.jsx` — checkout entry and skeleton gate  
2. `overrides/app/pages/checkout/context/checkout-context.tsx` — steps and shipping-method confirmation  
3. `overrides/app/pages/checkout/partials/shipping-address/shipping-address-step.tsx` — basket + customer persistence  
4. `overrides/app/pages/checkout/partials/shipping-address/shipping-address-selection.tsx` — form, taxonomy, FR/EU rules  
5. `overrides/app/pages/checkout/util/helpers.js` — `getShippingAddressFromBasket`, `mapCountrySelector`, `hasShippingCountryChanged`  
6. `overrides/app/pages/checkout/util/address/` — validation, filters, `extractShippingAddress`  
7. `overrides/app/hooks/use-contentstack-shipping-fields.jsx` — CMS → form metadata  
8. `overrides/app/components/_app-config/index.jsx` — `siteId` / `locale` on `CommerceApiProvider`  

---

*Generated from codebase review; behavior may change with feature flags (e.g. `newShippingAddressFlowEnabled`) and Contentstack configuration.*
