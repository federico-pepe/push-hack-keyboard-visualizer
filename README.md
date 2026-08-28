# Keyboard Visualizer

Renders a live piano-keyboard visualization on Push 3's own screen, driven by
the notes **actually sounding in Live** — after any octave-shift or Scale/
In-Key transform — rather than the pad grid's raw pre-transform MIDI.

A hack for [`push-hack`](https://github.com/federico-pepe/ableton-push-hack)
(Push 3's on-device hack framework) — install it via that repo's `push-store`
hack, or build and deploy it yourself, see "Build & deploy" below.

## Requirements

**Needs the `push-hack` framework's `push-manager` and `push-display` hacks
also installed and running** (from
[federico-pepe/ableton-push-hack](https://github.com/federico-pepe/ableton-push-hack)).
This
hack never touches Push's shared-memory framebuffer itself — it's an HTTP
client of push-manager's `/api/display/*` API, which in turn needs
push-display's LD_PRELOAD hook attached to actually show anything on Push's
screen. Installing keyboard-visualizer alone (without push-manager +
push-display) means the Shift+Note chord toggles state internally but
nothing appears on screen.

At startup and every 30s, this hack checks `push-manager`'s
`/api/display/status` and logs one of:
- `WARNING: push-manager not reachable...` — push-manager isn't installed/running.
- `WARNING: ... push-display's shared-memory framebuffer is not connected...` —
  push-manager is up but push-display's hook hasn't attached (not installed,
  or Push3 hasn't been restarted since installing it).
- `push-manager + push-display OK — display takeover available.`

Check `/data/push-hack/logs/keyboard-visualizer.log` if takeover doesn't do
anything visible.

## Why not the raw pad stream

Push's pad grid always sends the same fixed chromatic notes (36-99). Push's
Octave +/- buttons and Live's Scale/Note mode change what actually plays
without changing what the pads themselves transmit — a visualizer driven by
that raw stream would show the wrong notes whenever either is active.

## How it works

1. This hack creates its own writable ALSA sequencer port, **"Keyboard Viz
   In"** (mirrors push-manager's own "Push Manager In" port pattern).
2. On Push's own screen, route a Live track's MIDI **Out** to this port
   (stock Live MIDI routing — no script install, no Max for Live device). One
   time per project/track, similar in spirit to Browser Bridge's one-time
   activation, but pure stock Live UI.
3. Whatever Live actually sends out that route — post-transform — arrives on
   the port. This hack tracks held notes (`map`-like `[128]bool`, set on Note
   On, cleared on Note Off; velocity-0 Note On is treated as Note Off) and
   renders a keyboard image.
4. The image is pushed to push-manager's existing display API
   (`POST /api/display/image`) whenever held notes change — but only while
   display takeover is switched on (see below). This hack never touches the
   shared-memory framebuffer directly.

Display takeover is **off by default** and toggled by holding **Shift + Note**
(CC49 + CC50) on Push's hardware — the same chord-detection pattern
push-manager already uses for hardware chords. Toggling on calls
`POST /api/display/mode {"mode":2}` and immediately draws the current
keyboard state; toggling off calls `{"mode":0}` to hand the screen back to
the native Push UI. To watch for this chord, "Keyboard Viz In" self-subscribes
to Push 3's own hardware MIDI port (`"Ableton Push 3 Live Port"`) in addition
to whatever Live routes to it — an ALSA seq port can receive from multiple
senders at once, so this reuses the same port rather than creating a second
visible ALSA client just to watch two CCs. Because both Push 3's raw pad
stream and Live's routed notes now land on the same port, incoming Note On/
Off events are filtered by their ALSA source client: only notes whose source
is *not* Push 3 update the visualizer's held-note state, so a pad press
doesn't also show up as its own (pre-transform) key on top of Live's
post-transform one. Only CC49/50 from Push 3 are read; MIDI intercept is
untouched.

Runs fully independent of MIDI intercept: the pad grid keeps playing into
Live normally throughout, whether or not the keyboard is currently shown.

## Web view (phone/tablet)

Open `http://push.local:7702` on any device on the same network for a
mobile-friendly, phone-in-portrait keyboard view — independent of on-device
display takeover, so it works whether or not the Shift+Note chord is active
on Push's own screen. It shows:

- The same live keyboard, driven by an SSE feed (`GET /api/notes/stream`) of
  currently-held notes.
- **Chord detection** — once 3+ notes are held, the detected chord name is
  shown large at the top (e.g. "Cmaj7"), via [Tonal.js](https://github.com/tonaljs/tonal)'s
  `Chord.detect()`. Vendored as a static prebuilt browser bundle
  (`src/ui/vendor/tonal.min.js`, MIT license, `src/ui/vendor/tonal.LICENSE`)
  so the page works fully offline — no CDN dependency, matching this
  project's "no runtime deps" ethos for use at gigs without internet.

## v1 scope / open questions

- Fixed 49-key window (4 octaves, notes 36-84, centered on middle C)
  rendered across the screen width (both on-device and in the web view). A
  scrolling/auto-ranging window sized to recent activity would adapt better
  but adds complexity — revisit if 49 keys proves too narrow in practice.
- All senders routed to "Keyboard Viz In" are merged into one visualization
  (no per-channel/per-track distinction). Fine for the common case of routing
  one track at a time.
- Chord detection shows only the first name Tonal.js returns when multiple
  interpretations are possible (e.g. a chord that's ambiguous between two
  names) — good enough for a quick glance, not exhaustive.

## Build & deploy

**Via Push Store (recommended):** install `push-hack`'s `push-manager` +
`push-display` + `push-store` hacks first, then install this one from
`push-store`'s web UI or `push-store install keyboard-visualizer` on-device —
it's listed in [ableton-push-hack's catalogue](https://github.com/federico-pepe/ableton-push-hack/blob/main/catalogue/catalog.json).

**Manually**, from a clone of this repo:
```bash
PATH=$PATH:/usr/local/go/bin make            # build for Push 3 (linux/amd64)
```
This produces `build/keyboard-visualizer` (and copies it to the repo root).
Deploy it to `/data/push-hack/hacks/keyboard-visualizer/` alongside this
repo's `hack.json`, and register/start it the same way the framework's own
`install.sh` does (create `/etc/init.d/push-hack-keyboard-visualizer`,
`start-stop-daemon` it).

**Releasing a new version:** bump `hack.json`'s `version`, then
`git tag vX.Y.Z && git push origin vX.Y.Z` — `.github/workflows/release.yml`
builds, publishes the release tarball, and updates `release.json` so
push-store's next install/update picks it up automatically, no action needed
in the main repo's catalogue.

## API

- `GET /api/status` (port 7702) — `{"status":"ok","name":...,"version":...,"port":7702,"notes_held":<n>}`.
- `GET /api/notes/stream` — SSE feed of currently held MIDI notes, e.g.
  `data: {"notes":[60,64,67]}\n\n` on every change (sent once immediately on
  connect, then on every change).
- `GET /` — the mobile web view (embedded, single file).
