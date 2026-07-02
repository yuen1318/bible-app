# Sacred Space — Business Rules

> Extracted from the codebase on 2026-07-02. These rules define how the application behaves, not how it's implemented.

---

## 1. Core Identity

### 1.1 App Purpose
Sacred Space is a distraction-free, offline-first Catholic scripture reading and daily liturgical prayer application. Its primary goal is to facilitate contemplative engagement with sacred text by removing digital distractions.

### 1.2 Tagline
**"Disconnect to Reconnect"** — this is not just marketing copy; it is enforced by the application (see Rule 2.1).

### 1.3 Target Audience
English-speaking Catholics following the daily Mass lectionary cycle, or anyone seeking an offline Bible reader with the NABRE translation.

---

## 2. Connectivity Rules

### 2.1 Forced Disconnection
- When the app detects the device is online (`navigator.onLine === true`) after data has loaded, a full-screen lock overlay is displayed.
- The overlay message: *"To help you focus on Scripture without distractions, please turn off your WiFi or mobile data."*
- The "I'm Disconnected — Continue" button is **disabled** while the device remains online.
- The button becomes enabled **only** when `navigator.onLine` transitions to `false`.
- This rule is enforced on every page load and whenever the online/offline state changes.
- A dismissible WiFi banner on the home screen reinforces this: *"For uninterrupted prayer time, consider disconnecting from WiFi."*

### 2.2 Offline-First
- All Bible text and lectionary data must be available without a network connection.
- The app must never make a runtime network request for scripture content during normal reading.
- External dependencies are limited to Google Fonts (Inter, Cormorant Garamond) — the app functions with system fallback fonts if these fail to load.

### 2.3 `file://` Protocol Exception
- When opened from a local file (detected via `window.location.protocol === 'file:'`), the Service Worker and Cache API are disabled.
- In this mode, only IndexedDB is used for offline persistence.
- PWA installation is suppressed (manifest not registered).

---

## 3. Data Rules

### 3.1 Single Translation
- The only supported Bible translation is the **NABRE** (New American Bible, Revised Edition).
- No other translations (KJV, NIV, ESV, Douay-Rheims, etc.) are available.
- Users cannot switch translations.

### 3.2 Canon
- The Bible contains **73 books**: 46 Old Testament (including Deuterocanonicals: Tobit, Judith, 1 & 2 Maccabees, Wisdom, Sirach, Baruch) and 27 New Testament.
- Books are presented in canonical order as they appear in the NABRE data.
- Total: **1,328 chapters**, **35,091 verses**.

### 3.3 Lectionary Year
- The daily readings are hardcoded for the **2026 liturgical year**.
- The calendar defaults to the current system year and month.
- Days without assigned readings show the message: *"No readings available for this date."*
- After December 31, 2026, the calendar will display empty dates unless new lectionary data is loaded.

### 3.4 Reference Resolution
- Lectionary references are resolved against the NABRE text.
- The scraping source used **Douay-Rheims verse numbering**, so book name and chapter/verse mappings are normalized via the `BOOK_ALIASES` lookup table.
- An unrecognized book name causes the passage to be silently skipped — the accordion section shows *"Passage not available."*

### 3.5 Data Storage Tiers
Bible data is stored redundantly across three tiers:
1. **In-memory** — `NABRE_DATA` global variable (loaded synchronously from `nabre.js`)
2. **IndexedDB** — `SacredScriptureDB` database, `bible` object store (primary offline store)
3. **Cache API** — `sacred-scripture-v2` cache (secondary offline store)

Storage is approximately **doubled** (~10MB for ~5MB of data) due to this redundancy. This is intentional: if one tier fails, another serves as fallback.

---

## 4. User Preferences

