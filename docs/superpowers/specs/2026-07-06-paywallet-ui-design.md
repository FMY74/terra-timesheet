# Terra Timesheet v2 — "Wallet" UI redesign

2026-07-06. Owner has used a personal timesheet PWA ("PayWallet"-style UI) daily for ~2 months and
prefers that UI over the current trading-terminal design. This spec rebuilds the app with that UI,
keeps/adds the requested features, and renames it **Terra Timesheet**.

## Goals & success criteria

1. UI visually matches the owner's reference screenshots (wallet-style dark UI; see Design system).
2. Adding a shift gives clear visual feedback (owner's #1 complaint: "did it save or not?").
3. A wrong shift can be deleted by swiping the history row left (works on iOS Safari home-screen
   web apps) — with Undo. Tap row → Edit sheet remains the non-gesture path.
4. Pay engine reproduces the owner's real exported history **to the cent** (verified locally
   against the real CSV; real numbers never enter this repo).
5. Pay rates / tax / goals are user-editable in Settings (defaults are neutral sample values) —
   no personal pay data hardcoded.

## Invariants

- **Privacy**: no real rates, real shifts, or real pay figures in code, spec, tests-in-repo, or README.
  Defaults are the neutral sample family (base $45). Real-data verification happens outside the repo.
- **Money correctness**: engine rules below are exact; any change must re-pass the local real-CSV test.
- **Fail-closed persistence**: corrupt localStorage or a malformed import must never brick the app
  or destroy existing data (validate into a candidate object before commit — same policy as v1).
- Single self-contained HTML file, vanilla JS, no build step, offline after first font load.

## Architecture

- One file: `app.html` (v1 `terra-timesheet.html` stays untouched for now; integration decided at ship time).
- `localStorage` key `terra-timesheet-v2`. Loader also accepts the v1 (`terra-timesheet-v1`)
  week-map format and converts (one shift per day → shifts array); ignore silently if absent/invalid.
- Dark theme only (reference UI is dark). `<meta name="theme-color">`, `apple-mobile-web-app-*` metas.
- Fonts: Inter (UI) + JetBrains Mono (all numerals/times/amounts), Google Fonts with system fallbacks.

## Data model

```js
state = {
  version: 2,
  shifts: [ { id, date:'YYYY-MM-DD', start:'HH:MM', end:'HH:MM', phMode:'auto'|'yes'|'no' } ],
  settings: {
    taxRate: 18.6,                     // percent
    rates: { base:45, after18:54, sat:54, sun9:81, sun:63, ph:90 },  // $/h sample defaults
    weeklyGoal: 0, monthlyGoal: 0,     // 0 = off
  },
  ui: { tab:'day', selectedDate:'YYYY-MM-DD' },
}
```

- Multiple shifts per day are allowed (Duplicate exists). IDs are `crypto.randomUUID()`.
- Validation: `isDateStr` (regex + round-trip), `isTimeStr` (00:00–23:59), start<end else the shift
  is stored but flagged "Check times" and computes $0/0h (overnight not supported, same as v1).

## Pay engine (exact — verified against real export, 37/37 rows to the cent)

- Paid hours = (end − start) − 30 min unpaid break, floor 0.
- Rate buckets: weekday `base` (to 18:00) / `after18`; `sat` flat; Sunday `sun9` (before 09:00) /
  `sun`; `ph` flat (overrides everything when the date is a public holiday).
- Public holidays: VIC 2025–2026 list (incl. 2026-09-25 AFL GF Eve) with per-shift
  `phMode` override (auto/yes/no).
- **Break allocation: the 30-min break sits at the centre of the shift** — remove
  [mid−15min, mid+15min] from the paid window, then split the remaining minutes into buckets.
  (An equally valid "deduct from cheapest bucket" rule is indistinguishable on all real data;
  centre-of-shift is the documented choice.)
- Per shift: `gross = round2(Σ bucketHours × rate)`, `tax = round2(gross × taxRate)`,
  `net = round2(gross − tax)`.
