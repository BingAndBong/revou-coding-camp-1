# Design Document

## Overview

The Cat Clicker App is a self-contained single-page web application delivered as a single HTML file. It has no build step, no server, and no external dependencies beyond the Tailwind CSS CDN. All logic runs in the browser via Vanilla JavaScript.

The core experience: a cat character sits centred on the screen. Clicking it immediately triggers a randomly chosen Reaction — a facial expression change, a speech bubble, and a synthesised sound effect. A persistent click counter tracks total interactions and survives page reloads via `localStorage`. Rapid clicking (≥5 clicks in 2 seconds) escalates the experience into "Frenzy" mode with a distinct reaction pool. Leaving the cat idle for 5 seconds starts looping idle animations. The app works on any viewport from 320 px to 2560 px and is fully keyboard- and screen-reader-accessible.

**Technology constraints:**
- HTML5, Tailwind CSS (via CDN `<script>` tag), Vanilla JavaScript (ES2020+)
- No bundler, no npm, no framework, no backend
- Single `.html` file (or one `.html` + one `.js` file kept flat)
- Web Audio API for synthesised sound effects
- `localStorage` for counter persistence

---

## Architecture

The app follows a **single-module state-machine** pattern. All state lives in one plain JavaScript object. Event handlers read from and write to that object, then call a render function that reconciles the DOM to the current state. No virtual DOM — targeted DOM mutations only.

```
┌─────────────────────────────────────────────────┐
│                  Browser (DOM)                  │
│                                                 │
│  ┌──────────┐   click/keydown   ┌────────────┐  │
│  │  Cat UI  │ ──────────────► │  EventBus   │  │
│  └──────────┘                  └─────┬──────┘  │
│                                      │          │
│                               ┌──────▼──────┐  │
│                               │  Controller  │  │
│                               │  (handlers)  │  │
│                               └──────┬──────┘  │
│                                      │          │
│                    ┌─────────────────▼────────┐ │
│                    │       AppState            │ │
│                    │  clickCount, comboClicks, │ │
│                    │  comboActive, idleTimer,  │ │
│                    │  reactionHistory, isMuted │ │
│                    └─────────────────┬────────┘ │
│                                      │          │
│                    ┌─────────────────▼────────┐ │
│                    │       Renderer            │ │
│                    │  updateCounter()          │ │
│                    │  showReaction()           │ │
│                    │  setExpression()          │ │
│                    │  showComboIndicator()     │ │
│                    │  setIdleAnimation()       │ │
│                    └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**Data flow for a click event:**
1. User clicks cat → `handleCatClick()` fires
2. Controller updates `AppState` (increment count, record timestamp, pick reaction)
3. Controller calls Renderer functions to update specific DOM nodes
4. Combo and idle timers are reset/set via `setTimeout`/`clearTimeout`

No global variables are exposed; everything is wrapped in an IIFE or ES module.

---

## Components and Interfaces

### 1. CatDisplay

Renders the cat character. Implemented as a `<div>` with an inner `<img>` (or CSS emoji art fallback) plus a loading placeholder `<div>`.

**Responsibilities:**
- Show the cat at ≥200×200 px, centred in the viewport
- Show a same-size placeholder while loading (prevents layout shift — Requirement 1.3)
- Apply expression CSS classes (happy, surprised, sleepy, angry, heart-eyes, etc.)
- Play idle animation classes
- Expose a focusable element with `role="button"`, `tabindex="0"`, and `aria-label`

**DOM structure:**
```html
<div id="cat-container" class="...tailwind classes...">
  <div id="cat-placeholder" class="w-52 h-52 bg-gray-200 rounded-full animate-pulse"></div>
  <img id="cat-img" src="cat.svg" alt=""
       role="button" tabindex="0"
       aria-label="Cat: clicked 0 times"
       class="w-52 h-52 object-contain hidden" />
</div>
```

### 2. SpeechBubble

Overlays a short text/emoji message above or beside the cat.

**Responsibilities:**
- Appear within 50 ms of a click (Requirement 2.3)
- Auto-dismiss after the reaction duration (1000–3000 ms)
- Be immediately hidden when a new reaction interrupts (Requirement 2.6)

**DOM structure:**
```html
<div id="speech-bubble" class="absolute ... hidden" aria-live="polite">
  <span id="speech-text"></span>
