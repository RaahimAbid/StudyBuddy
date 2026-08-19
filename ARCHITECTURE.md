# Architecture & Design Decisions

## Overview

Like LinguaBridge, StudyBuddy is a single static HTML file (markup, styles, and logic together) with no backend, build tools, or frameworks — a deliberate choice to keep the entire system traceable end-to-end. The added complexity in this project isn't architectural (still no server, no database) but algorithmic and visual: a real spaced-repetition algorithm, and a custom WebGL shader background.

## Key design decisions

### 1. SM-2 as a pure function
```js
function calculateSM2(card, quality) { ... }
```
`calculateSM2` takes a card's current scheduling state plus a quality rating and returns a new state, with no side effects — it doesn't touch the DOM, localStorage, or any global variable directly. This keeps the algorithm testable and easy to reason about in isolation from the UI that calls it. All the "wiring" (persisting the result, updating what's on screen) happens in the calling code, not inside the algorithm itself.

On a failed rating, `repetitions` and `interval` reset, but `easeFactor` deliberately is **not** reset — a card that has historically been difficult keeps a lower ease factor even after restarting, so its future intervals grow more cautiously. This matches SM-2's intent: ease factor is a long-term signal of a specific card's difficulty for a specific learner, not a per-session value.

### 2. Separating "the deck" from "today's session"
```js
let deck = [];         // every card + its permanent SM-2 state
let sessionQueue = [];  // indices into `deck`, order still to review today
```
Initially, reviewing simply cycled through the deck array in a fixed loop, which meant difficulty had no effect on review order — clearly wrong for a spaced-repetition tool. The fix was to separate long-term state (`deck`) from short-term review order (`sessionQueue`), and give the queue its own logic: a card rated "Again" gets reinserted a few positions ahead (resurfaces soon, same session); "Hard" reinserts further out but still today; "Good"/"Easy" are removed from today's queue entirely, since their computed interval is 1+ days out. This is the same approach used by mainstream spaced-repetition tools (e.g. Anki's "again"/"learning" queue), scaled down to fit a single file.

### 3. Structured JSON output instead of prompt-only formatting
Flashcard generation needs data StudyBuddy can loop through in code — not prose. Rather than asking Gemini in plain English to "return JSON" (which can drift: extra commentary, markdown fences, minor formatting slips), the request uses `generationConfig.responseSchema` to constrain the model's output to an array of `{question, answer}` objects, both fields required. This is enforced by the API itself rather than hoped for via instructions, and is the appropriate approach whenever an LLM's output needs to be reliably machine-parsed.

### 4. Client-side API key storage (same trade-off as LinguaBridge)
The Gemini API key lives in the user's own `localStorage`, and requests are made directly from the browser. This is a deliberate simplification for a single-user, bring-your-own-key tool: every user supplies and stores their own key, so there's no shared secret to protect. A production multi-tenant deployment would need a backend/serverless proxy to hold the key server-side instead — noted here explicitly as the first thing that would need to change for that use case.

### 5. Whole-deck persistence, keyed by history entry
Each generated deck is saved as a full snapshot — original notes (for a readable title), plus every card including its live SM-2 state — under one `localStorage` entry per deck (`studybuddy_history`, an array of entries). Saving the *entire* card state (not just question/answer text) is what allows resuming a deck later without losing scheduling progress. `persistCurrentDeck()` runs after every single rating, not just on some exit event, so progress is never lost to an unexpected browser close.

### 6. The animated background: Three.js + a custom fragment shader
The moving background is one full-screen rectangle (`PlaneGeometry(2,2)`) viewed through an orthographic camera, with a GLSL fragment shader computing color per pixel from domain-warped 3D simplex noise, animated over time and gently offset by cursor position. This runs entirely on the GPU via `renderer.setAnimationLoop()` (which itself wraps `requestAnimationFrame`), keeping the CPU/JavaScript thread free and the animation synced to the display's refresh rate.

## Challenges faced and how they were solved

### Fixed-order review ignoring difficulty
Initial implementation advanced through the deck with simple index cycling (`currentIndex = (currentIndex + 1) % deck.length`), so a card's rating had no effect on when it reappeared. This was identified through testing and fixed by introducing the `sessionQueue` design described above — a genuinely different data structure, not just a tweak to the increment logic.

### Gemini model deprecation
Mid-development, `gemini-2.5-flash-lite` (used in both this project and LinguaBridge) returned an error stating it was no longer available to new API keys, with the API's own error message naming its replacement (`gemini-3.5-flash-lite`). Both projects were updated to the new model name. This is treated as a normal maintenance task rather than a bug — third-party AI model names and availability change on a timescale of months, and reading the API's own error text was sufficient to resolve it immediately.

### Visual direction changes mid-build
The UI went through three distinct visual phases during development: a minimal "study desk" theme, a "corkboard/funky" theme (hand-drawn fonts, pinned photo aesthetic), and finally a full pivot to an ultra-vibrant neon-glassmorphism look with a live WebGL shader background, per explicit direction. Each phase fully replaced the previous one's styling rather than layering on top, since blending distinct aesthetic directions (e.g. corkboard + neon-glass) would have produced an incoherent result. After the final pivot, a follow-up fix was needed: the initial dark-glass flashcard surface had low contrast against the also-dark shader background, resolved by switching specifically the *card face* (not the surrounding chrome) to a light, high-contrast "paper" surface — balancing the vibrant brief with actual readability.

## Possible future improvements

- Move API calls behind a backend/serverless proxy to support a shared (non-bring-your-own-key) deployment
- Support PDF/DOCX note uploads via a dedicated parsing library
- Add automated tests for `calculateSM2` specifically, given it's a pure function and straightforward to test in isolation
- Allow editing/deleting individual generated flashcards before starting a review session
- Surface more of a card's SM-2 history (e.g. a small graph of past intervals) rather than just the next scheduled date
