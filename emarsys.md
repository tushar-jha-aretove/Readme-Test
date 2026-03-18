# Emarsys integration — knowledge transfer (KT)

> **Note:** This project does not contain anything named “Immerses.” The marketing integration in the codebase is **Emarsys** (SAP Emarsys — email, forms, Web Extend / Scarab). This document walks through **how Emarsys is wired end-to-end** so you can follow it like a story: user action → UI → hook → API → Emarsys.

---

## 1. What Emarsys does here (in one paragraph)

Emarsys is used for:

1. **Newsletter / marketing opt-in** — When users subscribe (footer, expert advice, account registration, checkout), the storefront sends their data to **your backend**, which talks to the **Emarsys API** to create or update a **contact** and **trigger an Emarsys form** (so journeys / double opt-in / compliance can run in Emarsys).
2. **Web Extend (Scarab)** — On the **cart page**, a third-party script loads and the app pushes **cart contents** (and logged-in user email) into `ScarabQueue` so Emarsys can personalize recommendations and track behavior.

---

## 2. High-level architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  BROWSER (React storefront)                                                  │
│  • Forms call useEmarsys().subscribeToNewsletter()                           │
│  • POST /api/emarsys/subscribers + Bearer (Shopper token)                    │
│  • Cart page: Scarab script + useEmarsysWebExtendTrackCart()                 │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  CUSTOM API (Express) — src/api/main.js                                      │
│  • Mounts router at /api/emarsys                                             │
│  • POST /subscribers → EmarsysSubscriberService.createOrUpdateSubscriber()   │
│  • EmarsysError → 400 + { error, type }                                      │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Emarsys API (suite11.emarsys.net/api/v2/)                                   │
│  • WSSE auth (EMARSYS_USER + EMARSYS_SECRET)                                 │
│  • contact/getdata, contact/, form, form/{id}/trigger                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Per-site config** (language / country for Emarsys) lives in `config/sites.js` under each site’s `emarsys: { languageTag, countryCode }`.

**Secrets / IDs:**

| Variable | Role |
|----------|------|
| `EMARSYS_USER` | API user for WSSE |
| `EMARSYS_SECRET` | Used to build `PasswordDigest` in `X-WSSE` header |
| `EMARSYS_MERCHANT_ID` | Scarab / Web Extend script URL (`cdn.scarabresearch.com/js/{merchantId}/scarab-v2.js`) |

---

## 3. Story A — Footer newsletter (“Subscribe”)

### 3.1 Where the user clicks

- **Component:** `overrides/app/components/footer/components/newsletter-form.tsx`
- **UI:** Email field + submit button (“Enter your email” + CTA).

### 3.2 What happens on submit

1. User enters email and clicks submit.
2. **GA:** `gaTracker.footer.trackNewsletterSubmit()` runs.
3. **`useEmarsys`** (same file) provides `subscribeToNewsletter`.
4. **`subscribeToNewsletter`** is called with:
   - `email`
   - `checkboxState: true` (footer is always “I want newsletter”)
   - `formName: EmarsysFormNames.Footer` → string **`"Footer"`**

### 3.3 Frontend hook — `useEmarsys`

**File:** `overrides/app/hooks/use-emarsys.ts`

| Step | What happens |
|------|----------------|
| 1 | `getTokenWhenReady()` — gets **Shopper JWT** from Commerce SDK. |
| 2 | Reads `site.emarsys.languageTag` and `site.emarsys.countryCode` from **current site** (`useMultiSite`). |
| 3 | **POST** to `URLS.emarsysSubscribers` → **`/api/emarsys/subscribers`** (see `overrides/app/constants.js`). |
| 4 | Body: `email`, `languageTag`, `checkboxState`, `formName`, optional `firstName`, `lastName`, `countryCode`, address fields if present. |
| 5 | Header: `Authorization: Bearer <token>`. |

**Returns:** `response.data` on success.

### 3.4 On success (footer)

