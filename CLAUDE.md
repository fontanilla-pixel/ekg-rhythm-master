# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

EKG Rhythm Master: ICU Edition is a single-page browser quiz game that teaches EKG rhythm recognition. A patient monitor animates a scrolling waveform on a `<canvas>`, the player picks the rhythm name from multiple-choice options, and scoring/difficulty/health mechanics track progress across a session.

The entire application — HTML, CSS, and JavaScript — lives in one file: `index.html`. There is no build step, no package manager, and no external JS dependencies (only Google Fonts are loaded remotely). There are no automated tests.

## Development workflow

Because this is a static file with no build tooling:

- **Run it**: open `index.html` directly in a browser, or serve it locally, e.g. `python3 -m http.server` from the repo root and visit `http://localhost:8000/index.html`.
- **Edit it**: modify `index.html` directly — CSS is in the `<style>` block, markup in `<body>`, logic in the trailing `<script>` block.
- **Verify changes**: there is no linter, formatter, or test suite configured. Validate changes by loading the page in a browser and exercising the quiz flow (answer correctly/incorrectly, trigger a game over, resize the window to check the responsive layout below 860px).

## Architecture

Everything is driven by one in-memory data structure and a small set of functions operating on global state — there is no framework, module system, or build pipeline.

### `RHYTHMS` — the content model
An object keyed by rhythm code (e.g. `NSR`, `AFIB`, `VTACH`) where each entry defines:
- `name`, `hr` (heart rate shown on the monitor), `emergency` (bool), `difficulty` (1-4), `tags` (rendered as colored pills — `EMERGENCY`/`STABLE`/rate tags), `info` (the teaching explanation shown after answering)
- `draw(t)`: a hand-tuned waveform function returning a vertical offset for time `t`, using modulo arithmetic over a "beat cycle" to synthesize P-QRS-T morphology (or fibrillation/flatline noise for chaotic rhythms). This is the core trick for adding a new rhythm — model its ECG morphology as a periodic function of `t`.

### Difficulty system
`DIFFICULTY_LEVELS` defines four tiers (INTERN → RESIDENT → FELLOW → ATTENDING), each with a `minCorrect` threshold, a number of answer `choices`, and a `pool` of allowed rhythm difficulty tiers. `getDiffLevel()` derives the current tier from `correctCount`; difficulty rises automatically as the player answers correctly (there's no manual selector).

### Case queue
`buildQueue()` Fisher-Yates shuffles the rhythms eligible for the current difficulty pool into `rhythmQueue`, and `nextFromQueue()` pulls from it, rebuilding when exhausted or when the front no longer matches the (possibly just-changed) eligible pool. This gives no-immediate-repeat behavior within a difficulty tier.

### Canvas rendering loop
`drawLoop()` runs via `requestAnimationFrame`, advancing a horizontal scan position (`scanX`) at `SCAN_SPEED` and calling `currentRhythm.draw(t)` (with `t = Date.now() / 9`) to plot the next waveform point, clearing a small window ahead of the sweep to create the classic "ICU monitor sweep" erase effect. The canvas is resized to its container on load and on window resize.

### Game state and flow
Plain module-level variables (`score`, `streak`, `health`, `correctCount`, `currentRhythm`, `currentChoices`, `hasFailedCurrent`, `rhythmQueue`, `seenCount`) hold all session state — there is no state management library. Flow: `loadPatient()` picks the next case and renders options → `handleAnswer()` scores the choice, updates health/streak/XP, and shows the feedback panel (with a retry option on a wrong first attempt within the same case) → `next-btn` calls `loadPatient()` again. `health` drops 25 per first-attempt miss; reaching 0 triggers `triggerGameOver()`, which replaces `#app`'s contents with a game-over screen (`location.reload()` is the only way to restart).

Keyboard hotkeys (A-F, matched positionally to `currentChoices`) are wired via a single `keydown` listener guarded by whether the question zone is currently visible.

### Adding a new rhythm
To add a rhythm: add an entry to `RHYTHMS` with a unique key, then implement `draw(t)` to approximate its ECG morphology (existing entries are the best reference for the modulo-cycle pattern). No other wiring is needed — the queue, difficulty pools, and distractor-choice logic all read from `RHYTHMS` and `rhythmKeys` (`Object.keys(RHYTHMS)`) automatically. Set `difficulty` and `pool` membership deliberately, since that determines which difficulty tiers can draw the new rhythm as a case or distractor.
