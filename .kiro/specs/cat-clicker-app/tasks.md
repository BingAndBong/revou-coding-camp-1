# Implementation Plan: Cat Clicker App

## Overview

Build a single HTML file (plus an optional flat `.js` companion) that delivers the complete Cat Clicker experience: a centred cat character, random reactions, a persistent click counter, a combo/frenzy mode, idle animations, and full keyboard/screen-reader accessibility. All logic runs in the browser via Vanilla JavaScript (ES2020+). Tailwind CSS is loaded from CDN. No build step, no backend.

Implementation proceeds in layers: static HTML shell → pure-logic modules (ReactionEngine, AudioEngine, IdleController) with their property tests → state and rendering (AppState, Renderer) → interactive features (click, counter, combo, idle) → accessibility and responsive polish.

## Tasks

- [~] 1. Create the static HTML shell and project structure
  - Create `index.html` with the full HTML5 document skeleton
  - Add the Tailwind CSS CDN `<script>` tag in `<head>`
  - Scaffold all named DOM elements: `#cat-container`, `#cat-placeholder`, `#cat-img`, `#speech-bubble`, `#speech-text`, `#counter-wrapper`, `#counter-display`, `#combo-indicator`, `#mute-toggle`
  - Set `role="button"`, `tabindex="0"`, and initial `aria-label="Cat: clicked 0 times"` on `#cat-img`
  - Apply Tailwind layout classes so the cat is horizontally and vertically centred in the viewport
  - Display the placeholder `<div>` by default; keep `#cat-img` hidden until `load` fires
  - Add a `<script>` tag at the bottom of `<body>` referencing `app.js` (or an inline `<script>` block)
  - _Requirements: 1.1, 1.2, 1.3, 7.3, 8.1, 8.3_

- [ ] 2. Implement ReactionEngine — pure logic, no DOM
  - [-] 2.1 Define the Reaction data model and static pool constants
    - Create the `Reaction` object shape (`id`, `expression`, `message`, `soundId`, `duration`, `isEscalated`)
    - Define `NORMAL_POOL` with ≥10 reactions (`isEscalated: false`), each with a unique `id`, `expression`, `message`, and `soundId`
    - Define `ESCALATED_POOL` with ≥3 reactions (`isEscalated: true`), no IDs overlapping `NORMAL_POOL`
    - _Requirements: 3.1, 3.3, 5.3_

  - [ ]* 2.2 Write property test — Pool data integrity (Property 10)
    - **Property 10: Reaction pool data integrity**
    - Assert that every reaction in each pool has a unique `id`, unique `expression`, and unique `message`; normal pool size ≥ 10; escalated pool size ≥ 3; `NORMAL_POOL ∩ ESCALATED_POOL = ∅`
    - **Validates: Requirements 3.1, 3.3**

  - [~] 2.3 Implement `pickReaction(pool, history, comboActive)`
    - Select the correct pool based on `comboActive`
    - Filter out reactions whose IDs appear in the last-2 `history` array
    - Choose uniformly at random from the filtered pool (fall back to unfiltered pool if ≤2 reactions remain after filtering)
    - Return the selected `Reaction` object
    - _Requirements: 2.1, 3.2, 5.3_

  - [ ]* 2.4 Write property test — Eligible pool selection (Property 1)
    - **Property 1: Reaction selection stays within the eligible pool**
    - Generate random pool arrays (3–13 items), random 2-item history, random `comboActive`; assert selected item is in the correct pool and its ID is not in the last-2 history
    - **Validates: Requirements 2.1, 3.2, 5.3**

  - [ ]* 2.5 Write property test — No-repeat-twice invariant (Property 2)
    - **Property 2: No-repeat-twice invariant across a sequence**
    - Drive `pickReaction` for N = 3–50 sequential calls; assert `seq[i].id ≠ seq[i-1].id` and `seq[i].id ≠ seq[i-2].id` for all `i ≥ 2`
    - **Validates: Requirements 3.2**

  - [~] 2.6 Implement `shouldActivateCombo(timestamps, windowMs, threshold)`
    - Return `true` if and only if at least `threshold` (default 5) timestamps fall within any rolling `windowMs` (default 2000 ms) window
    - _Requirements: 5.1_

  - [ ]* 2.7 Write property test — Combo activation threshold (Property 5)
    - **Property 5: Combo activation matches rolling-window threshold**
    - Generate timestamp arrays with varying densities; assert `shouldActivateCombo` returns `true` ↔ a rolling 2000 ms window contains ≥5 timestamps
    - **Validates: Requirements 5.1**

  - [~] 2.8 Implement `shouldDeactivateCombo(lastClickTime, windowMs)`
    - Return `true` if `Date.now() - lastClickTime ≥ windowMs`
    - _Requirements: 5.5_