</div>
```

`aria-live="polite"` causes screen readers to announce new messages without interrupting ongoing speech.

### 3. ClickCounter

Displays the persistent click count.

**Responsibilities:**
- Always visible (Requirement 4.1)
- Animate with a scale-up transition (100–300 ms) on each increment (Requirement 4.2)
- Read initial value from `localStorage` on page load (Requirement 4.4)

**DOM structure:**
```html
<div id="counter-wrapper" class="...">
  <span class="label">Clicks</span>
  <span id="counter-display" class="text-4xl font-bold">0</span>
</div>
```

Animation: add CSS class `counter-pop` → remove after 300 ms via `setTimeout`.

### 4. ComboIndicator

A "FRENZY!" badge shown only during active Combo state.

**Responsibilities:**
- Appear/disappear without overlapping cat or reaction area (Requirement 5.2)
- Not interfere with other UI elements

**DOM structure:**
```html
<div id="combo-indicator" class="hidden fixed top-4 right-4 ...">FRENZY! 🔥</div>
```

### 5. MuteToggle

A toggle button for silencing sound effects.

**Responsibilities:**
- Persist mute preference to `AppState.isMuted`
- Not reload the page (Requirement 8.5)

**DOM structure:**
```html
<button id="mute-toggle" aria-pressed="false" class="...">🔊</button>
```

### 6. ReactionEngine

Pure logic module (no DOM access). Selects the next reaction according to the rules.

**Interface:**
```javascript
// Returns a Reaction object
function pickReaction(pool, history, comboActive) { ... }

// Returns true if combo should activate
function shouldActivateCombo(timestamps, windowMs = 2000, threshold = 5) { ... }

// Returns true if combo should deactivate
function shouldDeactivateCombo(lastClickTime, windowMs = 2000) { ... }
```

`pickReaction` enforces the no-repeat-twice rule (Requirement 3.2): it filters out the last two reactions from the pool before random selection.

### 7. AudioEngine

Wraps the Web Audio API. Synthesises short tones/beeps mapped to each reaction — no external audio files required.

**Interface:**
```javascript
function initAudio() { ... }           // Creates AudioContext on first user gesture
function playReactionSound(soundId) { ... }  // Plays a synthesised tone
function setMuted(muted) { ... }
```

**Graceful degradation:** `initAudio` checks `typeof AudioContext !== 'undefined'`. If unavailable, all `playReactionSound` calls are no-ops (Requirement 3.5).

### 8. IdleController

Manages the idle timer and animation cycling.

**Interface:**
```javascript
function resetIdleTimer() { ... }   // Call on every click
function startIdle() { ... }        // Called after 5s of no clicks
function stopIdle() { ... }         // Called when user clicks during idle
function nextIdleAnimation() { ... } // Cycles through blink/yawn/look-around
```

Idle animations are CSS classes toggled on `#cat-img`. Each class drives a CSS `@keyframes` animation. When one cycle ends, the `animationend` DOM event triggers `nextIdleAnimation()`.

---

## Data Models

### AppState

```javascript
const AppState = {
  clickCount: 0,              // integer ≥ 0, max 999999
  isMuted: false,             // boolean
  comboActive: false,         // boolean
  comboClickTimestamps: [],   // Array<number> — ms timestamps of recent clicks
  comboDeactivateTimer: null, // setTimeout handle
  idleTimer: null,            // setTimeout handle
  idleAnimationName: null,    // 'blink' | 'yawn' | 'look-around' | null
  reactionHistory: [],        // Array<string> — last 2 reaction IDs
  activeReactionTimer: null,  // setTimeout handle for dismissing speech bubble
};
```

### Reaction

```javascript
// Defined statically; not mutated at runtime
{
  id: string,          // unique, e.g. 'happy-paw'
  expression: string,  // CSS class name, e.g. 'expr-happy'
  message: string,     // Speech bubble text/emoji, e.g. '(=^･ω･^=)'
  soundId: string,     // key into AudioEngine sound map
  duration: number,    // ms — how long the reaction lasts (1000–3000)
  isEscalated: boolean // true → only shown during Combo/Frenzy mode
}
```

