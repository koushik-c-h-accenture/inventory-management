# Frontend Known Issues

> Identified via comprehensive audit on 2026-03-23. These are documented for future resolution — no changes have been made yet.

---

## Critical Bugs

### C1. `Spending.vue` — Charts crash with zero/empty data
- `maxRevenueValue` computed: `Math.max(...[])` returns `-Infinity` when data is empty → Y-axis renders `-Infinity`
- `getBarHeight()` hardcoded max of `25000` → cost bars overflow if monthly costs exceed $25K
- Y-axis labels (lines 83–88) hardcoded as `$25K / $20K / $15K / $10K / $5K` — not data-driven
- **Fix needed:** Add empty-array guard to `maxRevenueValue`; introduce `maxCostValue` computed from actual data; update Y-axis labels dynamically

### C2. `Dashboard.vue` — Donut chart circumference constant mismatch
- `getCircleSegment()` multiplies by `440` but SVG `stroke-dasharray` total is `408` (`2 * π * 65 ≈ 408`)
- **Fix needed:** Change multiplier in `getCircleSegment` from `440` to `408`

### C3. `Restocking.vue` — Category filter completely ignored
- `selectedCategory` is never imported from `useFilters()`, never applied to recommendations, and absent from the `watch`
- Changing the Category dropdown has zero effect on the Restocking page
- **Fix needed:** Import `selectedCategory`, add client-side `filteredRecommendations` computed, extend watch to `[selectedLocation, selectedCategory]`

---

## High-Impact UX Bugs

### U1. `FilterBar.vue` — Months hardcoded to year 2025 only
- All 12 `<option>` elements have hardcoded `value="2025-XX"` (lines 9–20)
- Users cannot select any 2026 months — the current year
- **Fix needed:** Replace static options with `v-for` over a dynamic `availableMonths` computed (last 24 months from today)

### U2. `Inventory.vue` — No empty state when search/filter returns zero results
- When `filteredItems` is empty, the table renders with just a header row — no message
- Users cannot tell if data is still loading or if nothing matches
- **Fix needed:** Add `v-if="filteredItems.length === 0"` empty-state div; wrap `<table>` in `v-if="filteredItems.length > 0"`; add `inventory.noResults` i18n key

---

## i18n Gaps

### I1. `App.vue` — "Reports" and "Restocking" nav tabs not translated (lines 26, 29)
- Hardcoded English; breaks on Japanese locale switch
- **Fix needed:** Add `nav.reports` / `nav.restocking` keys to locale files and use `t()` in template

### I2. `BacklogDetailModal.vue` — Multiple hardcoded English strings
- "Inventory Shortage Details", "Priority", "Backordered", "Close" — all hardcoded
- `formatDate` uses hardcoded `'en-US'` locale instead of `currentLocale` from composable
- **Fix needed:** Add `backlog.modal.*` keys; replace strings with `t()` calls; fix `formatDate` locale

### I3. `CostDetailModal.vue` — Cost category labels hardcoded
- "Procurement", "Operational", "Labor", "Overhead", "Cost Breakdown", "Close" — all hardcoded
- **Fix needed:** Re-use existing `finance.monthlyCostFlow.*` keys from locale files; add any missing keys

### I4. `InventoryDetailModal.vue` — All field labels hardcoded in English
- "Quantity on Hand", "Stock Level", "Category", "Location", "Reorder Point", etc.
- `formatDate` uses hardcoded `'en-US'` locale
- **Fix needed:** Re-use `inventory.table.*` keys; add missing keys; fix locale in `formatDate`

---

## Code Quality

### Q1. `Orders.vue:91` — `v-for` using array index as key for order items
- `<div v-for="(item, idx) in order.items" :key="idx">` — can cause incorrect DOM reuse
- **Fix needed:** Use composite key `:key="\`${order.id}-${item.name}\`"`

### Q2. `formatDate` calls `useI18n()` inside function body (not at setup root)
- Affects `Orders.vue`, `Dashboard.vue`, `Demand.vue`
- Composables should only be called at the root of `setup()`, not inside nested functions
- **Fix needed:** Destructure `currentLocale` at setup root; close over it in `formatDate`

### Q3. `Reports.vue` — Multiple issues (no full rewrite needed, surgical fixes only)
- Hardcoded `fetch('http://localhost:8001/api/...')` — should use `api.js`
- All variables use `var` instead of `const`/`let`
- ~15 `console.log` statements left in production code
- Hardcoded `$` currency symbol — doesn't respect `currentCurrency`
- **Fix needed:** Replace `fetch` with `api.*` calls; swap `var` → `const`/`let`; guard logs with `import.meta.env.DEV`; use dynamic currency symbol

### Q4. `Backlog.vue:96` — `inventoryItems` ref name doesn't communicate intent
- Fetched only to build a SKU set for cross-filtering; the name doesn't convey this
- **Fix needed:** Rename to `inventoryForSkuFilter` and add an explanatory comment

---

## Lower Priority / Out of Scope Now

| Issue | Why deferred |
|---|---|
| `useFilters.js` singleton pattern (test pollution) | No runtime impact, architectural change |
| Route guards / lazy-loaded routes in `main.js` | Architectural, no user-visible bug |
| Full `Reports.vue` rewrite to Composition API | Large scope, separate task |
| Accessibility (aria-labels on buttons/modals) | Separate accessibility pass |
| Pinia for shared submitted orders state | Architectural, requires new dependency |
| API request cancellation / timeouts | No current user-visible failures |
| `useFilters` month format validation | Defensive only, no current breakage |
