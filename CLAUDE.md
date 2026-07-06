# price-change-tracker

Single-file static dashboard: [index.html](index.html) (HTML + CSS + vanilla JS, no build step, no framework).

## Deployment

- Hosted on Vercel at price-change-tracker.vercel.app, connected directly to this GitHub repo (`anshulpanda06/price-change-tracker`). Vercel auto-deploys on every push to `main` — there is no separate deploy step and no Render.com or other backend service involved.
- The only backend is Supabase (project `numkcasfacwcwpvtipob`), accessed client-side from `index.html` via the embedded anon key (`SB` config near the top of the script). It hosts:
  - `channel_upload_meta` table + `channel-uploads` storage bucket — synced channel price-parity file uploads.
  - `price_log` table — the price-change log (added 2026-07-06), replacing the old in-memory-only array.
- Creating/altering tables requires the Supabase SQL editor (dashboard access) — the anon key cannot run DDL. If a future change needs a new table/column, give Anshul the SQL to run there.

## Git workflow (standing instruction, given 2026-07-06)

Anshul wants changes to this repo shipped automatically: after making and verifying changes to `index.html`, commit and push to `origin/main` without asking for confirmation each time — Vercel picks it up from there automatically. This pre-authorization is specific to this repo. Still:
- Verify changes (syntax check / browser check) before pushing.
- Use clear commit messages describing the change.
- Mention in the response that you pushed, so it's not a silent action.
- This does not extend to force-pushes, history rewrites, or branch deletion — those still require explicit confirmation.
- Before pushing, add or extend a dated entry in [PROJECT_LOG.md](PROJECT_LOG.md) describing what changed and why, and include it in the same commit. This is what keeps that log "auto-updating" — it's a step in every change, not a separate script.