- [ ] 3. Implement AudioEngine — Web Audio API wrapper
  - [~] 3.1 Implement `initAudio()`, `playReactionSound(soundId)`, and `setMuted(muted)`
    - Check `typeof AudioContext !== 'undefined'`; if unavailable, all functions become no-ops
    - Defer `AudioContext` creation to the first user click to satisfy browser autoplay policy
    - Define `SOUNDS` map (at least one entry per reaction's `soundId`) using `OscillatorNode → GainNode → destination`
    - `setMuted` stores the flag in `AppState.isMuted` and silences subsequent calls
    - _Requirements: 3.4, 3.5, 8.5_

- [ ] 4. Implement AppState and localStorage helpers
  - [-] 4.1 Define the `AppState` object with all fields from the design
    - Fields: `clickCount`, `isMuted`, `comboActive`, `comboClickTimestamps`, `comboDeactivateTimer`, `idleTimer`, `idleAnimationName`, `reactionHistory`, `activeReactionTimer`
    - Initialise `clickCount` by reading from `localStorage` key `"catClicker_clickCount"` on `DOMContentLoaded`; default to 0 if absent or unparseable
    - _Requirements: 4.3, 4.4_

  - [ ]* 4.2 Write property test — localStorage round-trip (Property 4)
    - **Property 4: localStorage round-trip preserves click count**
    - Generate random integers 0–999999; serialise to string, write to a mock `localStorage`, read back and parse; assert the result equals the original integer
    - **Validates: Requirements 4.4**

- [ ] 5. Implement CatDisplay and the Renderer
  - [~] 5.1 Implement `setExpression(expressionClass)` and idle animation toggling
    - Remove all `expr-*` CSS classes from `#cat-img`, then add the given class
    - Define CSS `@keyframes` for `blink`, `yawn`, and `look-around` idle animations as `<style>` rules in `index.html`
    - `setIdleAnimation(name)` removes existing animation classes and applies the new one
    - _Requirements: 2.5, 6.1, 6.3_

  - [~] 5.2 Implement `showReaction(reaction)` and `dismissReaction()`
    - Set `#speech-text` text content to `reaction.message`, un-hide `#speech-bubble`
    - Apply `reaction.expression` via `setExpression`
    - Clear any in-flight `activeReactionTimer`; set a new one for `reaction.duration` ms that calls `dismissReaction()`
    - `dismissReaction()` hides `#speech-bubble` and calls `setExpression('expr-idle')`
    - _Requirements: 2.2, 2.3, 2.5, 2.6_

  - [ ]* 5.3 Write property test — Speech bubble message round-trip (Property 7)
    - **Property 7: Speech bubble message round-trip**
    - Generate random Unicode strings (excluding null bytes); assign to `element.textContent` in a minimal DOM fixture (jsdom); assert `element.textContent` equals the original string
    - **Validates: Requirements 2.3, 3.1, 3.3**

- [ ] 6. Implement ClickCounter display and persistence
  - [~] 6.1 Implement `updateCounter(newCount)` in the Renderer
    - Set `#counter-display` text to `newCount`
    - Add CSS class `counter-pop` to `#counter-display` (define the class as a 100–300 ms scale-up `@keyframes` animation in `index.html`)
    - Remove `counter-pop` after 300 ms via `setTimeout`
    - Write `newCount` to `localStorage["catClicker_clickCount"]` inside a try/catch (handle `SecurityError`/`QuotaExceededError` silently)
    - _Requirements: 4.1, 4.2, 4.4_

  - [ ]* 6.2 Write property test — Click counter monotonic increment (Property 3)
    - **Property 3: Click counter monotonically increments by exactly N**
    - Start from a random `c` (0–999999), simulate N random clicks (1–200); assert `finalCount === c + N` after all events
    - **Validates: Requirements 2.4, 4.1, 4.5**

- [~] 7. Implement the main click handler (`handleCatClick`)
  - Wire `click` and `keydown` (Space / Enter) event listeners on `#cat-img`
  - On each activation:
    1. Call `initAudio()` (deferred AudioContext creation)
    2. Cancel any in-flight `activeReactionTimer`; immediately hide `#speech-bubble`
    3. Increment `AppState.clickCount`; call `updateCounter(AppState.clickCount)`
    4. Push `Date.now()` to `AppState.comboClickTimestamps`; trim timestamps older than 2000 ms
    5. Evaluate `shouldActivateCombo`; update `AppState.comboActive` and show/hide `#combo-indicator` accordingly
    6. Reset the combo deactivation timer (clear and restart 2000 ms `setTimeout`)
    7. Call `pickReaction(pool, AppState.reactionHistory, AppState.comboActive)`
    8. Update `AppState.reactionHistory` (keep last 2 IDs)
    9. Call `showReaction(reaction)`; call `playReactionSound(reaction.soundId)`
    10. Update `aria-label` on `#cat-img` to `"Cat: clicked N times"`
    11. Call `resetIdleTimer()`
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.6, 4.4, 4.5, 5.1, 5.2, 5.4, 8.2, 8.3, 8.4_

- [ ] 8. Implement IdleController
  - [~] 8.1 Implement `resetIdleTimer()`, `startIdle()`, `stopIdle()`, and `nextIdleAnimation()`
    - `resetIdleTimer()` clears `AppState.idleTimer` and starts a new 5000 ms `setTimeout` that calls `startIdle()`
    - `startIdle()` sets `AppState.idleAnimationName` to a random member of `{'blink', 'yawn', 'look-around'}`; calls `setIdleAnimation(name)`
    - `stopIdle()` clears the animation class from `#cat-img` and sets `AppState.idleAnimationName` to `null`
    - `nextIdleAnimation()` picks a new member of `{'blink', 'yawn', 'look-around'}` (different from the current one); calls `setIdleAnimation` with it
    - Attach an `animationend` listener on `#cat-img` that calls `nextIdleAnimation()` while in idle state
    - _Requirements: 6.1, 6.2, 6.3, 6.4_

  - [ ]* 8.2 Write property test — Idle timer resets on every click (Property 8)
    - **Property 8: Idle timer resets on every click**
    - Use fake timers (via fast-check's `modelRun` or sinon); generate random click sequences with random inter-click delays; assert idle state cannot activate within 5000 ms of any click
    - **Validates: Requirements 6.1, 6.2**

  - [ ]* 8.3 Write property test — Next idle animation membership (Property 11)
    - **Property 11: Next idle animation is always a valid animation name**
    - Call `nextIdleAnimation()` for any arbitrary current state; assert the returned value is in `{'blink', 'yawn', 'look-around'}`
    - **Validates: Requirements 6.4**

- [~] 9. Checkpoint — core interaction complete
  - Ensure all tests pass, ask the user if questions arise.
  - Verify in a browser: clicking the cat shows a reaction within 50 ms, the counter increments, rapid clicks trigger FRENZY, and 5 s of inactivity starts idle animation.

- [ ] 10. Implement ComboIndicator and MuteToggle
  - [~] 10.1 Implement `showComboIndicator()` and `hideComboIndicator()`
    - Toggle the Tailwind `hidden` class on `#combo-indicator`
    - Ensure `#combo-indicator` is `fixed top-4 right-4` (or equivalent) so it never overlaps the cat or speech bubble
    - _Requirements: 5.2, 5.6_

  - [~] 10.2 Implement the MuteToggle click handler
    - Toggle `AppState.isMuted`; call `setMuted(AppState.isMuted)`
    - Update `#mute-toggle` icon (🔊 / 🔇) and `aria-pressed` attribute
    - _Requirements: 8.5_

- [ ] 11. Implement accessibility attributes and aria-label updates
  - [~] 11.1 Implement `generateAriaLabel(n)` and wire it into `updateCounter`
    - Return exactly the string `"Cat: clicked N times"` where N is `n` as a decimal integer
    - Call `generateAriaLabel(AppState.clickCount)` after every counter update and set it on `#cat-img`'s `aria-label`
    - _Requirements: 8.3, 8.4_

  - [ ]* 11.2 Write property test — Aria-label format (Property 9)
    - **Property 9: Aria-label reflects current click count**
    - Generate random integers 0–999999; assert `generateAriaLabel(n) === \`Cat: clicked ${n} times\``
    - **Validates: Requirements 8.3, 8.4**

  - [~] 11.3 Verify keyboard focus indicator and mobile tap target
    - Add a Tailwind `focus:ring-4 focus:ring-blue-400` (or equivalent) class to `#cat-img` to satisfy visible focus (Requirement 8.1)
    - Ensure `#cat-img` has at least 44×44 CSS pixels tap target on viewports < 640 px using Tailwind responsive classes (Requirement 7.2)
    - _Requirements: 7.2, 8.1_

- [ ] 12. Implement Combo pool exclusivity verification and pool integration
  - [ ]* 12.1 Write property test — Combo pool exclusivity (Property 6)
    - **Property 6: Combo pool exclusivity and disjointness**
    - For random `comboActive` boolean, call `pickReaction`; assert returned reaction's `isEscalated` matches `comboActive`; assert `NORMAL_POOL ∩ ESCALATED_POOL = ∅`
    - **Validates: Requirements 5.3**

- [~] 13. Final checkpoint — full integration and accessibility
  - Ensure all tests pass, ask the user if questions arise.
  - Confirm no horizontal scroll at 320 px and 2560 px viewport widths (Requirement 7.1).
  - Run axe-core or Lighthouse; resolve any critical accessibility violations.
  - Confirm `aria-label` reads `"Cat: clicked N times"` after each click using a screen reader or browser dev tools.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Property tests use [fast-check](https://fast-check.dev/); run with `--run` flag (e.g. `vitest --run`) — do NOT use watch mode
- Checkpoints (Tasks 9 and 13) validate incremental progress before moving to the next layer
- `pickReaction` and `shouldActivateCombo` are pure functions with no DOM dependency — they can be unit- and property-tested without a browser or jsdom
- The combo deactivation timer and idle timer are managed with `clearTimeout`/`setTimeout` handles stored in `AppState`; always clear before resetting to avoid double-firing
- If `AudioContext` creation is blocked before the first user gesture, `initAudio` is deferred to the first click — the first click may be silent, which is acceptable

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "4.1"] },
    { "id": 1, "tasks": ["2.3", "2.6", "2.8", "3.1", "2.2"] },
    { "id": 2, "tasks": ["2.4", "2.5", "2.7", "4.2"] },
    { "id": 3, "tasks": ["5.1", "5.2", "6.1"] },
    { "id": 4, "tasks": ["5.3", "6.2", "8.1"] },
    { "id": 5, "tasks": ["8.2", "8.3", "10.1", "10.2", "11.1"] },
    { "id": 6, "tasks": ["11.2", "11.3", "12.1"] }
  ]
}
```
