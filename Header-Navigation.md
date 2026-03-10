# Content Stack Header & Footer – Data and Rendering Flow

This document walks through how Content Stack data is fetched, transformed, and used to render the **header navigation** and **footer**. Each step includes the relevant file and function names so you can jump into the code.

**Console logs:** For the header flow, step-by-step logs are written with the prefix **`[Header Flow]`**. In the browser console, filter by `[Header Flow]` to see the data at each step and the final `categoryTreeNavigation` map.

---

## Part 1: Header

### 1. Where the header is mounted

The app wraps the main layout with the header. The header is the entry point for all Content Stack header data.

| Step | What happens | Where in code |
|------|----------------|---------------|
| App renders | `Header` is rendered and wraps its children in `HeaderContentstackProvider`. All header Content Stack data is loaded and provided inside this provider. | `overrides/app/components/_app/index.jsx` (Header usage) → `overrides/app/components/header-ts/header.tsx` (Header component and provider) |

**File:** `overrides/app/components/header-ts/header.tsx`  
- The `Header` component wraps content in `HeaderContentstackProvider`, then renders `TopBanner`, optional `LogoBar`, and `HeaderNavigation`.  
- **HeaderNavigation** is only rendered when not on checkout (`!isCheckout`).

---

### 2. Header provider: two Content Stack calls + tree build

All header data is loaded in one place: the header context provider. It makes two main Content Stack queries and then builds the category tree.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 2.1 | **Header entry is fetched** from Content Stack (content type `header`) with references for `navigation_menu`, `logo`, `search_bar`, `top_teal_banner`, `membership_block`, and nested refs (e.g. `navigation_menu.top_level_menu.menu_item_reference`). | `overrides/app/components/header-ts/context/header-provider.tsx` → `useContentstack({ contentType: CONTENT_TYPES.HEADER, includeReference: [...] })` |
| 2.2 | **Categories with `show_in_menu: true` are fetched** from Content Stack (content type `category`). This returns a **flat list** of categories (no hierarchy yet). | Same file → `useContentstackCategories({ showInMenu: true })` |
| 2.3 | **Category tree is built** from the flat category list. Parents, children, and “fake parent” groupings are applied here. | Same file → `buildTree(categories)` |
| 2.4 | **Category IDs for the menu** are derived from the header’s `top_level_menu` (only items that reference a category). For each such category, its subcategory IDs are collected. | Same file → `categoryIds` from `topLevelMenu` + `categoryTreeNavigation` |
| 2.5 | **Menu campaigns** are fetched from Content Stack (content type `menu_campaign`) filtered by those category IDs so campaigns can be shown per category in the drawer. | Same file → `useContentstack({ contentType: CONTENT_TYPES.MENU_CAMPAIGN, query: { taxonomies.category: $in: categoryIds } })` |
| 2.6 | **Membership/loyalty block** from the header entry is transformed and exposed. | Same file → `transformLoyaltyBlockContent(membership_block)` |
| 2.7 | Context value is set: `header`, `categoryTreeNavigation`, `campaigns`, `membershipBlock`. Any child of the provider can read these via `useHeaderContentstack()`. | Same file → `HeaderContentstackContext.Provider value={...}` |

**File:** `overrides/app/components/header-ts/context/header-provider.tsx`  
- **useContentstack** is from `overrides/app/hooks/use-contentstack.tsx` (see step 3).  
- **useContentstackCategories** is from `overrides/app/components/header-ts/hooks/use-contentstack-categories.tsx` (see step 4).  
- **buildTree** is from `overrides/app/utils/categories-utils.js` (see step 5).

---

### 3. How “header” Content Stack data is fetched (useContentstack)

The header entry is loaded via the generic Content Stack hook, which uses the shared API service.

