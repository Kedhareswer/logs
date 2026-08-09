# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file work-shift tracker. `index.html` is the entire app — CSS, markup, and JS in one document. No build, no dependencies, no package.json. The only other files are `README.md` and the Pages workflow.

## Commands

- **Run locally**: open `index.html` directly, or `python -m http.server 8000`.
- **Self-check**: load `index.html?selftest` and read the console. `selfTest()` covers segment placement, cycle maths, pending-day detection, pay estimates, and CSV parsing. Keep it passing; extend it when you touch those functions.
- **Deploy**: push to `main`. `.github/workflows/static.yml` uploads the whole repo to GitHub Pages.

Nothing to build or lint.

## Layout

Two-column workspace. `.main` (left) holds the stat bar, the biweekly calendar, and the timeline; `.rail` (right, `position: sticky`) holds the entry form, the cycle checklist, and the CSV actions. Below 1240px the rail unsticks and stacks underneath, so the calendar still comes first. The calendar is meant to be fully visible without scrolling — if you add anything above it, check that.

## Architecture

**Data model.** A shift is `{id, date:'YYYY-MM-DD', segments:[{start,end}], note}`, in `localStorage` under `DATA_KEY`. `withLegacyFields()` mirrors the first segment into top-level `start`/`end` for older readers — never read those directly, go through `getSegments()`. Two sibling keys hold non-log state: `PREFS_KEY` (`{rate, currency}`) and `CYCLES_KEY` (`{ '<cycleStartDate>': {paid, received} }`, one record per pay cycle).

`loadShifts()` tries `DATA_KEY`, then `LEGACY_KEY`, then `seedShifts()`. `SEED_ROWS` is the user's real history imported from `work_log_2026-08-09.csv` — a data baseline, not demo data. Don't regenerate or trim it.

**Segment placement is the core of the render.** `placeSegments()` maps a day's segments onto a continuous hour line: sorts them, adds 24h to any segment that would land before the previous one ended, and lifts a day whose segments all start before `NIGHT_TAIL_BEFORE` (06:00) into the next-day lane so a lone `00:00–02:30` reads as the previous evening's tail. Everything positional — Gantt bars, the shared axis, calendar segment lines, unpaid-gap maths, the chronological order used at save time — comes from this one function. A day holds any number of segments.

Do not reintroduce an hour cutoff applied *per segment* rather than per day; that is what broke mixed days like `09:00–10:00` + `12:00–15:00`.

**Hours vs gaps.** `computeShiftHours()` sums each segment's own duration, so gaps between segments are excluded by construction. The unpaid-gap readout is derived from placement and displayed only — never stored.

**Pay cycles.** 14 days from `PAY_ANCHOR` (2026-05-09). The stat bar always reports the cycle containing `TODAY`; the calendar, timeline, and checklist all follow `viewCycleStart`, which the cycle nav moves in ±14-day steps.

**Checklist** (`renderChecklist`) answers three things for the *viewed* cycle: whether every past day has an entry (`unloggedDays()` — past days only, future days are "still to come", and each gap is a chip that jumps the form to that date), whether the cycle is marked paid (persisted per cycle), and what the pay should come to (`estimatePay()` = hours × rate, compared against an optional recorded amount). Rate and currency are user prefs; there is no hardcoded currency.

**CSV.** Export writes a UTF-8 BOM and ASCII `-` between times — the old export used an en dash with no BOM, which Excel rendered as mojibake. `importCSV()` reads that same shape and merges by date; `parseSegmentSpec()` accepts `-`, `–`, and `—` so old exports still import.

**Rendering.** No framework. `render()` fans out to `renderStats` / `renderCalendar` / `renderTimeline` / `renderChecklist` / `renderSegments`, each rebuilding its container from template strings; the cycle nav calls only the three cycle-scoped ones. Any user text in a template goes through `escapeHTML()`.

Two re-render hazards, both already handled — keep them in mind when adding fields: typing in a segment time calls `updateSegDurations()` rather than `renderSegments()`, so the focused input is never replaced; and `setInputValue()` skips the currently focused field, so a half-typed `18.` doesn't get rounded back to `18` underneath the cursor.

**Design language.** Dark-only warm monochrome editorial: `#0F0F0E` canvas, hairline `--border` dividers, Newsreader serif for figures, Geist for UI, Geist Mono for metadata, one data accent (`--data`). Every colour is a `:root` token — add one rather than hardcoding a hex. Gantt bars are light-on-dark (`--bar` / `--bar-text`); today's bar flips to `--data` / `--on-accent`. No emoji; the checklist tick is an inline SVG. Sections fade in via `IntersectionObserver` (`.reveal`), disabled under `prefers-reduced-motion`.

**Lock is cosmetic.** Ctrl/Cmd+E toggles edit mode against the `EDIT_PASSWORD` constant, compared client-side on a public static page. `setLocked()` disables every input and button inside `.panel`, so anything editable you add to a panel is gated automatically. It prevents accidental edits, not access — never put anything sensitive behind it.

`TODAY` is snapshotted at load and never refreshed, so a long-lived tab keeps yesterday's "today".