### 4.1 Persisted Preferences
| Preference | Storage Key | Default | Values |
|---|---|---|---|
| Last-read book | IndexedDB `prefs.bookIndex` | Genesis (index 0) | 0–72 |
| Last-read chapter | IndexedDB `prefs.chapter` | Chapter 1 | 1–N per book |
| Font size | IndexedDB `prefs.fontSizeIdx` | 1 (Standard, 16px) | 0–4 (14px–22px) |
| Theme | `localStorage.scripture-theme` | `dark` | `dark` or `light` |

### 4.2 Save Timing
- Preferences are written to IndexedDB on **every change**: book switch, chapter switch, font adjustment, and returning to the home screen from the Bible reader.
- Reading position is restored on next app load.
- WiFi banner dismissal is **not persisted** — it reappears on every page load.

### 4.3 Theme
- Default theme is **dark**. There is no detection of `prefers-color-scheme` OS setting.
- Theme toggle updates the `<html data-theme>` attribute and the `<meta name="theme-color">` tag for PWA chrome color.

---

## 5. Reading Rules

### 5.1 Bible Reader
- Verses are displayed sequentially with gold superscript verse numbers.
- Chapter heading is displayed in gold accent: *"Book Name Chapter"*.
- Font size is adjustable across 5 levels: Small (14px), Standard (16px), Medium (18px), Large (20px), X-Large (22px).
- Changing font size persists and only affects the Bible reader — not the daily readings.

### 5.2 Navigation
- Books are selected via a dropdown listing all 73 books in canonical order.
- Chapters are selected via a dropdown of numbered chapter options (e.g., "Ch 1" through "Ch 150" for Psalms).
- Keyboard shortcuts: **Arrow Left** = previous chapter, **Arrow Right** = next chapter, **Escape** = return to home screen.
- Touch gestures: **swipe left** = next chapter, **swipe right** = previous chapter (minimum 80px swipe distance).
- Chapter navigation resets scroll position to top.

### 5.3 Verse Copying
- Tapping any verse copies it to the clipboard in the format:
  ```
  "verse text" -- Book Chapter:Verse
  ```
- A *"Copied!"* toast confirms the action. The verse briefly highlights with a gold background.
- On `file://` protocol or browsers that deny clipboard access, copying fails silently (no error handling).

### 5.4 Empty Chapter
- If a chapter has no verses, the message *"No verses in this chapter."* is displayed.

---

## 6. Daily Readings Rules

### 6.1 Calendar
- A 7-column grid (Sunday–Saturday) displays the current month.
- **Today's date** is highlighted with a gold border.
- **Selected date** has a gold background.
- Days **with readings** are shown in bold.
- Days **without readings** are shown in normal weight.
- Calendar navigation supports previous/next month with `<` / `>` buttons.

### 6.2 Reading Display
Daily readings are shown in **accordion sections**, one per reading type:
1. First Reading
2. Responsorial Psalm
3. Second Reading (if present)
4. Alleluia (Gospel Acclamation)
5. Gospel

Each accordion shows:
- Reading type label
- Scripture reference in gold (resolved to full book name + chapter:verse)
- Full scripture text when expanded

### 6.3 BEST Reflection
After the Gospel, a journaling framework is displayed with four questions:
- **B — Blessing**: *What blessing, promise, or encouragement does God give me?*
- **E — Example**: *Is there an example I should follow?*
- **S — Sin**: *Is there a sin to confess or turn away from?*
- **T — Truth**: *What truth about God does this teach me to believe?*

This is displayed as a styled block with each question in a separate row. It is informational only — no user responses are collected or saved.

---

## 7. PWA Rules

### 7.1 Installation
- The `beforeinstallprompt` event is captured and deferred.
- After data loads successfully, a toast appears: *"Tap here to install this app for offline use."*
- On install success: *"App installed! You can now use it fully offline."*
- The manifest is generated dynamically at runtime as a Blob URL, not as a static `manifest.json` file.

### 7.2 Manifest
- `display: standalone` — the app opens without browser chrome.
- Icons are inline SVG data URIs (192×192 and 512×512) — a gold cross on a dark background.
- Theme color matches the current theme (`#09090b` for dark, `#f8f8f7` for light).
- `viewport-fit=cover` for notched devices.