| Step | What happens | Where in code | Log (`[Header Flow]`) |
|------|----------------|---------------|------------------------|
| 3.1 | Hook gets `locale` and `site` (with `site.stack`) from `useMultiSite`. | `overrides/app/hooks/use-contentstack.tsx` | — |
| 3.2 | React Query runs `fetchContentstackData`, which calls `getEntryByQuery` with the given `contentType`, `locale`, `Stack` (from site), and options like `includeReference`. | Same file → `fetchContentstackData` → `getEntryByQuery` | — |
| 3.3 | Content Stack SDK runs the query (content type, locale, references, etc.). Result is an array; for header we take the first entry. | `overrides/app/contentstack-api/services.js` → `getEntryByQuery` | — |
| 3.4 | Hook returns `dataContent` (array) and `loading`. Live preview can refresh this data. | Same file → `dataContent`, `loading` | The **header** result is logged in the provider as **"Content Stack call 1 — HEADER received"** (see Section 2). |

**File:** `overrides/app/hooks/use-contentstack.tsx`  
- **getEntryByQuery** is implemented in `overrides/app/contentstack-api/services.js`: it builds a Content Stack query (`.ContentType(contentType).Query()...`), optionally adds `includeReference`, `referenceIn`, and fallback locale, then executes and returns entries (flattened array).

---

### 4. How categories for the menu are fetched (useContentstackCategories)

Only categories that should appear in the menu are requested.

| Step | What happens | Where in code | Log (`[Header Flow]`) |
|------|----------------|---------------|------------------------|
| 4.1 | Hook gets `locale` and `site.stack` from `useMultiSite`. | `overrides/app/components/header-ts/hooks/use-contentstack-categories.tsx` | — |
| 4.2 | React Query calls `getEntryByQuery` with `contentType: CONTENT_TYPES.CATEGORY` and, when `showInMenu` is true, `query: { 'custom_attributes.show_in_menu': true }`. | Same file → `fetchCategories` | **"Content Stack call 2 — fetching CATEGORIES (show_in_menu: true)"** when the request runs. |
| 4.3 | Content Stack returns a **flat** list of category entries (each may have `parent_category`, `custom_attributes.fake_parent`, `position`, etc.). | `overrides/app/contentstack-api/services.js` → `getEntryByQuery` | **"Content Stack call 2 — categories received"** with `count` and `category_ids` array. |
| 4.4 | Hook returns `{ categories, isLoading, error }`. `categories` is this flat array, later passed to `buildTree` in the header provider. | Same file → return value | Provider logs **"Categories passed to buildTree"** when this data is available. |

**File:** `overrides/app/components/header-ts/hooks/use-contentstack-categories.tsx`  
**Constants:** `CONTENT_TYPES.CATEGORY` = `'category'` in `overrides/app/contentstack-api/constants.js`.

---

### 5. How the category tree is built (buildTree)

The flat category list is turned into a tree: parent/child by `parent_category`, and optional “fake parent” groups.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 5.1 | **Map by UID:** Each category is converted to a node with `category_id`, `uid`, `title`, `path` (slug or category_id), `subCategory: []`, `position`, `navigationImage`, `fakeParent`, `show_in_menu`. Stored in `categoryMap` keyed by `category.uid`. | `overrides/app/utils/categories-utils.js` → `buildTree` (first `reduce`) |
| 5.2 | **Assign to parent or root:** For each category, `parent_category[0].uid` and `custom_attributes.fake_parent` are read. | Same file → `categories.forEach` |
| 5.3 | **Real parent:** If there is a parent UID and no `fake_parent`, the node is pushed onto `categoryMap[parentUid].subCategory`. | Same file → `if (parentUid && categoryMap[parentUid] && !fakeParent)` |
| 5.4 | **Fake parent:** If `fake_parent` is set (e.g. a label like "Pet"), a synthetic parent node is created or reused under the real parent, and the current category is added to that synthetic parent’s `subCategory`. So you get Real Parent → Fake Parent → Category. | Same file → `else if (fakeParent && categoryMap[parentUid])` (fake UID from `fakeParent`, create/find fake node, push current category there) |
| 5.5 | **Top level:** If there is no parent (or no valid parent), the node is added to `tree[category.category_id]`. | Same file → `else { tree[category.category_id] = ... }` |
| 5.6 | **Sort:** All `subCategory` arrays in the tree are sorted by `position` (recursively). | Same file → `sortSubCategories`, then `Object.values(tree).forEach(sortSubCategories)` |
| 5.7 | Return value is a **CategoryMap**: `Record<category_id, CategoryTreeNode>`. Each node has `subCategory` array (and possibly nested `subCategory`). | Same file → `return tree` |