- Per period (day/week/month/year): `gross = Σ per-shift rounded gross`,
  `tax = round2(periodGross × taxRate)`, `net = round2(periodGross × (1 − taxRate))`
  (independent rounding — matches the reference app's observed behaviour).
- Weeks start Monday. All dates local-time (no UTC parsing).

## Screens (single page, 4 tabs)

Layout order (one scrolling column, max-width ~560px centred on desktop):

1. **Header**: rounded-square gradient icon ($ glyph) + "Terra Timesheet" + gear button (Settings sheet).
2. **Tabs**: segmented pill — Day / Week / Month / Year (active = raised pill).
3. **Period nav**: card with ‹ › chevrons; centre title + subtitle
   (Day: "Today"/weekday + `Jul 6, 2026`; Week: `Jul 6 – Jul 12` + "This Week"; Month: `July 2026` +
   "This Month"; Year: `2026` + "This Year"). Tapping the centre jumps back to today.
4. **Earnings card**: green dot + "{Period}'s Earnings"; huge mono gross `$X,XXX.XX` + small `AUD`;
   sub "Gross Pay · <ISO range>"; divider; 3 columns NET PAY (green) / TAX (red, `-$…`) / HOURS.
   If weeklyGoal (Week tab) or monthlyGoal (Month tab) > 0: slim progress bar + "x% of $goal".
5. **Month tab only — DAILY EARNINGS (MONTH)** card: per-day bar chart (gradient bars for worked
   days, dim stubs otherwise), "Peak: $X" top-right, x labels 1/5/10/15/20/25/31.
6. **ADD SHIFT** section: date row (calendar chip + WEEKDAY + `Jul 6, 2026`, native date input),
   CLOCK IN / CLOCK OUT tiles (native time inputs, big mono), "☕ Unpaid break −30 min (auto)" row,
   Public holiday row (Auto/Yes/No segmented, default Auto), and the full-width gradient
   **+ Add Shift** button.
   **Feedback on add**: button morphs to "✓ Added" ~1.2s, the new history row slide/flash-highlights,
   and a snackbar "Mon, Jul 6 shift added" appears. Editing an existing date is NOT implicit —
   Add always appends; edits go through the Edit sheet.
7. **IF I WORK NOW… (PAY SIMULATOR)** card: duration chips 1h/2h/4h/6h/8h; left "From HH:MM · Nh"
   + "$rate/h" (blended gross/duration); right big gross + "Net $…". Uses current time + settings
   rates; no break deduction; recomputes every minute while visible.
8. **PERIOD BREAKDOWN** card: one row per rate bucket used in the selected period
   (label e.g. "Weekday base" / "Weekday after 18:00" / "Saturday" / "Sunday before 9:00" /
   "Sunday" / "Public holiday"): hours + gross. Empty state: "No data for this period."
9. **SHIFT HISTORY** section (header + mint "Clear All" on the right): rows for the selected
   period, newest first. Row = weekday+date | times + paid hours (+ "PH" or amber "Check times"
   badge) | gross (mono, right) with small "Net $…" under it.
   - Tap row → **Edit Shift sheet**.
   - **Swipe left → Delete** (red reveal, threshold 40%, rubber-band, axis-locked). Must work with
     Pointer Events AND an iOS touch-event fallback; `touch-action: pan-y` on rows. After delete:
     snackbar "…deleted" with **Undo** (6s, restores the exact shift by id).
   - "Clear All" → confirm dialog → clears only the visible period's shifts → snackbar with Undo.
   - Empty state: "No shifts in this period."
10. **Edit Shift sheet** (bottom sheet, drag handle, dim scrim): title "Edit Shift" + "Update date,
    time or break."; same date/time/PH fields as Add form; gradient **✓ Save Changes**; row of
    secondary buttons **Duplicate** / **Cancel** / **Delete** (red outline). Save/duplicate/delete all
    close the sheet, flash the affected row, and show a snackbar (delete's has Undo).
11. **Settings sheet**: "Adjust your goals and preferences." rows — Weekly Goal ($, 0=off),
    Monthly Goal, Tax Rate (%, default 18.6), then **Pay rates ($/h)**: base / after-18:00 /
    Saturday / Sunday before 9 / Sunday / Public holiday (defaults above); gradient **Save
    Settings**; row **Export CSV** / **Close**; **Import CSV** + **Backup JSON** / **Restore JSON**
    (fail-closed) as quiet secondary actions; red-outline **Delete All Data** (typed/2-step confirm).

## CSV formats

- **Export** (same columns as the reference app):
  `"Date","Day","Clock In","Clock Out","Break (min)","Hours","Gross","Tax","Net"` — one row per
  shift, chronological, quoted, filename `terra-timesheet-shifts-YYYY-MM-DD.csv`.
- **Import**: accepts exactly that header (quoted or not). Uses Date/Clock In/Clock Out only
  (engine recomputes pay); Break must be 30 or blank; malformed file → reject whole file with a
  clear message, existing data untouched. Duplicate rows (same date+start+end as an existing
  shift) are skipped; summary snackbar "N imported, M skipped".

## Design system (from reference screenshots)

- Background near-black `#05070a` with a faint teal-green radial glow top-left; cards `#0d1117`-ish
  translucent with 1px hairline `rgba(255,255,255,.06)`, radius ~24px.
- Text: white; muted `#8b93a1`; mono numerals everywhere for money/time (dotted-zero look).
- Accents: mint `#34d399` (positive/net/dots), red `#f87171` (tax/delete), amber (warnings);
  signature **gradient mint→blue (#34d399 → #3b82f6)** for primary buttons (Add Shift, Save) and
  the app icon; simulator chip selected = blue outline + blue tint.
- Section labels: all-caps, letterspaced, muted (ADD SHIFT / PERIOD BREAKDOWN / SHIFT HISTORY).
- Bottom sheets: rounded-top 28px, drag handle, buttons ≥44px tall. Reduced-motion respected.

## Accessibility

Semantic roles (tablist/tab/tabpanel, dialog for sheets with focus trap + Esc/scrim close,
`aria-live` snackbar), keyboard path for everything gesture-based (row → Edit sheet → Delete),
visible focus rings, WCAG AA contrast on all text.

## Testing

- Engine unit checks runnable headlessly (expose `window.__tt = {computeShift, splitBuckets, …}`):
  bucket splits across 18:00/09:00 boundaries, centre-break allocation, PH override, invalid times,
  rounding, period aggregation, CSV round-trip, import rejection cases.
- Local-only (never committed): replay the owner's real CSV through the engine — all rows must
  match Hours/Gross/Tax/Net to the cent.
- Playwright visual pass at 390×844 + desktop; WebKit swipe-delete check; add/edit/delete/undo flows.

## Out of scope

Light theme, overnight shifts, 2027+ VIC holidays (yearly maintenance note stays in vault),
multi-user/sync, service worker.
