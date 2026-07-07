# Project Log — Nuuk Price Tracker

Running record of what this dashboard is, how it's built, and what's changed over time. Kept in the repo (not just Claude's private memory) so anyone opening the project — human or AI — can get up to speed without digging through `git log`.

## What this is

A single-file static dashboard (`index.html` — HTML + CSS + vanilla JS, no build step, no framework) for tracking price changes across Nuuk's sales channels (Amazon, Swiggy Instamart, Blinkit, Website). Deployed on Vercel at price-change-tracker.vercel.app, auto-deploying from the `main` branch of `anshulpanda06/price-change-tracker` on GitHub.

## Architecture

- **Frontend/only app**: `index.html`. Tabs: Latest Changes, Full Log, Price vs Channels, Channel Parity, Master SKU.
- **Backend**: Supabase project `numkcasfacwcwpvtipob`, accessed client-side via an anon key embedded in the page (no server, no auth beyond that — this is a trusted-team internal tool). Two tables + one storage bucket:
  - `channel_upload_meta` table + `channel-uploads` storage bucket — metadata and files for the per-channel price-parity uploads (Amazon buybox report, Swiggy/Blinkit/Website dumps).
  - `price_log` table — every price-change log entry. Added 2026-07-06; before that, the log was a hardcoded in-memory JS array (`SEED_LOG_RAW`) that reset on every page refresh and never synced between users. Has a nullable `batch_id` column (added 2026-07-06) grouping rows created together as one multi-channel update.
- `MASTER_SKUS` — a hardcoded array of {product, colour, sku, asin, swiggyId, blinkitId}, the canonical product/colour list driving all dropdowns and fuzzy matching.
- Creating or altering Supabase tables requires the SQL editor in the Supabase dashboard — the anon key can't run DDL, so schema changes need Anshul to run SQL by hand (see `CLAUDE.md`).

## Changelog

### 2026-07-06 — Replaced email SOP with message parser + admin mode; added Supabase persistence
Previously, price changes were logged by emailing a structured message to `prices@nuuk.in`, which someone then re-typed into the dashboard's Add-entry form — too much manual overhead. Changes:
- **Removed**: the email SOP banner, CC-email lock feature, "email template" modal, and dead Google-Sheets push/fetch stubs that were never actually wired up.
- **Added — message parser**: a "+ Paste update" button opens a modal where you paste a loosely-formatted update (e.g. "Lit Pro: Blue 4299, Grey 4199, White 3999"). `parseMessage()` splits it into candidate price changes, fuzzy-matches product/colour names against `MASTER_SKUS` (normalize + token-overlap + Levenshtein scoring), and produces editable draft rows — nothing saves until reviewed and confirmed. Lines mentioning a product with no price change, or lines that don't parse at all, are shown separately for transparency rather than silently dropped or guessed at.
- **Added — Supabase persistence for the price log**: `price_log` table, with `bootstrapPriceLog()` seeding it once from the old hardcoded array on first run, and all add/edit/delete/lock actions now read and write through it. Falls back to local read-only data if the table is unreachable, rather than breaking the page.
- **Added — password-gated Admin mode**: a header "Admin" button (password `admin123`, client-side only, same trust model as the rest of this static site) now gates Edit/Delete/lock-toggle on the Full Log tab. Without it, the log is read-only aside from adding new entries.
- **Added deploy automation**: pushes to `main` now go straight to production (Vercel auto-deploys from GitHub) — see `CLAUDE.md` for the standing git workflow instruction.

### 2026-07-06 (later) — Fixed Price vs Channels sorting and the product filter slicer
The "Price vs Channels" tab wasn't sorted by default and had no way to sort by price or deviation — only Product/Color/Dev % headers existed and Dev % was the only one wired to real sort logic. Separately, the product-filter chips at the top (shared across all tabs) were built only from `priceLog`'s product names, but Tabs 3/4 render from the literal product text in uploaded channel sheets — a different, uncontrolled string per source (e.g. priceLog/MASTER_SKUS say "STROM", but the uploaded Amazon sheet literally says "STROM GO", and MASTER_SKUS separately has "Strom GO V2" for a different SKU generation). So real products on Tab 3/4 had no matching chip at all, and clicking "STROM" filtered to nothing since no row's `product` field equaled that exact string.
- **Fixed**: `buildProductChips()` now unions product names from `priceLog`, `MASTER_SKUS`, and all four `channelData` sources, so every literal product string appearing anywhere in the app gets its own chip (sorted alphabetically) and exact-match filtering actually works against whichever tab you're on.
- **Added**: Tab 3 now sorts alphabetically by product by default (`sortState.t3` initialized to `{col:'product',dir:1}`, same for Tab 4), and "Approved Price" is now a sortable column (`sortTable('t3','price')`) — click again to flip high-to-low/low-to-high, same as the existing Dev % column.
- Deliberately did *not* try to merge "STROM"/"STROM GO"/"Strom GO V2" into one unified product — they're genuinely different literal strings from different data sources, and merging them would require a real product-name reconciliation pass across the sheets and master data, not a UI fix.

### 2026-07-06 (later still) — Admin: batch-aware editing, SKU rename from Latest Changes, plus three small UX fixes
Two related asks: (1) editing an entry in Full Log that was originally added as "All channels" required repeating the edit 4 times, once per channel row; (2) there was no way to correct a SKU (rename, or fix a mis-entered price) directly from the Latest Changes view — only Full Log's single-row edit existed.
- **Added `batch_id`** (new nullable column, migration run by Anshul) — `commitLogEntry()` now stamps every row created together for the same colour across multiple channels with one shared batch id. Rows added before this exist without one and fall back to the old single-row edit behavior — no retroactive guessing of which historical rows "belong together".
- **Batch-aware edit drawer**: editing a row that has siblings (admin mode only) shows a checkbox, checked by default, to apply the same field changes (product, colour, prices, dates, approver, reason — everything except channel) to every sibling row from that update in one save.
- **SKU edit from Latest Changes**: admins get an "✎ Edit" button per row. Renaming Product/Colour relabels *every* log entry for that SKU, past and present, across all channels (a rename is an identity correction, not a price event, so it must be retroactive to keep history consistent under the new name). Editing a single channel's price there only updates that channel's latest entry.
- **Fixed**: "Last Changed" column header on Latest Changes was right-aligned while its date cells were left-aligned — now both left-align.
- **Kept and improved** the "Sync" header button — now re-fetches the price log itself in addition to channel files, so it actually pulls in a teammate's edits without a full page reload (previously it only refreshed channel-parity uploads).
- **Changed default sort**: Full Log now opens sorted descending by Input Date (most recently entered first) instead of unsorted insertion order.
- Verified the batch-edit and SKU-rename flows against production Supabase using a disposable `ZZTEST` SKU created and deleted for the purpose, after an earlier verification pass accidentally mutated a real SKU ("BFF / Tokyo Totti Candy" got renamed and re-priced mid-testing) — reverted immediately, but going forward test writes use disposable rows instead.

### 2026-07-07 — Replaced the Price Increases/Reductions cards with an inline price-trend chart
The two secondary stat cards on Latest Changes ("Price Increases", "Price Reductions") were replaced with a compact, always-visible price-trend chart — Product/Colour/Channel selects driving the same canvas-based line chart that already existed behind each row's "Chart" button, now surfaced by default instead of requiring a click per SKU.
- **Refactored `drawChart()`** to take optional `canvasId`/`legendId`/`heightOverride` params (defaulting to the existing modal's `chartCanvas`/`chartLegend`/300px) so the same renderer draws both the full-size modal chart and this new compact inline one — no duplicated charting logic.
- **New trend controls** (`trendProduct`/`trendColor`/`trendChannel` selects) default to the most-recently-changed SKU on first load so the chart is never blank; picking a different product repopulates the colour dropdown from that product's actual logged colours. Shows the plotted date range (e.g. "20 May 2026 – 30 Jun 2026") next to the filters.
- **Active SKUs** card stays but shrunk to a small fixed-width card next to the chart instead of a full-width stat card — removed the now-dead `.stats-row`/`.stat-card` CSS along with it.
- Hit a flexbox sizing bug during this — `flex:1` + `height:150px` on a child with no bounded ancestor height caused the whole row to balloon to ~800px tall. Fixed by giving `.trend-row` an explicit `height:200px` so descendants have something concrete to size against.

### 2026-07-07 (later) — Step-line rendering for price charts
Both the per-row chart modal and the new inline trend chart were drawing diagonal lines between price-change points, implying a gradual ramp between log entries — misleading, since a price is flat until the exact day it changes, then jumps. Changed `drawChart()`'s stroke and fill paths to a step-after shape (flat horizontal segments, sharp vertical jumps only on actual change dates), matching the reference step-chart style Anshul shared. Applies to both chart surfaces since they share the same renderer.

### 2026-07-07 (later still) — Charts now always extend to today
The flat "still at this price" segment after a SKU's last logged change was drawing past the chart's own x-axis bounds — the axis range was computed from logged dates only, so `xScale(today)` extrapolated beyond the plot area instead of landing inside it, producing a stray overflow block at the right edge. Fixed by including today in the axis's max-date calculation, so the line correctly draws flat all the way to a labeled "Today" tick at the right edge, starting from the SKU's first logged change on the left. Updated the inline trend chart's date-range label to match (always "first change – today", not "first – last change").

### 2026-07-07 (later still) — Hover tooltips on price charts
Neither chart (per-row modal or the inline trend chart) showed the exact price at a point — you had to eyeball it against the y-axis gridlines. Added hover tooltips to both: `drawChart()` now records each plotted dot's on-screen position alongside its price/date/channel in a `chartPoints` map keyed by canvas id, and a shared `attachChartHover()` (wired up once at page load) finds the nearest recorded point to the cursor and shows "₹4,399 / Website · changed 20 May 2026" — the exact price and the date it changed to that price, not the date hovered.
- Tooltip uses `position:fixed` anchored to the point's actual screen coordinates (not `position:absolute` inside the chart container) — the compact trend card has `overflow:hidden` for its own layout reasons, which was clipping an absolutely-positioned tooltip; fixed positioning sidesteps that entirely.

### 2026-07-07 (later still) — Onboarding hint on Latest Changes
Full Log already had a short banner explaining the two ways to log a price change, but Latest Changes — the tab a brand-new user actually lands on — had none. Added a matching 2-line hint banner there: paste a message into "+ Paste update" for parsing, or use "+ Add entry" for a single manual change. Moved it (id `t1HintBanner`) to sit directly under the tab headings, above the trend chart and product filter chips, so it's the first thing visible — hidden/shown alongside `t1StatsWrap` via the same per-tab visibility toggle in `switchTab()`.

## Keeping this log current

Whenever a change is made to this project, add a dated entry above (or extend the latest one if it's the same work session) describing what changed and why — not just what, since "why" is what stops future work from re-litigating settled decisions. This is a manual step performed as part of each change, not an automated script — summarizing a diff meaningfully requires understanding the change, which is why this file gets updated at the same time the code does rather than by a hook.