**File:** `overrides/app/utils/categories-utils.js`  
- **Input:** Flat array of Content Stack category entries.  
- **Output:** Tree object keyed by top-level `category_id`; used as `categoryTreeNavigation` in the header context.

---

### 6. How the header navigation uses the data

The header UI reads from the context and from the header entry’s `navigation_menu` and `logo`.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 6.1 | **HeaderNavigation** gets `header` from `useHeaderContentstack()`, and uses `header.navigation_menu` and `header.logo`. It does not use `categoryTreeNavigation` directly; that is used inside the catalog/menu and drawer. | `overrides/app/components/header-ts/components/header-navigation/header-navigation.tsx` |
| 6.2 | **Desktop:** Renders search, **CatalogNavigation** (with `navigationMenu` from header), logo, **TopNavigationMenuContainer** (top-level menu), account, cart. **CatalogNavigation** receives `navigation_menu[0]` (one `CSNavigationMenu` with `top_level_menu`). | Same file → `renderDesktopNavigation` |
| 6.3 | **Mobile:** Renders hamburger, search, logo, cart, and **HeaderMobileDrawer**. The drawer receives the same `navigation_menu` and renders menu items from `top_level_menu`. | Same file → `renderMobileNavigation` |

**File:** `overrides/app/components/header-ts/components/header-navigation/header-navigation.tsx`  
- **TopLevelNavigation** and **CatalogNavigation** get `top_level_menu` from the header’s `navigation_menu[0]`.

---

**What you see (Section 6):** On desktop: search, catalog buttons, logo, account/cart. On mobile: hamburger, search, logo, cart; opening the hamburger shows the drawer (Section 9). No `[Header Flow]` logs here; data comes from context filled in Section 2.

---

### 7. Top-level menu items: links vs categories

Each item in `top_level_menu` has a `menu_item_reference` that can be either an internal link or a category.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 7.1 | **TopNavigationMenuContainer** takes `navigation_menu[0].top_level_menu` and passes it to **TopLevelNavigation**. | `overrides/app/components/header-ts/components/header-navigation/header-navigation.tsx` → `TopNavigationMenuContainer` |
| 7.2 | **TopLevelNavigation** maps over `topLevelMenuItems`. For each item it reads `menu_item_reference[0]`. **Only entries with `_content_type_uid === 'internal_link'`** are rendered; categories are skipped here. So this block shows things like “Account” / “Login” with `target_link_data.url`. | `overrides/app/components/header-ts/components/header-navigation/top-level-navigation.tsx` |
| 7.3 | **CatalogNavigation** also receives `navigation_menu[0]` and maps over `top_level_menu`. Here **only entries with `_content_type_uid === 'category'`** are rendered as catalog buttons (e.g. “Skin Care”, “Body”). Each button has a `categoryId` from `reference.category_id`. Clicking sets `activeCategoryId`, which opens the desktop drawer. | `overrides/app/components/header-ts/components/header-navigation/catalog-navigation.tsx` |

**Files:**  
- `overrides/app/components/header-ts/components/header-navigation/top-level-navigation.tsx` (internal links only).  
- `overrides/app/components/header-ts/components/header-navigation/catalog-navigation.tsx` (categories only, drives drawer).

**What you see:** Top-level nav = links like Account/Login (from internal_link refs). Catalog = buttons like “Skin Care”, “Body” (from category refs in `top_level_menu`). Same `top_level_menu` drives the first-level rows in the mobile drawer (Section 9).

---

### 8. Desktop drawer: tree + campaigns

When a catalog category is active, the drawer shows Level 1 tabs (subcategories) and Level 2 panels (sub-subcategories + campaigns).

