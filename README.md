# Speak

A bedside communication tool for people who can't talk well. The idea is simple: type a message, it auto-scales to fill the screen, show it to whoever is in the room. Fast, distraction-free, works on any phone in any browser.

This is a single-file static app (`index.html`) with no build step. Deploying to Vercel just means pointing it at this repo — Vercel will serve the `index.html` at the root.

---

## Context

The user is bedridden and using this to communicate verbally with people around them. Design decisions should always prioritise:

- **Large, legible text** above everything else
- **Minimal friction** — every extra tap is a cost
- **Reliability** — this is a communication aid, not a toy. It must work offline, not lose data, never crash
- **One-handed use** — the user may be holding the phone with one hand, lying down
- **Emotional tone** — the UI should feel calm and dignified, not medical-sterile or cutesy

The current implementation is a clean dark-mode single-page HTML/CSS/JS app with no dependencies beyond a Google Fonts import. Keep it that way unless there's a strong reason to add something. No frameworks, no bundlers, no build step.

---

## Current Implementation

### What it does

- `contenteditable` div that fills the screen
- On every keystroke, a binary-search algorithm finds the largest font size where the text still fits within the visible canvas (above the toolbar), accounting for the iOS virtual keyboard via `window.visualViewport`
- **Wipe button**: saves current text to `localStorage` history, then clears the editor
- **History panel**: slides up from the bottom, lists all past messages newest-first, tap any entry to restore it to the editor
- Swipe-down gesture to dismiss the history panel
- `apple-mobile-web-app-capable` meta tags so it works well when saved to the iOS home screen

### Key JS functions

- `fitText()` — binary search font size, called via `scheduleFit()` (rAF-debounced)
- `pushToHistory(text)` — prepends to the `speak_v1` localStorage array
- `getHistory()` — reads from localStorage with a try/catch
- `openPanel()` / `closePanel()` — slide the history sheet in/out
- `renderPanel()` — rebuilds the history list from localStorage

---

## File structure

```
index.html        # Everything. Inline CSS, inline JS, no external deps except Google Fonts
manifest.json     # PWA manifest — enables landscape lock and home-screen install
README.md         # This file
```

---

## Landscape orientation

This app is designed to be used in landscape mode. Landscape gives text more horizontal room to grow large, and the toolbar fits naturally as a short bottom strip. The user is lying down, likely holding the phone sideways.

### Why locking orientation is hard on the web

There are three approaches, each with different constraints:

**1. `screen.orientation.lock('landscape')` (JS API)**
The direct API. Works on Android Chrome. On iOS Safari it throws immediately unless the page is running as a fullscreen PWA (i.e. the user added it to their home screen and opened it from the icon). Must be called inside a user gesture. Fails gracefully.

**2. `manifest.json` with `"orientation": "landscape"` (PWA manifest)**
When the user installs the app to their home screen and opens it as a PWA, the OS respects the manifest orientation and locks it. This is the most reliable cross-platform path, but it requires the user to go through the "Add to Home Screen" flow. **The `manifest.json` in this repo is already configured for this** — see below.

**3. CSS transform rotation**
Visually rotate the entire app 90° when `screen.orientation.type` reports portrait. This is a last-resort fallback for cases where neither of the above work. However, on iOS, rotating the app element while a `contenteditable` is focused causes the software keyboard to appear in the wrong orientation relative to the rotated view — it's disorienting and effectively unusable. **Do not use CSS rotation as the primary strategy for this app.** It is fine for display-only contexts (kiosks, dashboards) but breaks keyboard input.

### Recommended implementation

**Step 1 — manifest.json (already in repo)**

The `manifest.json` at the root includes `"orientation": "landscape"`. The `index.html` already links it. When the user installs the app via "Add to Home Screen" on iOS or Android, the OS will lock it to landscape. This is the zero-friction path — no code changes needed.

**Step 2 — JS lock on first gesture**

Call `screen.orientation.lock()` on the first user interaction. It will succeed on Android Chrome (including regular browser tabs, not just PWA), and on iOS if the app is already running as a fullscreen PWA. Wrap in try/catch; failure is silent.

