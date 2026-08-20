# Repair Sheet — Project Notes

> **MAINTENANCE RULE (read this first):** Whenever `repair_sheet.html` changes —
> new feature, data-shape change, new localStorage key, new gotcha, UI redesign —
> update this file **in the same commit**. A PostToolUse hook in
> `.claude/settings.json` will remind you after every edit to `repair_sheet.html`.
> Keep this file accurate; it is the only documentation this project has.

## Repos
- **repair-sheet**: `tml828/repair-sheet` — GitHub Pages, single-file app
- **taxvault**: `tml828/taxvault` — separate repo, do NOT touch from this session

## Deploy
- Push directly to `main`: `git push origin main`
- GitHub Pages serves `repair_sheet.html` at the repo root
  (live: `https://tml828.github.io/repair-sheet/repair_sheet.html`)
- No CI/CD workflow — deploy is just the push
- Before committing, syntax-check the inline script:
  `node -e "new Function(require('fs').readFileSync('repair_sheet.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1])"`

## Architecture
- Single file: `repair_sheet.html` (~2.1k lines) — never rename it
- No npm, no build step, no CDN scripts (Supabase SDK removed)
- Supabase via raw `fetch()` REST calls only (see `sbFetch()`)
- Anthropic API called directly from the browser (see AI section)
- **Theme: black & gold.** Background `#0D0D0F`, cards `#17171B`/`#1D1D22`,
  borders `#2B2B33`, text `#F5F2E9`, muted `#9C9A90`, gold accent `#D4AF37` /
  bright `#E9C964` / dark `#B8902A`. Active buttons use
  `linear-gradient(135deg,#E9C964,#C9A032)` with **black** text (`#141414`).
  JS-built modals/bottom sheets use inline cssText with the same palette —
  when adding UI in JS, match these hex values. The printable/export sheet
  (`buildPDFHtml`) intentionally stays white with `#B8902A` accents.

## localStorage keys
| Key | Holds |
|---|---|
| `repairSheet_v3` | Repair Sheet state (`state`) |
| `apexWalkthrough` | Walkthrough (`wtData`) |
| `apexCloudSync` | Sync meta (`cloudMeta`: id + per-section `_at` timestamps) |
| `apexLastTab` | Last active tab |

Note: `apexOrders` and `apexAnthropicKey` were removed when the Orders tab was replaced with Settings.

## Supabase
- Project URL: `https://ehlipciasutoderxgptx.supabase.co`; anon key in `SB_KEY`
- Table `walkthroughs`: `id uuid pk`, `address text`, `city text`, `date text`, `items jsonb`, `created_at timestamptz`
- Table `app_state` (whole-app cross-device sync): `id uuid pk default gen_random_uuid()`, `sheet jsonb`, `walkthrough jsonb`, `orders jsonb`, `sheet_at bigint`, `walkthrough_at bigint`, `orders_at bigint`
- RLS: anyone can read/insert/update on both tables (required for contractor link + sync codes)

## Cloud Sync
- "Sync" button in tab bar → `openSyncModal()`; sync code = `app_state` row UUID
- Button is **green** (`.sync-on`) when connected, **red** (`.sync-off`) when not
- `cloudMark(section)` from `save()`/`saveWt()` → debounced 2s push, pull-before-push
- Per-section `_at` timestamps: newest wins per tab, never whole-row clobber
- Pull on load + `visibilitychange` + every 30 seconds (auto-poll interval)
- `_pulling` guard prevents concurrent `cloudPullMerge` calls (race condition)
- `applyCloudSheet/Walkthrough` write localStorage then call `load()/loadWt()`
  so pulled data gets the SAME normalization as local data — never apply cloud
  data without re-running the loader
- Orders section removed from cloud sync (Orders tab replaced by Settings)
- `cloudMeta` only tracks `sheet_at` and `walkthrough_at` (no `orders_at`)

## Tabs & Key Features
1. **Repair Sheet** — sectioned checklist (GROUPS/ALL_SECTIONS), party toggle
   (Contractor/Owner/TBD), notes; share via email/SMS bottom sheet (email body
   auto-truncated if >1800 chars with a note); printable HTML export
   (`downloadPDF`) and self-contained file export (`downloadFile` — warns user
   if cloud sync is active that the exported file won't stay synced).
2. **Walkthrough** — independent items with photos (800px JPEG 0.75) + notes
   textarea only (no separate description field). "Send to Contractor" PATCHes
   the existing `walkthroughs` row (same UUID = same link) or POSTs a new one
   (toasts user if previous link expired). Contractor opens `?wt=UUID`, checks
   items off → PATCH; owner merges `doneAt` (newest wins) + notes via
   `syncFromSupabase()`. Progress label shows "No items yet" when empty.
3. **Settings** — cloud sync controls (start/join/stop, sync code display) and
   app info. Cloud sync covers Repair Sheet + Walkthrough only (Orders removed).
   `renderSettings()` is called when switching to the Settings tab.


## Known Gotchas
- iOS `navigator.share()` must be called in a direct user-gesture handler →
  bottom-sheet modals where each button is its own tap
- Supabase PATCH returns 204 No Content — `sbFetch` parses via `r.text()`
- `loadWt()` must restore header input `.value`s or the next `saveWt()` wipes them
- `clearWalkthrough()` must set `supabaseId = null` so re-send creates a fresh link
- **Never call a full re-render from a text input's `oninput`** — it destroys
  the DOM and kills focus mid-keystroke. Save on input; re-render on blur or
  explicit actions only.
- Tab bar is `position:fixed` with `.tab-spacer` below it; contractor view
  (`loadContractorView`) must hide tab bar, spacer, all three tabs, and
  `#wtBottomBar`
- `#wtBottomBar` is `display:none` on load, shown only on the walkthrough tab
- `switchTab` assumes the first 3 `.tab-btn`s map to Sheet/Walkthrough/Settings
  (4th is Sync) — keep button order in the tab bar HTML
- localStorage quota: photos are the bulk; `storageWarn()` alerts once when
  `setItem` throws
- IDs are generated with `genId()` = `Date.now().toString(36) + random` — enough
  entropy to avoid collisions on rapid taps
- `.wt-check-wrap.has-checked` is the JS-applied fallback for `:has(.checked)`
  (CSS `:has()` not supported in Firefox <121); `toggleWtDone` and `buildWtItem`
  both maintain this class
- `clearAll()` calls `cloudMark('sheet')` after clearing so cloud sync won't
  restore the old data within 30 seconds

## Claude Code project config
- `.claude/settings.json` has a PostToolUse hook (Edit|Write on
  `repair_sheet.html`) that injects a reminder to update this CLAUDE.md.
  If you change the app and this file is stale, fix it before committing.