| Step | What happens | Where in code |
|------|----------------|---------------|
| 8.1 | **HeaderDesktopDrawer** gets `header`, `categoryTreeNavigation`, and `campaigns` from `useHeaderContentstack()`. It uses `activeCategoryId` (from **CatalogNavigation**) to choose which part of the tree to show. | `overrides/app/components/header-ts/components/header-drawer/header-desktop-drawer.tsx` |
| 8.2 | **Level 1 (tabs):** `categoryTreeNavigation[activeCategoryId]` gives the tree node for the selected top-level category. Its `subCategory` array is rendered as tabs (including fake-parent groups). | Same file → `renderLevel1Navigation` → `categoriesToRender.subCategory` |
| 8.3 | **Level 2 (panels):** For each tab, a **DesktopSubCategoryMenu** is rendered. It receives a `category` (one node from `subCategory`) and renders that node’s `subCategory` as links. So: top-level category → subcategory (tab) → sub-subcategories (links). Campaigns are filtered by category and rendered per panel. | Same file → `renderLevel2Navigation` → `DesktopSubCategoryMenu`; campaigns via `CategoryCampaignContainer` |
| 8.4 | **DesktopSubCategoryMenu** receives a `CategoryTreeNode` and iterates over `category.subCategory` to render **DesktopSubCategoryItem** (link + optional nested list). | `overrides/app/components/header-ts/components/header-drawer/desktop/desktop-subcategory-menu.tsx` |

**Files:**  
- `overrides/app/components/header-ts/components/header-drawer/header-desktop-drawer.tsx` (orchestrates tabs and panels, uses `categoryTreeNavigation` and `campaigns`).  
- `overrides/app/components/header-ts/components/header-drawer/desktop/desktop-subcategory-menu.tsx` (renders one level of `subCategory`).