```javascript
let orientationLocked = false;
async function tryLockLandscape() {
  if (orientationLocked || !screen.orientation?.lock) return;
  try {
    await screen.orientation.lock('landscape');
    orientationLocked = true;
  } catch {
    // Fails silently on iOS regular browser — that's expected
  }
}
document.addEventListener('pointerdown', tryLockLandscape, { once: true });
```

**Step 3 — Portrait nudge (fallback UI)**

If the device is in portrait and the lock failed, show a gentle non-blocking nudge. Check on `orientationchange` and `resize`. Don't block the UI — some users may actually prefer portrait.

```javascript
function checkOrientation() {
  const isPortrait = window.matchMedia('(orientation: portrait)').matches;
  document.getElementById('rotate-nudge').hidden = !isPortrait || orientationLocked;
}
window.addEventListener('orientationchange', checkOrientation);
window.visualViewport?.addEventListener('resize', checkOrientation);
```

The nudge element should be a small, dismissible banner (not a blocking overlay) at the top of the screen with a rotate icon and text like "Rotate for best view". The user can tap to dismiss permanently (persist in localStorage). It must not obscure the text canvas or toolbar.

### Testing matrix

| Context | Android Chrome | iOS Safari (browser tab) | iOS Safari (PWA home screen) |
|---|---|---|---|
| manifest orientation | ✗ not used | ✗ not used | ✓ locks landscape |
| JS lock API | ✓ works | ✗ throws | ✓ works |
| CSS rotation | ✓ visual only | ✓ visual only | ✓ visual only |

For the typical use case (installed to home screen), manifest + JS covers everything. The nudge covers the remaining edge case.

---

## Planned Improvements

These are the features to implement. They're listed in rough priority order.

---

### 1. Debounce-based auto-save to history

**Current behaviour:** Messages are only saved when the user explicitly taps Wipe.

**Desired behaviour:** After the user stops typing for a forgiving debounce window (suggest **4–5 seconds**), the current message is silently saved to history. The editor is _not_ cleared — this is purely a background checkpoint, not a "send". This means if the phone dies or the tab closes mid-conversation, nothing is lost.

**Implementation notes:**
- On each `input` event, reset a debounce timer
- When the timer fires and `editor.textContent.trim()` is non-empty, call `pushToHistory()` — but only if the text has changed since the last auto-save (avoid duplicate entries from the same message)
- Track `lastSavedText` in memory to deduplicate
- Do not show any UI feedback for the auto-save; it should be invisible
- The Wipe button should still trigger an immediate explicit save before clearing, as it does now

---

### 2. Manual light/dark mode toggle

**Current behaviour:** Always dark.

**Desired behaviour:** A toggle button in the toolbar that switches between a dark theme and a light theme. Hospital rooms have unpredictable lighting — bright overhead fluorescents during the day, dim night lights at night — so this must be a manual control, not a `prefers-color-scheme` auto-switch.

**Implementation notes:**
- Persist the preference in `localStorage` under the key `speak_theme` (`'dark'` | `'light'`)
- Apply the theme by toggling a class on `<html>` or `<body>` (e.g. `class="light"`)
- The default/dark theme is the current one. The light theme should be:
  - Background: `#f5f0e8` (warm off-white, easier on eyes than pure white)
  - Text: `#1a1a2e`
  - Surface: `#ebe5d8`
  - Border: `rgba(0,0,0,0.08)`
  - Accent: keep `#ff6340` — it works on both themes
  - Muted: `rgba(26,26,46,0.4)`
- The toggle button lives in the toolbar alongside History and Wipe. Use a sun/moon icon (SVG inline). Three buttons in a row is fine at these sizes.
- On initial load, read `speak_theme` from localStorage and apply before first paint (inline `<script>` in `<head>` before any CSS, to avoid flash)

---

### 3. Confirm-before-wipe

**Current behaviour:** Tapping Wipe immediately saves to history and clears the editor.

**Desired behaviour:** A two-stage confirmation so accidental taps don't clear mid-message, but the flow is still fast enough to not be annoying.

**Recommended pattern — armed state on the button itself:**
- First tap: the Wipe button enters an "armed" state. It changes appearance (e.g. label changes to "Confirm", button turns fully accent-coloured). A 3-second auto-disarm timer starts.
- Second tap within 3 seconds: wipe proceeds normally (save + clear).
- If 3 seconds pass with no second tap, the button silently resets to its normal state.
- Tapping anywhere else on the screen (outside the button) should also disarm.

