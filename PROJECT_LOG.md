# Project Log — Nuuk Price Tracker

Running record of what this dashboard is, how it's built, and what's changed over time. Kept in the repo (not just Claude's private memory) so anyone opening the project — human or AI — can get up to speed without digging through `git log`.

## What this is

A single-file static dashboard (`index.html` — HTML + CSS + vanilla JS, no build step, no framework) for tracking price changes across Nuuk's sales channels (Amazon, Swiggy Instamart, Blinkit, Website). Deployed on Vercel at price-change-tracker.vercel.app, auto-deploying from the `main` branch of `anshulpanda06/price-change-tracker` on GitHub.

## Architecture

- **Frontend/only app**: `index.html`. Tabs: Latest Changes, Full Log, Price vs Channels, Channel Parity, Master SKU.
- **Backend**: Supabase project `numkcasfacwcwpvtipob`, accessed client-side via an anon key embedded in the page (no server, no auth beyond that — this is a trusted-team internal tool). Two tables + one storage bucket:
  - `channel_upload_meta` table + `channel-uploads` storage bucket — metadata and files for the per-channel price-parity uploads (Amazon buybox report, Swiggy/Blinkit/Website dumps).
  - `price_log` table — every price-change log entry. Added 2026-07-06; before that, the log was a hardcoded in-memory JS array (`SEED_LOG_RAW`) that reset on every page refresh and never synced between users.
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

## Keeping this log current

Whenever a change is made to this project, add a dated entry above (or extend the latest one if it's the same work session) describing what changed and why — not just what, since "why" is what stops future work from re-litigating settled decisions. This is a manual step performed as part of each change, not an automated script — summarizing a diff meaningfully requires understanding the change, which is why this file gets updated at the same time the code does rather than by a hook.
