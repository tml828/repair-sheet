# Repair Sheet — Project Notes

## Repos
- **repair-sheet**: `tml828/repair-sheet` — GitHub Pages, single-file app
- **taxvault**: `tml828/taxvault` — separate repo, do NOT touch from this session

## Deploy
- Push directly to `main`: `git push origin main`
- GitHub Pages serves `repair_sheet.html` at the repo root
- No CI/CD workflow — `.github/workflows/deploy.yml` was deleted to avoid merge conflicts
- Branch for Pages must be set to `main` in GitHub Pages settings

## Architecture
- Single file: `repair_sheet.html` — never rename it
- No npm, no build step, no CDN scripts (Supabase SDK removed)
- Supabase via raw `fetch()` REST calls only (see `sbFetch()`)
- localStorage keys: `repairSheet_v3` (repair sheet), `apexWalkthrough` (walkthrough)

## Supabase
- Project URL: `https://ehlipciasutoderxgptx.supabase.co`
- Anon key: stored in `SB_KEY` var in the HTML file
- Table: `walkthroughs` — columns: `id uuid pk`, `address text`, `city text`, `date text`, `items jsonb`, `created_at timestamptz`
- Table: `app_state` — whole-app cross-device sync: `id uuid pk default gen_random_uuid()`, `sheet jsonb`, `walkthrough jsonb`, `orders jsonb`, `sheet_at bigint`, `walkthrough_at bigint`, `orders_at bigint`
- RLS: anyone can read/insert/update on both tables (required for contractor link + sync codes)

## Cloud Sync
- "Sync" button in tab bar → `openSyncModal()`; sync code = `app_state` row UUID, stored in localStorage `apexCloudSync`
- `cloudMark(section)` called from save()/saveWt()/saveOd(); debounced 2s push, pull-before-push
- Per-section `_at` timestamps: newest wins per tab, never whole-row clobber
- Pull on load + `visibilitychange`

## Key Features
1. **Repair Sheet tab** — checklist with notes, photos, categories; saved to localStorage
2. **Walkthrough tab** — independent from Repair Sheet; add items with photos + notes
   - "Send to Contractor" → PATCHes existing Supabase row (same UUID = same link) or POSTs new
   - Shows bottom sheet modal with Text / Share / Copy options (iOS-safe — no async before share)
   - Contractor opens `?wt=UUID` URL, sees live checklist, checks items off → PATCH to Supabase
   - Owner refreshes → `syncFromSupabase()` merges by `doneAt` timestamp (newest wins)
   - Photos: tap to zoom, X button to close; compressed to 800px JPEG 0.75 via canvas

## Known Gotchas
- iOS `navigator.share()` must be called in direct user-gesture handler (not after async)
  → solved with bottom sheet modal where each button is its own tap
- Supabase PATCH returns 204 No Content — `r.json()` would throw; use `r.text()` then parse
- `loadWt()` must restore input `.value` fields or next `saveWt()` will wipe address/city/date
- `clearWalkthrough()` must set `supabaseId = null` so re-send creates a fresh row/link
- Tab bar is `position:fixed` with a `.tab-spacer` div below it to prevent content overlap
- Walkthrough bottom bar (`#wt-bottom-bar`) must be `display:none` on load, shown only on walkthrough tab
