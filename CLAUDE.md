# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

「わり算ラボ」— a single-file, browser-based division learning game for Japanese
elementary kids. The entire app is `index.html`: HTML + CSS + vanilla JS in one
IIFE. No build step, no dependencies, no package manager, no tests.

## Running / Developing

- Run: open `index.html` directly in a browser (e.g. `open index.html`). There is
  no dev server, build, lint, or test command.
- All UI strings are Japanese and written in a kid-friendly tone (mixed
  hiragana/kanji, emoji). Match this tone when adding any user-facing text.
- Fonts load from the Google Fonts CDN, so a network connection is needed for the
  intended look; logic works offline.

## Architecture

Everything lives inside the IIFE at the bottom of `index.html`. Three things
matter for being productive:

**1. Two-level state machine.**
- Top level: three `<section>`s — `home`, `game`, `result` — toggled by `show(id)`.
- Inside `game`, each question runs three phases tracked by the `phase` variable:
  `'beaker'` (① split a quantity visually) → `'calc'` (② long division / 筆算) →
  `'check'` (③ verify by multiplication, たしかめ算). The phase transitions are
  `checkBeaker() → goCalc() → goCheck() → finishQuestion()`.

**2. Modes.** `mode` (0/1/2) selects the math type:
`小数÷整数` / `整数÷小数` / `小数÷小数`. Almost every render function
(`gen`, `buildBeakers`, `refreshBeakers`, `checkBeaker`, `goCalc`) branches on
`current.mode`. `current` is the active question object produced by `gen()` and
carries everything the three phases need (`a`, `b`, `ans`, `rem`, `hasRem`, story
strings, hints, etc.). Adding a math type = adding a new `mode` branch in those
functions, not a new code path.

**3. The "work in tenths" trick.** To avoid float errors with decimals, decimal
problems are handled as integers scaled by 10 (`a10`, `Q10`, `rem10`). `longDivision(a10, b)`
does standard place-by-place long division on integers and returns `{qd, qfull,
finalRem, steps}`; `renderLong()` then displays it digit-by-digit and the learner
picks each quotient digit. Mode 1/2 instead use the `tenfold` "multiply both sides
by 10" shortcut (`buildTenfold`).

## Supporting systems

- **Difficulty ramp:** `buildPlan()` precomputes a per-question plan (which divisor,
  whether there's a remainder) for a 10-question run; `loadQuestion()` consumes it.
  Remainder/no-remainder candidate problems are pre-generated into `MODE0`/`MODE0R`.
- **Timer + explosion:** `startQTimer()` runs a per-phase countdown; on timeout
  `explode()` plays a Web Audio boom (`playBoom()`) and restarts the *same* question
  from the beaker phase (`reloadSameQuestion()`).
- **Persistence:** best times per mode in `localStorage` via `getBest`/`setBest`
  (keys `warizanlab2_<mode>`).
- `missedThisQ` tracks whether the learner erred anywhere in a question; combo only
  counts if the whole question was clean.

## Conventions

- Keep it a single self-contained file. Reuse existing CSS classes (`.ld`,
  `.blankcell`, `.stepline`, `.choice`, beaker classes) rather than adding new
  systems. Follow the existing terse, branch-on-`mode` style.

## Planned work

`docs/superpowers/specs/` holds design specs for in-progress features (e.g. a
`2ケタ÷1ケタ` integer-division mode = `mode` 3). Read the relevant spec before
implementing a feature it covers.