- **`legacyNavigator.navigateToNewsletterThankYouPage()`** — thank-you page.

### 3.5 On error (footer)

- If API returns `type === EMARSYS_ERROR_TYPES.ACCOUNT_EXISTS` → show “email already exists” style message.
- Else → generic API error on the email field.

**Special backend rule for Footer:** If the contact **already exists** in Emarsys **and** is already opted in (`optIn === '1'`), the server throws **`EmarsysError`** with `ACCOUNT_EXISTS` — so duplicate footer signups are blocked with a friendly message.

---

## 4. Story B — Expert Advice newsletter

### 4.1 Entry point

- **Page block:** `expert-advice-newsletter.tsx` renders **`ExpertAdviceNewsletterForm`**.
- **`formName`** comes from **CMS / Contentstack** (`ExpertAdviceNewsletterContent.formName`) — it must match a **form name** configured inside Emarsys (same as Footer / Checkout naming).

### 4.2 Flow

Same as footer, but:

- `formName` is **dynamic** (`emarsysFormName` prop), not hardcoded `Footer`.
- Thank-you: `navigateToNewsletterThankYouPage(emarsysFormName)` — can vary by form.

**File:** `overrides/app/pages/category/expert-advice/partials/expert-advice-newsletter-form.tsx`

---

## 5. Story C — Account registration + newsletter

### 5.1 Where

**File:** `overrides/app/hooks/use-register.ts`

### 5.2 Flow (story)

1. User fills registration (name, email, password, optional “subscribe to newsletter”).
2. **`register.mutateAsync`** — creates the customer in **Salesforce Commerce** first.
3. If **`subscribeToNewsletter`** is checked:
   - **`subscribeToEmarsysNewsletter`** runs **in the background** (not awaited).
   - Payload includes `firstName`, `lastName`, `email`, `checkboxState: true`, `formName: EmarsysFormNames.Account` → **`"Account"`**.
4. Errors are **swallowed** (`catch { }`) — registration still succeeds even if Emarsys fails.

---

## 6. Story D — Checkout shipping step + newsletter

### 6.1 Where

**Primary:** `overrides/app/pages/checkout/partials/shipping-address/shipping-address-step.tsx`  
(Legacy parallel: `shipping-address.jsx`)

### 6.2 Checkbox behavior

**File:** `shipping-address-selection.tsx` (and `.jsx` variant)

When the user saves the shipping form:

- `checkboxState = form.getFieldState('c_subscribeToNewsletter').isDirty ? filteredAddress.c_subscribeToNewsletter : null`
- Meaning: if the user **never touched** the newsletter checkbox, `checkboxState` is **`null`** (backend uses this in opt-in logic). If they touched it, value is boolean.

### 6.3 After address is saved successfully

1. If newsletter checked → `setIsNewsletterCheckboxChecked(true)` (for analytics / datalayer elsewhere).
2. **`subscribeToNewsletter({...})`** is called with:
   - Email from address / customer
   - Spread **`restAddress`** (firstName, lastName, countryCode, address fields, etc. — whatever the form maps to Emarsys field keys)
   - `checkboxState` as above
   - `formName: 'Checkout'` (string; should align with `EmarsysFormNames.Checkout`)
3. Wrapped in **try/catch** — failures are **ignored** (checkout continues).
4. Then **`goToNextStep()`**.

---

## 7. Backend deep dive — one request, full pipeline

### 7.1 Route registration

**File:** `src/api/main.js`

- `app.use('/api/emarsys', emarsysController)`
- Global error handler: **`EmarsysError`** → **400** + `{ error, type }`.

### 7.2 Controller

**File:** `src/api/modules/emarsys/emarsys.controller.js`

| Method | Path | Handler |
|--------|------|---------|
| POST | `/subscribers` (full path `/api/emarsys/subscribers`) | Parses `req.body`, `new EmarsysSubscriberService()`, `createOrUpdateSubscriber(data)`, returns `{ success: true }`. |