### 7.3 Service Worker
- **Version:** `v1.3.0`
- **Install:** Pre-caches app shell files (`index.html`, `nabre.js`, `2026.js`, root `/`).
- **Activate:** Deletes all caches not matching the current version, then claims all clients.
- **Fetch:** Network-first for same-origin and Bible data URLs; falls back to cache on network failure.
- **Update:** Auto-checks for updates on every page load. If a new worker is waiting, it activates it and reloads the page.

---

## 8. UI/UX Rules

### 8.1 Screen Transitions
- All screen transitions use a 350ms ease animation (`cubic-bezier(0.4, 0, 0.2, 1)`).
- Incoming screens animate in with `opacity: 0 → 1` and `translateY: 8px → 0`.
- Only one screen is visible at a time (absolute positioning within the app container).

### 8.2 Toast Notifications
- A single shared toast element is used for all brief messages.
- Toast auto-dismisses after a timeout (varies by message type).
- Messages include: copy confirmation, no readings available, install prompt, install success.

### 8.3 Loading Screen
The download/loading screen ("Preparing Your Sanctuary") is displayed while Bible data initializes:
- Animated cross icon with breathing/pulse concentric rings.
- Rotating scripture verse previews that change every **5 seconds**. Current rotation: Psalm 23:1, John 3:16, Isaiah 41:10, Matthew 11:28, Romans 8:38-39, Joshua 1:9.
- Progress bar (gold gradient glow).
- Status text that updates through initialization stages.
- Since `nabre.js` loads synchronously, this screen typically flashes briefly or is skipped entirely.

### 8.4 No Zoom
- `user-scalable=no` is set in the viewport meta. Pinch-to-zoom is disabled.
- `overscroll-behavior: none` prevents pull-to-refresh.

### 8.5 Text Selection
- Text selection is disabled globally (`user-select: none` on `*`).
- Selection is restored only on `.allow-select` regions (Bible content area, daily readings content).

---

## 9. Error Handling Rules

### 9.1 Data Unavailable
- If `NABRE_DATA` is undefined (nabre.js failed to load), the app falls back to IndexedDB, then Cache API.
- If all three tiers are empty: *"Could not load scripture data. Please check your connection or try refreshing."*
- The progress bar turns red on failure.

### 9.2 Lectionary Data Missing
- If `LECTIONARY_2026` is undefined, no explicit error is shown — the daily readings screen will simply have no data.

### 9.3 Silent Failures (by design)
- Resolution failures in `lookupPassage` (book not found, chapter not found, verse parse error) are silent — the affected reading part is skipped with a *"Passage not available"* placeholder.
- Clipboard write failures are silent — no error toast, no fallback.

---

## 10. Privacy & Analytics

### 10.1 No Tracking
- The app collects **zero analytics, telemetry, or usage data.**
- No cookies are set.
- No third-party requests are made during normal operation (only Google Fonts on first load).
- This is a deliberate design choice aligned with the "sacred space" philosophy.

---

## 11. Constraints & Known Limitations

| Constraint | Detail |
|---|---|
| **Single translation** | Only NABRE; no mechanism for adding translations |
| **Hardcoded year** | Lectionary data for 2026 only; requires manual update for subsequent years |
| **No search** | No full-text search across Bible or lectionary |
| **No reading plans** | No structured plans, streaks, or progress tracking |
| **No annotations** | No bookmarking, highlighting, or note-taking |
| **No social features** | No sharing, community, or prayer groups |
| **No audio** | No text-to-speech or audio Bible |
| **Monolithic file** | All logic in single `index.html`; no modular code structure |
| **Blocking load** | 7MB `nabre.js` loads synchronously, blocking the main thread during initial parse |
| **No CSP** | No Content Security Policy; `innerHTML` used extensively |
