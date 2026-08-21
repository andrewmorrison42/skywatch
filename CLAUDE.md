# Skywatch

A self-observation tool for mental-health growth. Two standalone web apps, served
from GitHub Pages, storing every log on the user's own device.

The purpose is not to collect data. It is to give someone something useful to
read **while** they are struggling — evidence drawn from their own history that
storms like this one have passed before, and a plan they wrote when calm that
they no longer have to recall from memory.

**This is not a medical device, a clinical instrument, or a crisis service.** It
does not diagnose, treat, or advise. It reflects the user's own words and numbers
back to them.

## The two apps

Both are single self-contained HTML files. No build step, no dependencies, no
framework, no backend, no network calls.

| File | Name | Purpose |
|---|---|---|
| `index.html` | Skywatch | Logging individual storms as they happen |
| `climate.html` | Skywatch Climate | Longer-horizon belief and pattern work |

`index.html` links to Climate with a plain `<a href="climate.html">`. There is no
shared code between them — see the duplication warning below.

**Screens** (each app is a set of `.screen` divs switched by a `show(id)` router):

- `index.html` — `home`, `entry`, `quick`, `followup`, `afterglow`, `breathe`,
  `detail`, `plan`, `weathered`, `reference`
- `climate.html` — `home`, `entry`, `checkin`, `close`, `detail`, `guide`,
  `ledger`, `mantra`, `outcome`

## Storage

All data is local. Nothing is transmitted anywhere.

| Key | Written by | Read by |
|---|---|---|
| `skywatch-entries` | `index.html` | `index.html` |
| `skywatch-plan` | `index.html` | `index.html` |
| `skywatch-climate` | `climate.html` | `climate.html` |
| `skywatch-beliefs` | `climate.html` | `climate.html` |
| `skywatch-mantra` | `climate.html` | **both** |

### Three things that cause silent bugs here

**1. `skywatch-mantra` is the only cross-app key.** `climate.html` writes it;
`index.html:620` reads it and must never write it. Changing its shape in one app
breaks the other with no error.

**2. Each app has its own near-duplicate `store` object.** They are separate
implementations that happen to look alike. Fixing a bug in one does *not* fix it
in the other — check both. Each has three modes, selected at `init()`:

- `claude` — `window.storage` when running inside a Claude Artifact
- `local` — `localStorage`
- `memory` — in-page fallback when storage throws (private browsing, etc.)

**3. `localStorage` is per-origin.** A copy opened from `file://` and the same
app on the Pages URL keep entirely separate data. Users moving between them must
export and re-import; their entries do not follow them.

### Data shapes

`index.html` entry — created in `saveEntry()`:

```js
{ id, ts, situation, thoughts, feeling, patterns[], facts, control,
  friend, notMine, intensity,
  partial?, updated?,                       // partial: saved via the quick path
  followUp?: { ts, intensity, helped[], note } }
```

`index.html` plan — `PLAN_LISTS` is `["signs","works","worse","noDecide","tell"]`,
plus `opener` and `line` strings. Always read through `normalizePlan()`
(`index.html:705`), which coerces anything malformed into a valid shape. Reuse
that defensive pattern rather than trusting stored JSON.

`climate.html` entry — `{ id, ts, trigger, schemas[], perception, body, pulls[],
followed, payoffs[], costs[], action, prediction, conf, notice, human, offer,
need, outcome: null | { ts, what, match, learned } }`. Check-ins under
`skywatch-beliefs` are `{ ts, scores }`.

Entries predating a feature simply lack those fields. **Every render path must
tolerate missing fields** — old entries have no `intensity` and no `followUp`,
and must still display correctly in the home list, detail view, and exports.

## Serving it

GitHub Pages is enabled and deploys from `main` on push:

```
https://andrewmorrison42.github.io/skywatch/
```

These do **not** work and should never be given to a user as a link:

- `github.com/.../blob/main/index.html` — source viewer, shows code
- `raw.githubusercontent.com/...` — serves HTML as `text/plain`

Because Pages is HTTPS, the app is a secure context: installable to the iOS Home
Screen, and eligible for a service worker if offline caching is ever added.

## The evidence engine

The `weathered` view pairs each entry's `intensity` with its `followUp.intensity`
to produce a line the user can read mid-crisis — *your last 6 storms at 7+ all
passed, median 4 hours.* The stat narrows to the band the user is currently in,
so an 8 is compared against other 7+ days rather than a diluted lifetime average.

Thresholds at `index.html:857-862`:

| Constant | Value | Meaning |
|---|---|---|
| `FU_AFTER` | 4h | Don't ask for a follow-up while they're probably still in it |
| `FU_WINDOW` | 72h | Stop asking after three days |
| `HIGH` | 7 | The "bad day" band |
| `MIN_PAIRS` | 3 | Below this, say nothing at all |
| `MIN_TIME` | 4 | A median duration needs this many pairs |
| `MIN_HELPED` | 3 | A coping chip must recur this often before it's named |

**These floors are a safety decision, not tuning.** They exist so the app stays
silent rather than making a confident claim off two data points. Do not lower
them to make the stat appear sooner — an early, wrong reassurance is worse than
an empty panel, because it teaches the user not to trust the number later.

## The reminder path

The follow-up loop only works if the user comes back, so entries offer a
check-back reminder. It looks convoluted because the platform leaves no direct
route:

- A page cannot wake itself once closed — service workers need a secure context,
  which rules out `file://`.
- Notification Triggers (`TimestampTrigger`) never shipped past an experimental
  Chrome flag, so there is no client-side scheduled notification anywhere.
- iOS web push needs an HTTPS install *and* a server pushing at the right moment.

So scheduling is handed to the OS:

- **Apple Reminders** via `shortcuts://run-shortcut`, matched by the Shortcut
  name `Skywatch check-in` (`index.html:969`). There is no public URL scheme that
  creates a reminder directly. Renaming the Shortcut silently breaks the button.
- **Calendar `.ics`** underneath — no setup, works anywhere, shared via
  `navigator.share` where files can be shared and downloaded otherwise.

Apple devices get both; everything else gets the calendar route alone rather than
a dead button. `remindWhen()` clamps times forward so a days-old entry never
schedules a reminder in the past.

## Guardrails

The failure mode for this project is not a crash. It is a well-intentioned change
that makes the app shaming, over-claiming, or manipulative. Before adding
anything, check it against these.

**No engagement mechanics.** No streaks, badges, counters of consecutive days, or
anything that turns a gap into a failure. Someone who stops logging for two weeks
was probably having a hard time; the app must never greet them with evidence of
their own inconsistency. A gap is information, not a lapse.

**No clinical or diagnostic claims.** The app does not tell users what they have,
what they should do, or what will happen. It reflects their data back. Avoid
anything that reads as a prognosis.

**Echo the user's own words.** The afterglow uses `pickEcho()`
(`index.html:1128`) to quote what the user themselves wrote in a past entry,
rather than a generic affirmation. Their own sentence carries weight a platitude
cannot. Preserve this whenever adding supportive copy.

**Never fabricate crisis resources.** Do not invent helpline numbers, hours, or
service names. If crisis signposting is added, verify every detail against
official sources first — a wrong number at the wrong moment is a serious harm.

**Keep the statistics honest.** Report what the data supports, including when it
supports nothing. Never round a small sample into a confident statement, never
hide the sample size, and never present a selected subset as the whole picture.

**Data stays on the device.** No analytics, no telemetry, no error reporting, no
CDN, no third-party anything. The app makes no outbound requests, and that is a
promise to the user, not an implementation detail. Note that the repo is public,
so the *code* is visible — but no entry ever leaves the browser.

Fonts are **self-hosted in `fonts/`** and declared with `@font-face` in each
app's inline `<style>`. Both apps previously linked Google Fonts, which sent the
user's IP to a third party on every load; do not reintroduce a `<link>` to any
font CDN. When checking this guarantee, grep for `<link[^>]*href="http` as well
as `fetch(` and `<script src=` — a stylesheet link is easy to miss.

**Write for someone at 9/10.** Copy that appears mid-crisis should be short,
concrete, and undemanding. Never ask a distressed user to fill in a form, make a
decision, or read a paragraph. The `quick` path exists precisely for this.

## Conventions

- One self-contained file per app; HTML, CSS, and JS all inline.
- Vanilla JS. No framework, no bundler, no package manager.
- Screens are `.screen` divs; `show(id)` swaps them.
- All storage calls are `async` (the Artifact backend is promise-based) — `await`
  them even though `localStorage` is synchronous.
- Comments explain *why*, not *what*, and are used sparingly. Several constants
  carry a short trailing comment giving their rationale — keep that habit.

## Testing

There is no test framework in the repo. Verification is done by driving the real
page:

- **Playwright** against Chromium, launched from `PLAYWRIGHT_BROWSERS_PATH`
  (`/opt/pw-browsers`). Do not run `playwright install`. Cover the legacy-data
  case (entries with none of the newer fields), storage persistence across
  reload, and each screen's happy path.
- **Syntax check** by extracting the `<script>` block and running `node --check`.

Test with realistic data, including old entries missing recent fields — that is
where this codebase actually breaks.
