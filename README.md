# Skywatch

A self-observation tool for noticing your own patterns over time. Two small web
apps that run entirely in the browser and keep everything on your own device.

**[Open Skywatch →](https://andrewmorrison42.github.io/skywatch/)**

The idea is not to collect data. It's to have something worth reading *while*
you're struggling — evidence from your own history that storms like this one
have passed before, and a plan you wrote when calm so you don't have to
improvise one at 9/10.

## What's here

| | |
|---|---|
| **Skywatch** | Logging individual storms as they happen |
| **[Climate](https://andrewmorrison42.github.io/skywatch/climate.html)** | Longer-horizon belief and pattern work |

Write a report while something is happening — the situation, the thoughts, how
bad it is out of 10. There's an **It's bad right now** path that asks for almost
nothing when a full report is too much to face.

Hours later, Skywatch asks where it got to. Those two numbers make a pair, and
enough pairs turn into a line you can read next time: *4 of the last 5 storms
you logged at 7 or higher came down. Middle time: 6 hours.* It stays silent
until there's enough behind it to mean something.

A **storm plan** written while things are calm — early signs, what helps, what
makes it worse, what isn't yours to decide right now — feeds one-tap suggestions
back into the form when you're in it.

## Your data stays on your device

Everything you write lives in your browser's local storage. Your reports are
never uploaded — there's no account, no analytics, no server, and no way for
anyone else to read them. The code is public; nothing you log ever is.

The app makes no outbound requests at all — no CDN, no third-party fonts, no
telemetry. Opening it doesn't tell anyone you opened it.

Two practical consequences:

- **Back it up.** Export to CSV or JSON from the home screen. Clearing your
  browser data deletes your reports and there is no copy anywhere else.
- **Storage is per-site.** A copy opened from a local file and the version on
  the web address above keep completely separate reports. Moving between them
  means exporting and re-importing.

On iPhone, Share → **Add to Home Screen** makes it behave like an app and keeps
it up to date automatically.

## What this isn't

Not a medical device, not a clinical instrument, not a crisis service. It
doesn't diagnose anything, doesn't tell you what to do, and doesn't predict what
will happen. It shows you your own words and numbers back.

If you're in crisis, please contact your local emergency or crisis line — this
app is not a substitute for one and was never built to be.

## Running or changing it

No build step, no dependencies, no framework. Each app is a single HTML file
with its CSS and JavaScript inline — open it in a browser and it works, offline
and from disk. The only other assets are the icons and the two self-hosted
typefaces in `fonts/`.

`CLAUDE.md` documents the architecture, the storage keys, the data shapes, and
the design guardrails the project is held to.