### ReactionPool layout

```
NORMAL_POOL  (≥10 reactions, isEscalated: false)
ESCALATED_POOL  (≥3 reactions, isEscalated: true)
FULL_POOL = NORMAL_POOL ∪ ESCALATED_POOL  (no overlap — Requirement 5.3)
```

`pickReaction` receives `comboActive` and chooses the appropriate pool.

### Sound Map

```javascript
const SOUNDS = {
  'meow-soft':   { type: 'sine',   freq: 880,  duration: 0.15 },
  'meow-happy':  { type: 'triangle', freq: 1046, duration: 0.2 },
  'purr':        { type: 'sawtooth', freq: 220,  duration: 0.4 },
  'chirp':       { type: 'square',  freq: 1500, duration: 0.1 },
  // ... one entry per reaction
};
```

Each entry drives `OscillatorNode` → `GainNode` → `AudioContext.destination`.

### localStorage Schema

```
Key:   "catClicker_clickCount"
Value: JSON-stringified integer
```

Written synchronously on every click increment. Read once on `DOMContentLoaded`.

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Reaction selection stays within the eligible pool

*For any* active reaction pool, any 2-item reaction history, and any value of `comboActive`, calling `pickReaction` returns a reaction that belongs to the pool appropriate for `comboActive` and whose ID does not appear in the history.

**Validates: Requirements 2.1, 3.2, 5.3**

### Property 2: No-repeat-twice invariant across a sequence

*For any* sequence of `N ≥ 3` reaction selections produced by repeated calls to `pickReaction`, no reaction at position `i` has the same ID as the reaction at position `i-1` or position `i-2`.

**Validates: Requirements 3.2**

### Property 3: Click counter monotonically increments by exactly N

*For any* initial click count `c` and any sequence of `N` click events (1 ≤ N ≤ 200), the click count after all N events equals `c + N` — it never decreases and never increments by anything other than 1 per click.

**Validates: Requirements 2.4, 4.1, 4.5**

### Property 4: localStorage round-trip preserves click count

*For any* integer `n` in the range [0, 999999], serialising `n` to a string, writing it to `localStorage`, reading it back, and parsing it returns exactly `n`.

**Validates: Requirements 4.4**

### Property 5: Combo activation matches rolling-window threshold

*For any* array of click timestamps, `shouldActivateCombo` returns `true` if and only if at least 5 timestamps fall within some rolling 2000 ms window, and returns `false` for all arrays where no such window exists.

**Validates: Requirements 5.1**

### Property 6: Combo pool exclusivity and disjointness

*For any* call to `pickReaction` with `comboActive = true`, the returned reaction has `isEscalated = true`; with `comboActive = false`, the returned reaction has `isEscalated = false`. Additionally, the intersection of `NORMAL_POOL` IDs and `ESCALATED_POOL` IDs is always empty.

**Validates: Requirements 5.3**

### Property 7: Speech bubble message round-trip

*For any* Unicode string `m` assigned as a reaction's `message`, writing `m` to the speech bubble's `textContent` and reading `textContent` back returns the original string `m` unchanged.

**Validates: Requirements 2.3, 3.1, 3.3**

### Property 8: Idle timer resets on every click

*For any* app state where the idle timer is running (regardless of remaining time), a click event cancels the in-flight timer and restarts a fresh 5000 ms idle countdown, so the idle state cannot activate within 5 seconds of any click.

**Validates: Requirements 6.1, 6.2**

### Property 9: Aria-label reflects current click count

*For any* non-negative integer `n` (0 ≤ n ≤ 999999), `generateAriaLabel(n)` returns exactly the string `"Cat: clicked N times"` where `N` is the decimal representation of `n`.

**Validates: Requirements 8.3, 8.4**

### Property 10: Reaction pool data integrity

*For any* pool (normal or escalated), every reaction has a unique `id`, a unique `expression` class name, and a unique `message` string; the normal pool contains at least 10 reactions and the escalated pool contains at least 3 reactions.

**Validates: Requirements 3.1, 3.3**

### Property 11: Next idle animation is always a valid animation name

*For any* call to `nextIdleAnimation()`, the returned value is a member of the set `{'blink', 'yawn', 'look-around'}`.