**What you see:** Clicking a catalog button opens the top drawer. Tabs = `categoryTreeNavigation[activeCategoryId].subCategory` (e.g. Cleansers, Moisturizers, Pet). Each tab’s panel = that node’s `subCategory` as links + campaigns from the `campaigns` array (log #6). No new logs; data is from context.

---

### 9. Mobile drawer: same tree, different UI

The mobile menu also uses the header’s `top_level_menu` and the same `categoryTreeNavigation`.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 9.1 | **HeaderMobileDrawer** gets `header` from `useHeaderContentstack()`, reads `navigation_menu[0].top_level_menu`, and splits into `mainItems` and `lastItem`. | `overrides/app/components/header-ts/components/header-drawer/header-mobile-drawer.tsx` |
| 9.2 | Each item is rendered as **MobileMenuItem**. It checks `menu_item_reference[0]._content_type_uid`: if **internal_link**, it renders a simple link; if **category**, it renders **MobileCategoryMenu**. | `overrides/app/components/header-ts/components/header-drawer/mobile/mobile-menu-item.tsx` |
| 9.3 | **MobileCategoryMenu** gets `categoryTreeNavigation` from context. It looks up `categoryTreeNavigation[category.category_id]` and uses `.subCategory` to decide if there are children. If yes, it renders an accordion and **MobileSubcategoryList** with those `subCategory` nodes. | `overrides/app/components/header-ts/components/header-drawer/mobile/mobile-category-menu.tsx` |
| 9.4 | **MobileSubcategoryList** and **MobileSubcategoryItem** render the nested levels using the same tree nodes (`CategoryTreeNode` with `subCategory`). | `overrides/app/components/header-ts/components/header-drawer/mobile/mobile-subcategory-list.tsx`, `mobile-subcategory-item.tsx` |

**Files:**  
- `overrides/app/components/header-ts/components/header-drawer/header-mobile-drawer.tsx`  
- `overrides/app/components/header-ts/components/header-drawer/mobile/mobile-menu-item.tsx`  
- `overrides/app/components/header-ts/components/header-drawer/mobile/mobile-category-menu.tsx`  

**What you see:** The drawer’s **collapsible rows with chevrons** = each item in `top_level_menu` (log #1’s `topLevelMenuItems`). Rows that are categories expand when tapped; the **expanded list** = `categoryTreeNavigation[category_id].subCategory` (the same tree you see in the FINAL log). Nested levels use the same tree recursively. Campaigns (e.g. “15% OFF exfoliants”) come from log #6. No new `[Header Flow]` logs in these components.

So: **Content Stack header entry** + **Content Stack categories (show_in_menu)** → **buildTree** → **categoryTreeNavigation**; **navigation_menu.top_level_menu** drives which items are links vs categories; **categoryTreeNavigation** drives all nested menu levels (desktop and mobile).

**Log order summary (header):** In the console you should see, in order: **(1)** Content Stack call 1 — HEADER received; **(2)** Content Stack call 2 — fetching CATEGORIES / categories received; **(3)** Categories passed to buildTree; **(4)** buildTree Step 1 (input) → Step 2 (categoryMap) → Step 3 (placement) → Step 4 (tree keys) → Step 5 (full tree); **(5)** Category IDs used for menu campaigns; **(6)** Content Stack call 3 — MENU_CAMPAIGN received; **(7)** FINAL — categoryTreeNavigation (full map). Filter by `[Header Flow]` to see only these.

---

### 10. From built data to what you see in the header (flow + logs → UI)

Once the provider has built `header`, `categoryTreeNavigation`, and `campaigns`, that data flows into the header components and becomes the navigation you see. There are no additional `[Header Flow]` logs in the UI components; the logs above are the full data pipeline. Below is how that data maps to the screen and how to trace it.

#### Logs in order and what they represent

| Order | Log message | What it tells you |
|-------|-------------|--------------------|
| 1 | **Content Stack call 1 — HEADER received** | The header entry from Content Stack: `top_level_menu` items (labels + ref type + category_id). These are the **same items** that will become the first-level rows in the mobile drawer and the catalog buttons on desktop. |
| 2 | **Content Stack call 2 — fetching CATEGORIES** / **categories received** | Flat list of categories (show_in_menu). Count and `category_ids` — this is the raw input to the tree. |
| 3 | **Categories passed to buildTree** | Confirms the flat list is passed into `buildTree`. |
| 4 | **buildTree Step 1 — INPUT** | The flat categories with uid, category_id, title, parent_uid, fake_parent, position. |
| 4b | **buildTree Step 2 — categoryMap built** | All nodes created and keyed by uid. |
| 4c | **buildTree Step 3 — placement** | Which categories went under a real parent, under a fake parent, or as top-level. |
| 4d | **buildTree Step 4 — tree keys** | The top-level `category_id`s (roots). These are the keys of the object you see in FINAL. |
| 4e | **buildTree Step 5 — FULL TREE** | The full `categoryTreeNavigation` object (same structure as FINAL). Keys = top-level category_ids; each value has `subCategory` arrays that may nest further. |
| 5 | **Category IDs used for menu campaigns** | The `categoryIds` array (top-level categories’ direct children). Used to fetch **menu_campaign** entries; those campaigns are what you see in the drawer (e.g. promos per category). |
| 6 | **Content Stack call 3 — MENU_CAMPAIGN received** | Campaigns fetched for those `categoryIds`; `campaignsCount` is how many campaign entries came back. They are shown in the desktop drawer panels and in the mobile drawer. |
| 7 | **FINAL — categoryTreeNavigation (full map sent to header UI)** | The exact object passed to the header via context. This is what the drawer and catalog use to render nested menus. |

#### How the data flows into the UI (no new logs; trace in code)

| Step | Data used | Component | What you see on screen |
|------|-----------|-----------|-------------------------|
| A | `header.navigation_menu[0].top_level_menu` | **HeaderNavigation** passes it to **CatalogNavigation** (desktop) and **HeaderMobileDrawer** (mobile). | **Desktop:** The main catalog buttons (e.g. Skin Care, Body). **Mobile:** The list of **collapsible rows with chevrons** in the drawer — each row is one item from `top_level_menu`. |
| B | For each item: `menu_item_reference[0]._content_type_uid` | **TopLevelNavigation** (desktop) / **MobileMenuItem** (mobile). | **Desktop:** Only `internal_link` items are rendered (e.g. Account, Login) next to the catalog. **Mobile:** Each row is either a link (internal_link) or an expandable section (category). |
| C | For a **category** item: `categoryTreeNavigation[category.category_id]` | **CatalogNavigation** (desktop) uses it implicitly via drawer. **MobileCategoryMenu** (mobile) reads it from context. | **Desktop:** Clicking a catalog button sets `activeCategoryId`; the drawer opens. **Mobile:** Tapping a category row expands it; the **children** come from `categoryTreeNavigation[category_id].subCategory`. |
| D | `categoryTreeNavigation[activeCategoryId].subCategory` (desktop) or `categoryTreeNavigation[category_id].subCategory` (mobile) | **HeaderDesktopDrawer** (tabs + panels) / **MobileSubcategoryList** + **MobileSubcategoryItem**. | **Desktop:** Tabs in the drawer = first level of `subCategory`; each panel shows the next level (links) + campaigns. **Mobile:** The **expanded content** under a row = that category’s `subCategory` (and their `subCategory` recursively). |
| E | `campaigns` (filtered by category) | **CategoryCampaignContainer** (desktop) / **CampaignContainer** (mobile). | Promo blocks in the drawer (e.g. “15% OFF exfoliants”) — driven by the **MENU_CAMPAIGN** data you see in log #6. |

#### What you see vs. what’s in the logs

- **The collapsed rows in the mobile drawer** = `header.navigation_menu[0].top_level_menu` (log #1 shows these as `topLevelMenuItems`: label, refType, category_id).
- **The big object in “FINAL — categoryTreeNavigation”** = the tree used for all nested levels. Its **keys** are the top-level category_ids (e.g. `skin-care`, `body`). Each key’s **value** has a `subCategory` array; those are the items that appear when you expand a category row (mobile) or the tabs/panels (desktop).
- **Categories in the tree** = the same `category_id`s and `uid`s you see in buildTree Step 1 and in the FINAL map. The **order** of rows/tabs follows the order of `top_level_menu` and the `subCategory` arrays (after sorting by `position` in buildTree).
- **Campaigns in the drawer** = the entries returned for **Content Stack call 3 — MENU_CAMPAIGN received** (log #6); they are filtered by the `categoryIds` from log #5 and rendered next to the category links.

So: **logs 1–7** = data load and tree build; **steps A–E** = that same data flowing through components with no extra logs, ending in the header and drawer UI you see. To debug “why does this row show / not show?” use log #1 for top-level items and the FINAL log for the tree; for “why this campaign?” use logs #5 and #6.

---

## Part 2: Footer

### 1. Where the footer is mounted

The footer is rendered only when the user is not on checkout and not on the cart page.

| Step | What happens | Where in code |
|------|----------------|---------------|
| Footer visibility | `Footer` is rendered when `isFooterVisible` is true (`!isCheckout && !isCartPage`). | `overrides/app/components/_app/index.jsx` (e.g. `{isFooterVisible && <Footer />}`) |

**File:** `overrides/app/components/_app/index.jsx`

---

### 2. Footer component and content hook

The footer component gets all Content Stack footer data through a single hook that fetches and transforms the footer entry.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 2.1 | **Footer** calls `useFooterContent()`. | `overrides/app/components/footer/footer.tsx` |
| 2.2 | If the hook returns no content (or empty object), the footer returns `null`. Otherwise it destructures `complianceLinks`, `businessSections`, `socialMedia`, `ourCollection`, `paymentIcons`, `logo` and passes them to the presentational sections. | Same file |

**File:** `overrides/app/components/footer/footer.tsx`

---

### 3. How footer Content Stack data is fetched and transformed

The hook fetches the footer content type and then runs it through a single transformer.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 3.1 | **useFooterContent** calls **useContentstack** with `contentType: CONTENT_TYPES.FOOTER` and an `includeReference` list for: `our_collections_section.section_references`, `our_collections_section.view_all_reference`, `section_references`, `payment_icons_section`, `logo`, `business_sections.section_references`, `social_media_links`, `social_media_section.social_media_links.social_media_link`. | `overrides/app/components/footer/hooks/use-footer-content.tsx` |
| 3.2 | **useContentstack** (same as for header) uses `getEntryByQuery` so the Content Stack SDK returns the footer entry with those references resolved. Result is in `dataContent` (array). | `overrides/app/hooks/use-contentstack.tsx` → `overrides/app/contentstack-api/services.js` → `getEntryByQuery` |
| 3.3 | The hook takes the first entry: `content = dataContent?.[0]`. | `overrides/app/components/footer/hooks/use-footer-content.tsx` |
| 3.4 | **transformFooterContent(content)** converts the raw Content Stack footer entry into the shape the footer UI expects: `complianceLinks`, `businessSections`, `socialMedia`, `ourCollection`, `paymentIcons`, `logo`. | Same file → `overrides/app/utils/contentstack/adapters/footer-content-adapter.ts` → `transformFooterContent` |

**Files:**  
- `overrides/app/components/footer/hooks/use-footer-content.tsx` (fetch + pass to transformer).  
- `overrides/app/utils/contentstack/adapters/footer-content-adapter.ts` (transformer).

---

### 4. What the footer transformer does (transformFooterContent)

The adapter pulls sections from the raw entry and maps them to link lists and structured data.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 4.1 | **Compliance links:** `section_references` (array of links) is mapped with `extractLinkData` → `transformComplianceData` → `complianceLinks`. | `overrides/app/utils/contentstack/adapters/footer-content-adapter.ts` → `transformComplianceData(section_references)` |
| 4.2 | **Business sections:** `business_sections` (array of `{ section_title, section_references }`) is mapped to `{ title, data }` where `data` is each section’s links via `extractLinkData` → `transformBusinessSections` → `businessSections`. | Same file → `transformBusinessSections(business_sections)` |
| 4.3 | **Social media:** `social_media_section` (title + `social_media_links`) is mapped to `{ title, data }` with each link and icon → `transformSocialMediaData` → `socialMedia`. | Same file → `transformSocialMediaData(social_media_section)` |
| 4.4 | **Our collection:** `our_collections_section` (title, `section_references`, `view_all_reference`) is mapped to `{ title, data, viewAllLink }`; category refs give `url` (slug), `title`, etc. → `transformOurCollectionsData` → `ourCollection`. | Same file → `transformOurCollectionsData(our_collections_section)` |
| 4.5 | **Payment icons** and **logo** are passed through (payment icons may be transformed elsewhere). Result is one object: `{ complianceLinks, businessSections, socialMedia, ourCollection, paymentIcons, logo }`. | Same file → `transformFooterContent` return value |

**File:** `overrides/app/utils/contentstack/adapters/footer-content-adapter.ts`  
- **extractLinkData** normalizes a single link to `{ url, title, inNewTab, id, _version }`.  
- No tree or hierarchy: footer is flat sections and link lists.

---

### 5. How the footer renders each section

The footer component maps the transformed content to specific UI components.

| Step | What happens | Where in code |
|------|----------------|---------------|
| 5.1 | **Top:** Logo (`logo?.[0]`) and **Newsletter** (no Content Stack in this doc; may be separate). | `overrides/app/components/footer/footer.tsx` |
| 5.2 | **Middle:** **CollectionSection** (ourCollection title, links, viewAll), **BusinessSection** per item in `businessSections`, **SocialMediaSection** (socialMedia title + links), **LanguageSelector**. | Same file |
| 5.3 | **Bottom:** **ComplianceSection** (complianceLinks), **PaymentIconsSection** (paymentIcons), and copyright text. | Same file |

**File:** `overrides/app/components/footer/footer.tsx`  
Section components live under `overrides/app/components/footer/components/` (e.g. `collection-section`, `business-section`, `social-media-section`, `compliance-section`, `payment-icons-section`).

---

## Summary

| Area | Content Stack content types | Main data flow |
|------|-----------------------------|----------------|
| **Header** | `header`, `category`, `menu_campaign` | Header entry + categories (show_in_menu) → **buildTree** → `categoryTreeNavigation`. Header’s `navigation_menu.top_level_menu` drives top-level items (links vs categories). Tree drives all nested levels and drawer; campaigns are fetched by category IDs and shown in the drawer. |
| **Footer** | `footer` | Single footer entry → **transformFooterContent** → `complianceLinks`, `businessSections`, `socialMedia`, `ourCollection`, `paymentIcons`, `logo` → rendered by footer section components. |

All Content Stack queries go through **getEntryByQuery** (or **getSingleEntryByQuery**) in `overrides/app/contentstack-api/services.js`, and the Stack instance comes from **useMultiSite** (`site.stack`) with locale from **useMultiSite** (`locale.contentstackLocale`).
