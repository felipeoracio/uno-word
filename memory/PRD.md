# Word Count — Writing Session Chrome Extension

## Original problem statement

Build a modern, minimalistic Chrome extension that lets a user click **Start
Session**, type anywhere in Chrome, click **Stop Session**, and see exactly
how many words they wrote. Phase 1 is strictly the session word counter —
no accounts, no cloud, no analytics dashboard, no writing feedback. Code
must be modular so future phases plug in cleanly.

## User personas

- **Writer** (primary) — journalists, authors, bloggers, students who want
  a distraction-free way to know how much they wrote in a sitting.
- **Future paid subscriber** — same writer, later, who wants historical
  stats (daily / weekly / monthly / lifetime).

## Core requirements (locked)

1. Chrome Manifest V3.
2. Single popup with idle / active / done states.
3. Session survives popup close and page navigation.
4. Word count is based on **typing activity**, not the current contents of
   a text field.
5. On first run, ask the user whether pasted text should be counted as
   typed or tracked separately; store the preference.
6. When "separate" mode: big number = words typed, smaller line = pasted
   count.
7. Auto light/dark theme (follows OS preference).
8. Zero network calls. Writing content never leaves the device.
9. Only `storage` permission requested.
10. Graceful failure on Chrome-restricted pages.

## Architecture

- `manifest.json` — MV3 config.
- `background.js` — session state, `chrome.storage.local` persistence, pure
  word-counting engine (`countWords`, `applyDelta`), message router.
- `content.js` — injected on every allowed page, classifies `InputEvent`s
  via `inputType` and reports `{ kind, added|removed }` deltas.
- `popup/` — HTML + CSS + JS controller; renders idle / active / done /
  onboarding / settings panels.
- `icons/` — brand mark (rounded blue square, white W).
- `tests/engine.test.js` — 13 unit tests for pure functions.

## What's been implemented — 2026-01

- MV3 extension scaffold with manifest, background service worker, content
  script, and popup UI.
- Session lifecycle: start → track → stop → show result → start new.
- Typed / pasted / deleted delta pipeline with best-effort deletion accounting.
- First-run onboarding (Keep separate / Count pasted as typed) and Settings
  panel to change it later.
- Premium minimal UI with auto light/dark, tabular numerals, pulse dot for
  active state, duration timer.
- Data-testid attributes on every interactive element.
- 13 unit tests passing (`node tests/engine.test.js`).
- README with install steps, architecture diagram, privacy notes, known
  limitations, and manual test checklist.
- **On-screen floating counter** (previous iteration): Shadow-DOM overlay
  injected by `content.js`, showing `● N words` in the bottom-right of
  every supported page while a session is active. Draggable with position
  persisted in `wc_settings.counterPos`. Click-to-expand reveals a
  Stop Session button. Fully synchronized with the popup via
  `chrome.storage.onChanged`. Auto light/dark, keyboard accessible,
  cross-tab consistent.
- **Session history** (previous iteration): On stop, the completed session is
  appended to `wc_history` in `chrome.storage.local` (cap 500, newest
  first). A History panel in the popup (clock icon in header) shows
  sessions grouped by day (Today / Yesterday / weekday) with start time,
  duration, and word count, plus a "words today" summary. Empty state and
  a "Clear history" action included. Zero-word sub-second ghost sessions
  are dropped.
- **Paid features Phase 1** (this iteration):
  - Free vs Pro subscription state (demo toggle in Account settings).
  - Local **profile** (timezone, createdAt) — no cloud.
  - 4-step pro onboarding: Language → Goal → Themes → Done.
  - **Session word goal** with strict validation (positive integer,
    multiple of 25). Inline error suggests neighbours.
  - Floating counter shows `X / goal words` for Pro, dot turns blue on
    goal reached, session never stops automatically.
  - **Daily prompt** system: static library of ~29 themes × 4 prompts ×
    2 languages, deterministic per day, "Another prompt" cycles through
    respecting history. Prompt regenerates automatically when language
    changes.
  - **English / Spanish** localization across popup, floating counter,
    and dashboard. Auto-detected from `navigator.language` on first run;
    stored preference wins after that.
  - **Full-page Dashboard** opened in a new tab: Today / This week /
    This month / This year / All time views with hero total, bar chart,
    avg per day / best day / longest session stats, and recent-sessions
    list. Free users see a paywall banner above; Pro users see a PRO
    badge and full stats.
  - **Upgrade screen** with $4.99 / month placeholder pricing and demo
    note. Free-user popup shows a tasteful Upgrade CTA under the session.
  - New tests: goal validation, prompt picker (25 total passing).

## Known limitations (documented in README)

- Deletion source (typed vs pasted) is best-effort.
- Chrome-restricted pages (`chrome://…`, Web Store) can't be observed.
- Cross-origin iframes are excluded (`all_frames: false`).
- Rich editors with non-standard input handling (Monaco, CodeMirror) may
  behave inconsistently.
- IME intermediates ignored; committed composition is counted.

## Prioritized backlog (future phases — DO NOT build until requested)

**P1 — Free tier polish**