**Validates: Requirements 6.4**

---

## Error Handling

| Scenario | Handling |
|---|---|
| `localStorage` unavailable (private mode, storage full) | Catch `SecurityError`/`QuotaExceededError`; operate with in-memory counter only; no crash |
| Web Audio API unavailable | `initAudio` returns a no-op stub; all sound calls are silently skipped (Requirement 3.5) |
| Cat image fails to load | `onerror` handler keeps placeholder visible; app remains functional |
| `AudioContext` blocked before user gesture | `initAudio` is deferred to the first click event; first click may be silent, subsequent clicks have audio |
| Reaction pool exhausted after filtering (pool ≤ 2 reactions) | Fall back to full pool without the no-repeat filter rather than throwing |
| `requestAnimationFrame` / CSS animation not supported | Degrade gracefully: skip animation classes, still apply expression and message |

---

## Testing Strategy

### Unit Tests (Vanilla JS, no test framework required — or use Vitest)

Focus on pure logic functions that have no DOM dependency:

- `pickReaction(pool, history, comboActive)` — verify no-repeat rule, pool selection, uniform distribution over many samples
- `shouldActivateCombo(timestamps, windowMs, threshold)` — verify threshold boundary (exactly 4 vs 5 clicks, overlapping windows)
- `shouldDeactivateCombo(lastClickTime, windowMs)` — verify 2-second boundary
- `localStorage` read/write helpers — verify round-trip with mock `localStorage`
- Reaction pool definitions — assert no overlap between normal and escalated pools, all IDs unique, all messages unique, all expressions unique

### Property-Based Tests (fast-check)

Each property test runs a minimum of **100 iterations**. Tag format: `Feature: cat-clicker-app, Property {N}: {title}`.

| Property | Generator inputs | Assertion |
|---|---|---|
| P1 – Eligible pool selection | Random pool arrays (3–13 items), random 2-item history, random `comboActive` | Selected item in correct pool ∧ not in last-2 history |
| P2 – No-repeat-twice | Random reaction sequences (length 3–50) | `seq[i] ≠ seq[i-1]` and `seq[i] ≠ seq[i-2]` for all `i ≥ 2` |
| P3 – Counter monotonic increment | Random initial count (0–999999), random N clicks (1–200) | `finalCount === initialCount + N` |
| P4 – localStorage round-trip | Random integers 0–999999 | `parse(serialize(n)) === n` |
| P5 – Combo threshold | Random timestamp arrays with varying density | Combo activates ↔ rolling-window count ≥ 5 |
| P6 – Combo pool exclusivity | Random `comboActive` bool | Selected reaction's `isEscalated` matches `comboActive`; `NORMAL_POOL ∩ ESCALATED_POOL = ∅` |
| P7 – Message round-trip | Random Unicode strings (excluding null bytes) | `textContent` after DOM assignment equals original string |
| P8 – Idle timer reset | Fake timers; random click sequences with random inter-click delays | Idle fires ↔ no click in prior 5000 ms; resets on every click |
| P9 – Aria-label format | Random integers 0–999999 | `generateAriaLabel(n) === \`Cat: clicked ${n} times\`` |
| P10 – Pool data integrity | Static assertion on pool constants | All IDs/expressions/messages unique; size ≥ 10 normal, ≥ 3 escalated |
| P11 – Next idle animation membership | Arbitrary call | Result ∈ `{'blink', 'yawn', 'look-around'}` |

**Library:** [fast-check](https://fast-check.dev/) — compatible with any test runner; run with `--run` flag for single execution (no watch mode).

### Integration / Smoke Tests

- Page loads without console errors in Chrome and Firefox
- Keyboard Tab reaches the cat; Space/Enter triggers a reaction (Requirement 8.1–8.2)
- Counter value survives a page reload (localStorage — Requirement 4.4)
- Mute toggle silences sounds without reload (Requirement 8.5)
- Layout renders without horizontal scroll at 320 px and 2560 px viewport widths (Requirement 7.1)

### Accessibility Checks

- Run axe-core or Lighthouse accessibility audit; target zero critical violations
- Verify `aria-label` updates to `"Cat: clicked N times"` after each click (Requirement 8.3–8.4)