> Code comment: body validation is still TODO. There is also a TODO that the endpoint should verify the shopper matches the token.

### 7.3 `EmarsysSubscriberService.createOrUpdateSubscriber(data)`

**File:** `src/api/modules/emarsys/emarsys.subscriber.service.js`

| Step | Function / action | Purpose |
|------|-------------------|---------|
| 1 | `getContact(email)` | Load existing Emarsys contact or null. |
| 2 | Split `checkboxState`, `formName` from rest → **address** fields. |
| 3 | **Footer guard:** if `formName === 'Footer'` and contact exists with `optIn === TRUE` → throw **`EmarsysError` (`ACCOUNT_EXISTS`)**. |
| 4 | `getContactData({ address, accountExists, optIn, checkboxState })` | Builds field map + opt-in value per rules (incl. Germany-specific behavior). |
| 5 | `createOrUpdateAccount(contactData)` | PUT Emarsys `contact/` with `create_if_not_exists=1`. |
| 6 | `getForms()` | GET Emarsys `form` — list of forms. |
| 7 | Find form where `name === formName` → get `id`. |
| 8 | If no form → throw Error (message in code references Checkout even when `formName` is different — worth knowing when debugging). |
| 9 | `submitForm(formId, email)` | POST `form/{id}/trigger` — ties contact to that form in Emarsys (e.g. double opt-in flow). |

### 7.4 `EmarsysService` (API orchestration)

**File:** `src/api/modules/emarsys/emarsys.service.js`

| Method | Emarsys endpoint | Role |
|--------|------------------|------|
| `getContact` | POST `contact/getdata` | Returns contact fields or `null` if not found (error code 2008). |
| `createOrUpdateAccount` | PUT `contact/?create_if_not_exists=1` | Upsert contact. |
| `getForms` | GET `form` | List forms by name. |
| `submitForm` | POST `form/{formId}/trigger` | Trigger form for email. |

### 7.5 `EmarsysApiService` (HTTP + auth)

**File:** `src/api/modules/emarsys/emarsys.api.service.js`

- **Base URL:** `https://suite11.emarsys.net/api/v2/`
- Every request sends header **`X-WSSE`** with **UsernameToken** (nonce, timestamp, SHA1 digest of `nonce + timestamp + EMARSYS_SECRET`).

### 7.6 Contact payload & opt-in logic

**File:** `src/api/modules/emarsys/emarsys.utils.js`

- **`getContactData`** maps incoming address keys to **Emarsys numeric field IDs** from `EMARSYS_FIELD_IDS` (email=3, optIn=31, languageTag=4342, countryCode=14 mapped via `EMARSYS_COUNTRY_CODES`, etc.).
- **`getOptInState`** decides opt-in:
  - New contact + checkbox checked → usually **TRUE** opt-in; **Germany (`DE`)** can use **EMPTY** for compliance nuances.
  - Existing contact + checkbox checked → depends on country and current `optIn`.
  - Unchecked → **FALSE** for new contacts.
  - If opt-in state is `null`, the opt-in field may not be sent.

**Constants:** `src/api/modules/emarsys/emarsys.constants.js` — field IDs, form names (`Checkout`, `Account`, `Footer`), opt-in values (`'1'` true, `'2'` false).

---

## 8. Story E — Cart page (Web Extend / Scarab)

### 8.1 Load the script

**Where:** `overrides/app/components/_app/index.jsx`

- When **`isCartPage`** is true, render **`<EmarsysWebExtendScript merchantId={EMARSYS_MERCHANT_ID} />`**.

**Component:** `overrides/app/components/emarsys-web-extend/emarsys-web-extend-script.tsx`

- Injects **deferred** script: `//cdn.scarabresearch.com/js/{merchantId}/scarab-v2.js`
- Script id: `scarab-js-api` (avoids double injection).

### 8.2 Push cart + email

**Where:** `overrides/app/pages/cart/index.jsx` → inside cart component:

```ts
useEmarsysWebExtendTrackCart(basket, customer)
```