- Persist last N completed sessions locally so users can see recent counts
  even after starting a new session.
- Keyboard shortcut to start/stop sessions.
- Options page with data-clear button.

**P2 — Paid tier foundations**

- Authentication (playbook driven — call `integration_playbook_expert_v2`
  before writing any auth code).
- Cloud sync adapter (statistics only, never raw text unless explicitly
  opted in).
- Dashboard page: daily / weekly / monthly / yearly / all-time totals,
  longest session, streaks, WPM.

**P3 — Writing coach**

- Writing evaluator that surfaces clarity / repetition / structure
  observations without rewriting the user's prose. Explicit opt-in per
  session.

## Next tasks (recommended)

1. Manual QA using the checklist in `extension/README.md`.
2. Ship Phase 1 to the Chrome Web Store (optional).
3. Ask the user which P1 item to prioritize next.

---

## Word-counting accuracy fix (2026-06)

Fixed the tokenizer and the Cut/Undo synchronization bug.

- New centralized engine: `extension/shared/wordcount.js` — `countWords`,
  `classifyDelta`, `applyDelta`. Loaded by background.js (importScripts),
  content.js (manifest content_scripts) and tests (require). One source of truth.
- Tokenizer rule: a whitespace-split token counts as a word only if it contains
  at least one Unicode letter/number (`/[\p{L}\p{N}]/u`). Standalone punctuation
  (`!!!`, `...`, `-`) → 0. One-char words (`I`, `a`) and contractions/hyphenated
  words count. No special-casing.
- Word count is now reconciled against the *actual* editor text on every `input`
  event via word-level diff + `inputType` classification. Cut (`deleteByCut`),
  Undo (`historyUndo`), Redo (`historyRedo`) and Delete all resync the count.
- Paste protection preserved: paste/drop additions go to `pastedWords`, never
  `typedWords`. Not full-document counting.
- Session state stores authoritative `typedWords`/`pastedWords` integers
  (previously derived from accumulated text buffers that couldn't represent
  mid-text cut/undo).
- Tests: `extension/tests/engine.test.js` extended from 25 → 44 passing.
  Browser-verified via `extension/tests/preview.html` (type→4, cut→3, undo→4;
  punctuation-only→0; paste protection with "+N pasted").

---

## Phase 2 — Streak · Free/Paid history · CSV · Feedback · first-launch language (2026-06)

Extended the shipped extension (no rebuild). All logic client-side; feedback via `mailto:`.

- New engine `extension/shared/history.js` (pure, tested): `sessionWords` (typed
  only — pasted never counts), `computeStreak` (consecutive days in user tz),
  `visibleHistory` (Free = last 24h, Pro = all — access rule, no deletion),
  `toCSV`/`parseCSV`, `mergeHistory` (id/composite dedupe).
- Daily streak: popup session view + Pro dashboard. Message `GET_STREAK`.
- History gating: `GET_HISTORY` returns visible set + `hasMore`; Free sees a 24h
  notice + Upgrade CTA; older records preserved so upgrade restores them instantly.
- CSV export (`EXPORT_HISTORY`) + import (`IMPORT_HISTORY` merge/replace) with a
  validation → summary → merge/replace dialog and a destructive-replace confirm.
  Clear History has its own confirm. Pro-gated; Free gets the Upgrade CTA.
- Removed the "copied/pasted text" setting entirely (UI + storage purge in
  `getSettings`); floating counter and popup now show TYPED words only.
- First-launch language screen (before any UI) via `langChosen`; Settings →
  Language still changes it; manual choice wins over auto-detect.
- Send Feedback (Settings → Support) composes `mailto:unowordapp@gmail.com`
  with type/version/lang/plan/browser — never any writing content.
- Full EN/ES localization for every new string.
- Rich-editor coverage: `tests/preview-rich.html` (Notion-style contenteditable)
  + contenteditable unit tests. Google Docs canvas limitation documented in README.
- Tests: `engine.test.js` 50 + `history.test.js` 18 = **68 passing**. Browser-
  verified content script on textarea + contenteditable (cut/undo/paste), popup +
  settings render checks.

---

## Phase 3 — Progress moved inside the popup (2026-06)

- `Progress` now opens as an in-popup view (`#progress-panel`) instead of a new
  dashboard tab. The header chart icon + a new "Progress →" link in the session
  view both call `openProgress()`; `OPEN_DASHBOARD` / `chrome.tabs.create` is no
  longer used from the popup (no external navigation).
- Popup widens to 720px only while Progress is open (`body.wc--wide`), then
  returns to 360px on Back. No horizontal scrolling.
- Reuses existing `GET_STATS` / `GET_STREAK`; chart logic ported compactly into
  `popup.js` (`renderProgChart`) mirroring the dashboard — no new dependency, no
  duplicate calculations. Range tabs Today/Week/Month/Year/All, two summary cards,
  streak, and a History button (opens the existing in-popup history).
- Free users see the Today card + streak + a tasteful Upgrade card (tabs/chart
  hidden); Pro sees the full view. Session state + floating counter untouched
  (Progress only reads state). EN/ES strings added (`progress.*`).