This avoids modals and overlay dialogs, which require precise interaction and interrupt the visual flow.

**Implementation notes:**
- Use a CSS class `wipe-btn--armed` to drive the visual state change
- Store `wipeArmed = false` and `wipeTimerId = null` in module scope
- On first tap: set `wipeArmed = true`, add class, start `setTimeout(disarm, 3000)`
- On second tap (if armed): `clearTimeout`, call `doWipe()`, disarm
- `disarm()` clears the timer, removes the class, sets `wipeArmed = false`
- Add a `pointerdown` listener on `document` to disarm when touching outside the button

---

### 4. Shortcuts panel

**Purpose:** Quick-tap chips for common responses — yes, no, maybe, later, the user's name, the names of frequent visitors, etc. — so the user doesn't have to type them every time. This is the highest-frequency interaction for short confirmatory replies.

**Visual placement:**

In landscape mode, the screen is wide and short. The shortcuts live as a horizontally scrollable row of chips sitting directly above the toolbar, below the text canvas. This uses very little vertical space (around 52px) and gives each chip plenty of tap area. In the rare portrait orientation, the row should scroll horizontally the same way.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   I need a nurse                                            │  ← text canvas
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  [Yes]  [No]  [Maybe]  [Later]  [Water]  [Riz]  [+]        │  ← shortcut chips
├─────────────────────────────────────────────────────────────┤
│         [ ⏱ History ]          [ 🗑 Wipe ]                  │  ← toolbar
└─────────────────────────────────────────────────────────────┘
```

**Behaviour:**

- Tapping a chip **replaces** the editor content with that phrase and calls `fitText()`. The cursor moves to the end.
- The user can then immediately show the text, or continue typing after the chip text (since it's now in the editor, they can just keep going).
- The `[+]` chip at the end of the row opens an "add shortcut" input.

**Defaults:**

Pre-populate on first launch with: `["Yes", "No", "Maybe", "Later", "Thank you", "One moment"]`. Store in localStorage under `speak_shortcuts`. Only write the defaults if the key is absent — don't overwrite user-edited shortcuts on upgrade.

**Adding a shortcut:**

Tapping `[+]` opens a compact bottom sheet (same slide-up animation as the history panel, but shorter — about 40% screen height) with:
- A text input: "Add a shortcut" placeholder
- A "Save" button
- A "Cancel" button

The shortcut is appended to the end of the array (before the `[+]` chip). Limit to 30 characters per shortcut. After saving, dismiss the sheet and scroll the chip row to show the new chip.

**Editing and deleting:**

Long-press (500ms `pointerdown` hold) on any chip enters **edit mode** for that chip:
- The chip text becomes an inline input pre-filled with the current text
- A small trash icon appears on the chip
- Tapping the trash deletes the shortcut
- Pressing Enter or tapping elsewhere confirms the rename

Alternatively, if inline editing feels fiddly at the chip scale, implement a dedicated "Manage shortcuts" sheet: tapping the `[+]` when shortcuts already exist shows both "Add new" and "Manage" options, and "Manage" opens a full-height list view where shortcuts can be renamed, reordered (drag or up/down arrows), and deleted. This is less seamless but more reliable for one-handed use.

**Reordering:**

Users will want their most-used shortcuts first. The simpler approach: a long-press drag to reorder within the chip row (using the same `pointerdown`/`pointermove`/`pointerup` sequence, no library needed). The "Manage" sheet approach above with up/down arrows is more accessible but uglier.

**Persistence:**

```javascript
const SHORTCUTS_KEY = 'speak_shortcuts';
const DEFAULT_SHORTCUTS = ['Yes', 'No', 'Maybe', 'Later', 'Thank you', 'One moment'];

function getShortcuts() {
  try {
    const raw = localStorage.getItem(SHORTCUTS_KEY);
    return raw ? JSON.parse(raw) : null;
  } catch { return null; }
}