**Hook:** `overrides/app/components/emarsys-web-extend/use-emarsys-web-extend-track-cart.ts`

| Step | What happens |
|------|----------------|
| 1 | Tracks **product line items** in state; only re-runs cart push when items **change** (ids/qty/price). |
| 2 | If no items or no `window`, return. |
| 3 | If **`window.ScarabQueue`** missing (script not loaded yet), return silently. |
| 4 | If customer is **registered** and has **email** → `ScarabQueue.push(['setEmail', email])`. |
| 5 | `ScarabQueue.push(['cart', [...]])` — each line: `item` (productId), `price`, `quantity`; bonus lines excluded. |
| 6 | `ScarabQueue.push(['go'])` — flush to Emarsys. |

Errors are caught and logged as **`EmarsysWebExtend: error tracking cart data`**.

---

## 9. Function / file map (quick reference)

| Concern | File | Symbol |
|---------|------|--------|
| Shopper → API newsletter call | `overrides/app/hooks/use-emarsys.ts` | `useEmarsys`, `subscribeToNewsletter` |
| Form name enum (frontend) | `overrides/app/types/emarsys.ts` | `EmarsysFormNames` |
| API URLs | `overrides/app/constants.js` | `URLS.emarsysSubscribers` |
| Express route | `src/api/modules/emarsys/emarsys.controller.js` | POST `/subscribers` |
| Orchestration | `src/api/modules/emarsys/emarsys.subscriber.service.js` | `EmarsysSubscriberService` |
| Emarsys REST calls | `src/api/modules/emarsys/emarsys.service.js` | `EmarsysService` |
| Auth + axios | `src/api/modules/emarsys/emarsys.api.service.js` | `EmarsysApiService` |
| Field IDs & form names | `src/api/modules/emarsys/emarsys.constants.js` | `EMARSYS_FIELD_IDS`, `EMARSYS_FORM_NAMES`, … |
| Opt-in mapping | `src/api/modules/emarsys/emarsys.utils.js` | `getContactData`, `getOptInState`, `EmarsysError` |
| Scarab script | `.../emarsys-web-extend-script.tsx` | `EmarsysWebExtendScript` |
| Cart tracking | `.../use-emarsys-web-extend-track-cart.ts` | `useEmarsysWebExtendTrackCart` |
| Tests (hook) | `overrides/app/hooks/use-emarsys.test.js` | — |

**UI entry points using Emarsys newsletter API:**

| UI | File |
|----|------|
| Footer | `overrides/app/components/footer/components/newsletter-form.tsx` |
| Expert advice | `.../expert-advice-newsletter-form.tsx` |
| Register | `overrides/app/hooks/use-register.ts` |
| Checkout shipping | `.../shipping-address-step.tsx` (+ `.jsx`) |

---

## 10. Operational checklist for you

1. **Form names in Emarsys** must exist and match: **`Footer`**, **`Checkout`**, **`Account`**, plus any **Expert Advice** `formName` from CMS.
2. **Per-site** `emarsys.languageTag` / `countryCode` in **`config/sites.js`** drive defaults in `useEmarsys` when the client doesn’t send overrides.
3. **Footer** is the only flow that **surfaces** `ACCOUNT_EXISTS` to the user; checkout/register either ignore or swallow errors.
4. **Web Extend** only runs on **cart** route in `_app`; cart data is not pushed on other pages by this code.

---

## 11. Glossary

| Term | Meaning |
|------|---------|
| **Emarsys** | Marketing cloud; contacts, forms, automation. |
| **Web Extend / Scarab** | Emarsys JavaScript for onsite personalization and behavioral data. |
| **WSSE** | Auth scheme used by Emarsys API v2 (`X-WSSE` header). |
| **Form trigger** | After updating the contact, the backend triggers a named Emarsys form so the contact enters the right journey (e.g. DOI email). |

---

*Document generated from codebase scan. If you meant a different product than Emarsys, say which name/integration and we can map that separately.*
