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
  `away`, `detail`, `plan`, `weathered`, `reference`
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
  followUp?: { ts, intensity, helped[], note },
  reset?: { kind, pre, post, after, mind,   // kind: "away" | "desk" — see The reset path
            ms, walked, override, reason } }
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
| `AWAY_MIN_MS` | 90s | Below this, a reset is recorded but doesn't vote in the stat |
| `MIN_AWAY` | 5 | Before/after taken five minutes apart needs more pairs than a storm pair |

**These floors are a safety decision, not tuning.** They exist so the app stays
silent rather than making a confident claim off two data points. Do not lower
them to make the stat appear sooner — an early, wrong reassurance is worse than
an empty panel, because it teaches the user not to trust the number later.

## The reset path

The breaths do not end by forwarding into the writing form, and they do not begin
cold either. Both flows now open on a reading, run the breaths, and ask again — three
moments on one scale, all three folded into the `breathe` screen as `#breathPre`,
`#breathBody` and `#breathRate`. The first is one tap with a skip; nothing is demanded
before any relief.

**The gate** (`needsWalk()`) routes on the second reading, two ways in:

- still at or above `plan.resetAt`, or
- it came in at or above `plan.resetAt` and the breathing moved it less than
  `plan.minDrop`.

The second clause is the one that matters. A threshold alone waves through the case the
whole path exists for — a storm that came in high and did not move is going through the
motions, whatever number it stopped at. The clause is deliberately limited to storms that
came in high, so a flat 2 is never sent out for a walk.

`away` is four states in one screen (`#awayLead`, `#awayDark`, `#awayWhy`, `#awayRate`),
toggled by `hidden`, never navigated between. The lead-in is the one place in either app
where `plan.works` is rendered as something to *read* rather than tapped as a chip. The
dark state has nothing to consume: no countdown, no progress bar, one line, and an
"I'm back" that only appears if you tap.

### Things that will bite

**Elapsed is always `Date.now()` minus `awayStart`, never a count of ticks.** A locked or
backgrounded phone throttles timers to a stop; counting ticks would let a five-minute
walk be reported off ninety seconds of foreground time. `visibilitychange` recomputes on
resume.

**The end signal is best-effort.** `navigator.wakeLock` keeps the page alive where the
platform allows it; the chime is generated with Web Audio from an `AudioContext` built on
the Start tap (iOS grants audio no other way) and `navigator.vibrate` where it exists.
None of it is load-bearing — the state change driven by the timestamp is. **Never fetch
an audio file for this**; it would be the app's first outbound request.

**`intensity` is the storm's height, not the eased number.** `stormLevel()` returns the
earliest reading (`pre`, falling back to `post`), and that is what the form pre-fills and
what gets saved. Writing the post-walk number there would quietly walk every successful
reset back into `stormStats()` as a smaller storm.

**A prefilled number is not something the user typed.** `quickSeed`/`entrySeed` hold it
and the discard prompts compare against those — otherwise backing out of an untouched
form asks "Discard this?" of someone who wrote nothing.

**The walk's authority is the user's own rule, not the app's judgement.** The lead-in
states the rule as it was written in fair weather — *"Your rule, written calm: at 6 or
above, you walk."* — and deliberately does not name the current number or argue for going.
The barrier this addresses is stated plainly by the user: at high distress it is hard to
*justify* stepping away, and a distressed brain cannot generate the justification. So the
app supplies the one they already wrote, rather than asking them to produce a new one.
Never replace this with a case for walking; the moment it argues, it is persuading rather
than reflecting.

**`plan.walkLine` is the load-bearing element of that screen, not decoration.** It sits
directly under the heading, above the route and the rule, because the blocker is a
standard applied to oneself that would never be applied to anyone else — the default line
(*"You'd send a team member for this walk. Same rule."*) names that asymmetry rather than
arguing. It is the same move as the `friend` field on the long form and `pickEcho()` in
the afterglow: hand back the user's own words, said about someone else. Keep the ordering;
demoting this line below the logistics guts the screen.

**The app does not comment on the user's self-judgement.** Naming an asymmetry the user
has stated is reflecting; telling them they are hard on themselves is a clinical claim and
belongs to their psychologist, not to a web page. No copy anywhere should praise, reassure,
or diagnose.

**`mind` is not the number.** The post-walk check-in asks the 0–10 scale *and*
`MIND` — how the head itself is. They come apart: back at 6 and thinking straight is a
different afternoon from 4 and still churning. Both are optional; `"Can't tell"` is a real
answer and is excluded from the stat rather than counted as a failure.

**Coming back early costs a tap, never a justification.** `#awayWhy` offers four chips and
an optional line, with a skip. It records `override` and `reason` honestly. There is no
lock on "I'm back", and no copy anywhere compares the elapsed time to what was asked for —
that friction is the line between recording a pattern and shaming someone at 9/10.

**The steer field is withheld by state, not by verdict.** `applySteerGate()` hides
"One thing for you" whenever the latest reading is at or above `plan.resetAt`, and renders
`worksBlock()` in its place. Nothing is said about why. Do not add an explanation — a
sentence about why the field is missing turns a design decision into a comment on how the
user is doing.

**`saveResetOnly()` exists because a walk with no report would otherwise vanish.** It
writes the existing `partial:true` shape, so the session still renders on home, opens in
detail, and can collect a follow-up later.

### What the stat may and may not say

`resetStat()` reports counts and the two moves separately, each behind `MIN_AWAY`. It
must never:

- divide one arm by the other, or call the difference an effect — the walk only ever
  happens on the storms that were already worse, so the arms are not comparable, and the
  copy says so out loud;
- report a rate of walks taken versus skipped. That is a compliance score, and a low one
  would land as a verdict on a bad week.

It reports the number and the head separately, never combined into a single score.

### Settings

`plan.route`, `plan.walkLine`, `plan.resetAt`, `plan.minDrop` and `plan.walkMins` live on
the `plan` screen and go through `normalizePlan()`, which fills defaults and clamps the
numbers via `num()`. These are personal tuning. The **stat** floors — `MIN_PAIRS`,
`MIN_TIME`, `MIN_HELPED`, `MIN_AWAY`, `AWAY_MIN_MS` — are not, and must stay hard-coded.

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