function initShortcuts() {
  if (getShortcuts() === null) {
    localStorage.setItem(SHORTCUTS_KEY, JSON.stringify(DEFAULT_SHORTCUTS));
  }
}
```

**Chip styling:**

Chips should be visually distinct from the toolbar buttons — smaller, pill-shaped (`border-radius: 999px`), with a slightly lighter background than the surface. Selected/active state on press. The chip row itself has no visible border or divider — it floats between the canvas and the toolbar. Overflow hidden on the row container, so chips don't wrap — horizontal scroll only.

```css
.shortcut-row {
  display: flex;
  gap: 8px;
  padding: 6px 18px;
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none;
  flex-shrink: 0;
}
.shortcut-row::-webkit-scrollbar { display: none; }

.chip {
  flex-shrink: 0;
  padding: 9px 18px;
  border-radius: 999px;
  background: var(--surface);
  border: 1px solid var(--border);
  font-family: 'Epilogue', system-ui, sans-serif;
  font-weight: 700;
  font-size: 15px;
  color: var(--text);
  cursor: pointer;
  touch-action: manipulation;
  white-space: nowrap;
  transition: transform 0.1s, background 0.1s;
}
.chip:active { transform: scale(0.93); background: var(--accent-bg); }

.chip-add {
  color: var(--accent);
  border-color: rgba(255,99,64,0.22);
  background: var(--accent-bg);
}
```

---

### 5. Screen wake lock

**Desired behaviour:** While the editor has content and the history panel is closed, request a screen wake lock so the display doesn't dim while someone is reading the message.

**Implementation notes:**
- Use the [Screen Wake Lock API](https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API): `navigator.wakeLock.request('screen')`
- Acquire the lock when text is present (`editor.textContent.trim()` is non-empty)
- Release it when the editor is cleared or when the history panel is open (user is navigating, not showing text)
- Re-acquire on `visibilitychange` if the page becomes visible again and text is present (the lock is always released when the page is hidden)
- Wrap everything in a feature-detect: `if ('wakeLock' in navigator)`; fail silently if unsupported

---

### 6. Minor UX polish

These are smaller items to address alongside the above:

- **Keyboard-aware toolbar:** On iOS/Android, when the software keyboard is open, the toolbar can end up behind it or produce a double-bottom-bar. Use `window.visualViewport` offset (`visualViewport.offsetTop` + `visualViewport.height`) to pin the toolbar just above the keyboard rather than at the physical bottom of the screen.
- **History entry delete:** Add a small trash icon on each history entry (appears on tap/hold or always visible) so the user can prune old messages they don't need.
- **Restore doesn't duplicate:** When tapping a history item to restore it, check if that exact text is already at the top of history. If so, don't push it again when the next auto-save or wipe fires. Use the `lastSavedText` tracking from improvement #1.
- **Font loading fallback:** If Google Fonts fails (offline/poor hospital wifi), `system-ui` is already in the stack, but the weight may differ. Add `font-weight: 900` and `font-style: normal` to the `@font-face` fallback chain so the layout doesn't shift.

---

## Deployment

This is a zero-config static site. To deploy on Vercel:

```bash
vercel --prod
```

Or connect the GitHub repo to Vercel and it will auto-deploy on push. No `vercel.json` needed for a single-file static site, but adding one is harmless:

```json
{
  "cleanUrls": true
}
```

The app works fully offline after first load (all logic is inline). A service worker / PWA manifest would make it installable with true offline support — worth considering if the user wants to add the app to their home screen and use it in areas with no signal.

---

## Design constraints (don't change these)

- **No frameworks.** Vanilla JS only. The simplicity is a feature — it means the file can be copy-pasted anywhere, emailed, opened from Files, etc.
- **Single file.** All CSS and JS stays inline in `index.html`. No separate `.css` or `.js` files. This makes it trivially portable.
- **No build step.** Edit and reload. That's the whole dev loop.
- **Font:** Epilogue 700/900 from Google Fonts. Don't change this — it was chosen for legibility at large sizes and works well for this exact use case.
- **Colour system:** CSS custom properties on `:root`. All colours go through variables; never hardcode a hex value outside the variable declarations.
- **localStorage keys:** `speak_v1` (history array), `speak_theme` (theme preference), `speak_shortcuts` (shortcuts array), `speak_nudge_dismissed` (portrait nudge dismissed flag). Don't rename these; existing users would lose their data.
