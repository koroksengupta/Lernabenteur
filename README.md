# Lernabenteuer 🦔

**Lernabenteuer** ("learning adventure") is a single-file, offline-friendly web app that helps a young German-speaking child (roughly Grade 1 level) practice **Deutsch** (German), **Mathematik** (Math), and **Sachkunde** (General Studies) through short, spaced-repetition quiz rounds — with a built-in parental dashboard to track progress.

Live app: deployed via GitHub Pages from the `main` branch (see [Deployment](#deployment)).

> This README documents the app as implemented in `index.html`. There is intentionally no build step, no framework, and no backend — the entire application is one HTML file.

---

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
- [How the app works](#how-the-app-works)
  - [Screens / navigation](#screens--navigation)
  - [Question content](#question-content)
  - [Spaced repetition engine](#spaced-repetition-engine)
  - [Daily limits](#daily-limits)
  - [Retention tracking](#retention-tracking)
  - [Parent gate & dashboard](#parent-gate--dashboard)
  - [Theming](#theming)
  - [Text-to-speech](#text-to-speech)
- [Data & persistence](#data--persistence)
- [Accessibility](#accessibility)
- [Mobile / responsive design](#mobile--responsive-design)
- [Deployment](#deployment)
- [Development notes](#development-notes)
- [Known limitations](#known-limitations)
- [License](#license)

---

## Features

- **Three subjects, eight topics**: letters & sounds, words & pictures, and rhymes for German; number recognition, more-or-less comparison, addition/subtraction, and shapes for Math; seasons, animals & habitats, traffic & safety, and the human body for General Studies.
- **Spaced repetition (Leitner system)**: every question the child has seen is tracked in a 5-box Leitner scheduler, so items the child struggles with come back sooner and mastered items come back later.
- **Daily practice caps** per subject, weighted to roughly match a Grade 1 weekly class-hour ratio, to keep daily screen time short and predictable.
- **Day-over-day retention checks**: ~20% of each day's round is deliberately drawn from yesterday's questions (not repeated immediately — a day later, once it would otherwise start fading) to measure genuine retention rather than same-session memorization.
- **Procedurally generated math**: number recognition, comparison, and addition/subtraction questions are generated on the fly (not from a fixed bank), so there's effectively unlimited practice material for those topics.
- **Parent gate + dashboard**: a simple math "captcha" keeps the settings/analytics screen out of a young child's reach; the dashboard shows mastery %, recent accuracy, retention history, a "needs more practice" list, and a full activity log.
- **Read-aloud**: every question can be read aloud in German via the browser's built-in speech synthesis.
- **Light / dark / auto theme toggle**, persisted per device.
- **Installable PWA**: manifest and icons are embedded inline, so the app can be added to a phone's home screen.
- **No backend required**: runs entirely client-side with `localStorage`, with an optional pluggable cloud-storage hook (see [Data & persistence](#data--persistence)).

## Tech stack

- **Plain HTML, CSS, and vanilla JavaScript (ES5-leaning syntax)** — no framework, no build tools, no npm dependencies.
- **Google Fonts**: [Fredoka](https://fonts.google.com/specimen/Fredoka) (UI) and [Andika](https://fonts.google.com/specimen/Andika) (letter/word display, designed for early literacy).
- **Web Speech API** (`speechSynthesis`) for German read-aloud.
- **`localStorage`** for progress persistence by default.
- Deployed as a **static site via GitHub Pages**.

Because it's a single static HTML file with no dependencies, it can be opened directly in a browser, hosted from literally any static file host, or embedded/iframed elsewhere with no server-side requirements.

## Project structure

```
.
├── index.html                    # The entire application (markup + CSS + JS)
└── .github/
    └── workflows/
        └── static.yml             # GitHub Actions workflow that deploys index.html to GitHub Pages on push to main
```

Everything — data, styling, rendering logic, state management — lives inside `index.html`:

| Section (inside `index.html`) | Responsibility |
|---|---|
| `<head>` | Meta tags, inline base64 PWA manifest & icons, Google Fonts, all CSS |
| `SUBJECTS` / `QUESTION_BANKS` | Static content: subjects, topics, and curated question banks |
| `PROCEDURAL_TOPICS` / `gen*`/`build*` functions | On-the-fly question generation for the three math topics |
| `STORAGE` section | Load/save progress to `localStorage` or an optional pluggable DB API |
| `SPACED REPETITION ENGINE` | Leitner-box scheduling, daily caps, carryover/retention bookkeeping |
| `SPEECH` | Text-to-speech helper |
| `STATE + RENDER` | A single `state` object plus `render()` / `renderView()` functions that re-render `#app`'s `innerHTML` on every state change |
| `ACTIONS` | Event-handler functions (navigate, start a round, answer a question, etc.) |
| `EVENT DELEGATION` | One click listener and one keydown listener on `#app`, dispatching on `data-action` attributes |
| `BOOT` | Applies the saved theme, then loads progress and does the first render |

The app follows a small, self-contained **"model → render → event → update" loop**: a single `state` object (and a `progress` object holding all persisted data) drives everything; every action mutates `state`/`progress` and calls `render()`, which fully re-renders `#app.innerHTML` from scratch. There is no virtual DOM and no component framework — just template-literal-style string building and one delegated event listener.

## Getting started

No build step, no install step. Any of the following works:

**Option 1 — just open the file**
```bash
open index.html        # macOS
# or double-click index.html in a file browser
```

**Option 2 — serve it locally** (recommended, since some browsers restrict `localStorage`/speech APIs on `file://` origins)
```bash
cd Lernabenteur
python3 -m http.server 8000
# then open http://localhost:8000
```
Any static file server works equally well (`npx serve`, `php -S localhost:8000`, etc.) — there is nothing to `npm install`.

## How the app works

### Screens / navigation

The app is a small state machine over `state.view`:

| View | Purpose |
|---|---|
| `loading` | Shown while progress is being loaded from storage |
| `map` | Home screen — pick a subject (Deutsch / Mathematik / Sachkunde) |
| `topics` | Pick a topic within the chosen subject |
| `play` | The quiz itself — one question at a time, multiple choice |
| `roundend` | Round summary (stars earned), with options to replay, pick another topic, or go home |
| `parentgate` | A simple addition "captcha" gating parent-only content |
| `parentdash` | The parent dashboard (mastery, retention, activity log, settings) |

Navigation is handled by small `render*()` functions that return HTML strings, and `render()`, which sets `document.getElementById('app').innerHTML` to the current view's output. All user interaction is captured by **one delegated click listener** on `#app` that dispatches based on each element's `data-action` attribute (e.g. `data-action="start-round"`), rather than attaching a listener per button.

### Question content

Each question is a plain object with a `prompt`, a `displayType` (`letter`, `emoji`, `shape`, `dots`, `compare`, `equation`, or `none`), the data needed to render that display, an array of `choices`, and the correct `answer`. Two sources feed questions:

1. **Curated banks** (`QUESTION_BANKS`) — hand-written questions for German and General Studies, plus the *shapes* topic in Math:

   | Subject | Topic | # curated questions |
   |---|---|---|
   | Deutsch | Buchstaben & Laute (letters & sounds) | 24 |
   | Deutsch | Wörter & Bilder (words & pictures) | 20 |
   | Deutsch | Reime (rhymes) | 16 |
   | Mathematik | Formen (shapes) | 14 |
   | Sachkunde | Jahreszeiten (seasons) | 14 |
   | Sachkunde | Tiere & Lebensräume (animals & habitats) | 16 |
   | Sachkunde | Verkehr & Sicherheit (traffic & safety) | 14 |
   | Sachkunde | Mein Körper (my body) | 16 |

2. **Procedural topics** (`PROCEDURAL_TOPICS`) — generated at request time, effectively unlimited:
   - **Zahlen erkennen** (number recognition): "how many dots?" for 0–20.
   - **Mehr oder weniger** (more or less): compares two random dot groups (1–12 each).
   - **Plus & Minus** (addition/subtraction): random equations, biased toward sums/differences ≤ 20, with a visual dot-grid aid shown for smaller numbers.

Every question has a stable `id` (e.g. `deutsch.buchstaben.5` for curated items, or `mathematik.rechnen.plus-3-4` for a generated one), which doubles as the key used by the spaced-repetition engine to track history for that specific item.

### Spaced repetition engine

Each answered question updates a per-question stats record (`progress.stats[id]`) implementing a classic **5-box Leitner system**:

- Box **1 → 5**, starting at box 1.
- A correct answer moves the item up one box (capped at 5); a wrong answer sends it straight back to box 1.
- Each box has a re-review interval (`BOX_INTERVAL_MS`): box 1–2 are due immediately, box 3 is due after 1 day, box 4 after 3 days, box 5 after 7 days, and there's a further 14-day interval defined for future use beyond box 5.
- A round (`ROUND_LEN = 8` questions) is built by mixing: a small slice of **carryover** questions from the previous day (for retention checking — see below), items that are currently **due** for review, and **new/unseen** items, so practice stays varied rather than just grinding the same due items.
- Getting a question wrong doesn't just re-file it into box 1 — it's also **re-inserted a few questions later in the same round's queue**, so the child gets an immediate second attempt at it before the round ends.

### Daily limits

Each subject has a **daily question cap** (`DAILY_CAPS`): Deutsch 9, Mathematik 7, Sachkunde 4 (20 total), deliberately weighted close to a **6 : 5 : 3** ratio — matching Baden-Württemberg's Grade 1 weekly class-hour split for these three subjects — and sized to keep total daily screen time short. Once a subject's cap is reached for the day, its topics show as locked (🔒) until the next calendar day, unless a parent enables "extra practice today" from the dashboard (a per-day bonus toggle that temporarily lifts the cap).

### Retention tracking

To distinguish "the child remembers this" from "the child just answered it a moment ago," the app tracks two different things on the parent dashboard — worth understanding as distinct metrics:

- **Recent accuracy** (per subject): whether the *most recent* attempt at each practiced item was correct. This is a snapshot of current performance.
- **Day-over-day retention**: specifically how the child does on the ~20%-carryover questions pulled in from the *previous* day's session — a cleaner signal of whether material actually stuck overnight, tracked with a rolling history (last 30 days).

### Parent gate & dashboard

Tapping the 👪 icon prompts a simple two-number addition problem before revealing the dashboard — enough friction to keep a young child from wandering into settings, without requiring an account or password. The dashboard shows:

- Today's overall progress and a toggle to lift the daily cap for the rest of the day.
- Day-over-day retention (today's number plus a 7-day history strip).
- Per-subject mastery % (share of practiced items that have reached box 4+), recent accuracy, and total items practiced.
- A "needs more practice" list of the most frequently missed items.
- A filterable activity log (last 60 entries shown, up to 200 retained) marking which answers were retention carryovers (🔁).
- A "reset progress" control (with a confirmation step) that wipes all stored progress.

### Theming

The app defaults to following the operating system's light/dark preference (`prefers-color-scheme`). A theme button in the top bar (🌓/☀️/🌙) cycles **auto → light → dark → auto**, overriding the OS setting and persisting the choice in `localStorage` per device/browser — useful since two devices with different OS theme settings would otherwise render the same page differently.

### Text-to-speech

The "🔊 Vorlesen" button on each question uses the browser's `SpeechSynthesis` API to read the question prompt aloud, preferring a German (`de-*`) voice when one is available on the device. This is purely client-side and requires no API key or network call; on very quiet unavailable-voice browsers it silently degrades (button does nothing) rather than erroring.

## Data & persistence

Progress is a single JSON object (`progress`) containing:

```
{
  meta: { createdAt, lastPlayedAt, streakDays, totalStars, totalRounds, totalAnswered, totalCorrect },
  stats: { "<questionId>": { box, correctStreak, totalSeen, totalCorrect, totalWrong, dueAt, history, ... } },
  log: [ { ts, id, subject, topic, correct, box, carryover }, ... ],   // capped at 200 entries
  daily: { date, counts: { deutsch, mathematik, sachkunde }, bonus, retentionCheck: { attempted, correct } },
  yesterdayShown: { date, ids },
  retentionHistory: [ { date, attempted, correct }, ... ]              // capped at 30 entries
}
```

By default this is persisted to the browser's **`localStorage`** under the key `lernabenteuer_progress_v1`. The app also supports an optional pluggable cloud-storage hook: if `window.claude.use('db')` resolves to a database API (a capability available when the page is hosted as a [Claude Artifact](https://www.anthropic.com/news/artifacts)), progress is written there instead, with automatic fallback to `localStorage` if that write ever fails. If a save to `localStorage` itself fails (e.g. storage quota exceeded, private browsing restrictions), the parent dashboard shows a visible warning rather than silently losing data.

**Nothing is sent to any server.** All data stays on the device (or, if the Claude Artifact DB hook is active, within that hosting platform's storage) — there is no analytics, tracking, or external API call anywhere in the app.

## Accessibility

- Answer feedback ("Super gemacht!" / "Fast!") is announced via `aria-live="polite"`.
- Icon-only controls (the parent-gate button, theme toggle) carry `aria-label`s.
- The parent-facing screens (mostly English copy, since the app itself is in German) are wrapped in `lang="en"` so screen readers pronounce them correctly against the document's `lang="de"`.
- The viewport allows pinch-zoom (no `user-scalable=no`/`maximum-scale` lock), so users who need to zoom in aren't blocked from doing so.
- `prefers-reduced-motion` disables the loading-screen bob animation and all CSS transitions.

## Mobile / responsive design

This app is designed mobile-first, since it's primarily used on phones/tablets handed to a child:

- All typography is defined in `rem` units against a **responsive root font-size** — 16px on desktop/tablet widths, scaling up via `clamp()` on phone-width viewports (≤480px) so text reads meaningfully larger on a phone rather than just technically larger.
- Layout uses flexible containers (flex/grid) with `flex-wrap` so controls reflow instead of overflowing on narrow screens.
- The single-column `#app` container caps at `max-width: 640px` and centers itself, so the same markup scales cleanly from a small phone up to a tablet/desktop window.

## Deployment

The site deploys automatically via **GitHub Pages**, driven by `.github/workflows/static.yml`:

- Trigger: every push to `main` (or manually via `workflow_dispatch`).
- The workflow uploads the entire repository as a Pages artifact and deploys it — since `index.html` is at the repo root, GitHub Pages serves it directly as the site's index page.
- No build step runs; the committed `index.html` is deployed byte-for-byte.

To deploy elsewhere, just copy `index.html` to any static host (Netlify, Vercel, S3, nginx, etc.) — there's nothing to build first.

## Development notes

- The entire app is one `(function(){ 'use strict'; ... })();` IIFE — all state and helpers are module-private; nothing is attached to `window` except the standard boot sequence.
- There is no test suite or linter configured. Before committing changes, at minimum:
  - Syntax-check the inline script (e.g. extract the contents of the `<script>` tag and run `node --check` on it).
  - Manually verify the affected screen(s) in a browser — this is a rendering-heavy app driven by string concatenation, so a typo in an HTML string won't be caught by JS syntax checking alone.
- Code style: ES5-leaning syntax (`var`, `function` expressions, no arrow functions/`let`/`const`/classes) is used consistently throughout — match this style rather than introducing modern syntax piecemeal.
- When adding a new field to the `daily` progress object, remember `mergeWithDefaults()`/`defaultDaily()` exist specifically to backfill missing fields for users with progress saved by an older version of the app — extend `defaultDaily()` rather than reading a new field directly off `progress.daily` elsewhere in the code.

## Known limitations

- **Single local profile**: there's no concept of multiple child profiles or user accounts — progress is tied to one browser/device's local storage.
- **No offline service worker**: the PWA manifest makes the app installable, but there's no service worker, so a fresh load still requires network access the first time (subsequent loads may be served from the browser's ordinary HTTP cache).
- **No automated tests**: correctness is currently maintained by manual verification and syntax checking rather than a test suite.
- **Text-to-speech quality depends on the device**: voice availability and quality for German varies by browser/OS, and there's no bundled audio fallback.

## License

No license file is currently included in this repository. If you intend to reuse or distribute this code, add a `LICENSE` file specifying the terms.
