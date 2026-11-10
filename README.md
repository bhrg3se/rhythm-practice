# Rhythm

A random rhythm generator and metronome for guitar/bass practice. Each phrase is
4 bars: bars 1–3 repeat the same rhythm, bar 4 is a variation. Notes are drawn as
real notation (whole → 16th, triplets, dotted values, rests) with a sweeping
playhead, an accented click, and an optional pluck on every note.

Plain HTML/CSS/JS. No dependencies, no build step. The whole app lives in
`index.html`; the other files only add home-screen install and offline support.

```
index.html             the app (markup, styles, script)
manifest.webmanifest   name, colours and icons for "Add to Home Screen"
sw.js                  service worker: network-first, offline fallback
icons/                 icon.svg (source) + rendered PNGs
```

## Run it

**Quickest:** open `index.html` in a browser. Everything works except offline
caching, because service workers don't run on `file://`.

**Served locally** (needed for the service worker / install prompt):

```sh
python -m http.server 8000      # then open http://localhost:8000
```

**On your phone, from the laptop:** service workers need HTTPS or `localhost`, so
plain `http://<laptop-ip>:8000` works but won't install offline. To test the
full experience over USB:

```sh
adb reverse tcp:8000 tcp:8000   # phone's localhost:8000 → laptop
```

then open `http://localhost:8000` in Chrome on the phone.

**Install:** on Android Chrome, menu → *Add to Home Screen* / *Install app*. It
opens in its own window and works offline after the first visit. Same on iOS
Safari via Share → *Add to Home Screen*.

Keyboard: `Space` play/stop · `N` new rhythm · `T` tap tempo · `0`–`5` level ·
arrows on the tempo slider.

## Deploy

It's a static site, so any static host works. Both of these are free and give you
HTTPS and a custom domain:

- **GitHub Pages:** push to GitHub → Settings → Pages → deploy from `main`, root.
- **Cloudflare Pages:** connect the repo, no build command, output directory `/`.

When you change cached files, bump `CACHE` in `sw.js` (`rhythm-v1` → `rhythm-v2`)
so old caches get cleaned up. The worker is network-first, so online visitors
see new deploys right away either way.

### Support link

Set `SUPPORT_URL` at the top of the script in `index.html` to your Buy Me a
Coffee / Ko-fi page and a small link appears under the controls. It's hidden
while empty.

If you later add ads: the service worker ignores cross-origin requests, so ad
scripts are never cached. Keep the slot away from Play/Tap/New Rhythm (accidental
taps violate AdSense policy) and reserve its height to avoid layout shift. AdSense
also needs `ads.txt`, a privacy policy and a consent banner for EU/UK visitors.

## Architecture

### Time model

Everything is measured in **quarter notes**. Each time signature in `SIGS` defines:

| field    | meaning                              | 4/4 | 3/4 | 6/8 |
|----------|--------------------------------------|-----|-----|-----|
| `barQ`   | bar length                           | 4   | 3   | 3   |
| `pulseQ` | metronome click spacing              | 1   | 1   | ½   |
| `beatQ`  | what the BPM number counts           | ♩   | ♩   | ♩.  |
| `groupQ` | beat group for cells and beaming     | 1   | 1   | 1½  |
| `acc`    | click accent per pulse (2/1/0)       | strong 1 | strong 1 | strong 1, medium 4 |

So 6/8 at 90 BPM means 90 dotted quarters per minute, clicking every eighth.

### Generator

A note is `{t, d, rest?, tup?}` (start, duration, in quarters). Rhythms are built
from weighted **cells**, small patterns that fill one beat group (a quarter in
simple time, a dotted quarter in 6/8) or a longer value. Cells live in two tables,
`SIMPLE_CELLS` and `COMPOUND_CELLS`, written in note values (`"8. 16"`, `"t tr t"`,
`"4.r"`; `r` = rest, `t` = triplet eighth). Each row has the level that introduces it:

| level | name        | adds (simple time)                                   | adds (6/8)                         |
|-------|-------------|------------------------------------------------------|------------------------------------|
| 0     | Quarters    | quarter, half, whole / dotted half                   | dotted quarter, dotted half        |
| 1     | Rests       | quarter and half rests                               | quarter + eighth, dotted-quarter rest |
| 2     | Eighths     | eighth pairs                                         | three eighths                      |
| 3     | Offbeats    | offbeat eighths, dotted quarter + eighth             | eighth + quarter, eighth rests     |
| 4     | Sixteenths  | 16ths, dotted 8th + 16th, 8–16–16, 16–16–8, triplets | 16th groups, dotted 8th + 16th     |
| 5     | Syncopation | 16–8–16, 16 + dotted 8th, 16th rests, 8–4–8, triplets with rests | more 16th mixes, rests inside beats |

Each level includes everything below it (half rests are retired above level 3).
To tune difficulty, edit the tables: add a row, or change a weight.
`makeBar()` picks weighted cells until the bar is full and rejects all-rest bars.
`regen()` makes bar A, then a different bar B, giving `[A, A, A, B]`.

Because cells never cross a beat group, the notation stays readable: no ties,
and beams always land on beat boundaries.

### Notation

Each bar is one inline SVG in pixel units (`barSvg()`), re-rendered when the width
changes. A note head sits just right of its onset, so the playhead reaches it as
it sounds. `durInfo()` maps a duration to a base value plus dot. Flagged notes
inside one beat group with no rest between them are beamed: one primary beam, a
secondary beam between adjacent 16ths, and a stub for a lone 16th. A beamed
triplet gets a "3" over the beam; a triplet broken by a rest gets a bracket. Colours come from `currentColor`, so the variation bar turns coral via CSS.

### Audio scheduler

This is the standard Web Audio look-ahead pattern. A 25 ms `setInterval` schedules
sounds up to 150 ms ahead on the sample-accurate `AudioContext` clock, so timer
jitter doesn't affect timing. It schedules **one pulse at a time**:

- the click for that pulse, plus a pluck for every note starting inside it
- a **segment** `{row, q0, q1, t0, t1}` that maps that slice of the bar to audio
  time, which the animation frame interpolates to move the playhead

Tempo changes therefore take effect at the next click, for sound and playhead
alike. "New Rhythm" while playing is queued and swapped in at the next bar line,
restarting the phrase at bar 1; the redraw waits until that bar actually starts.
All sound goes through a per-session gain node that is disconnected on stop, so
already-scheduled clicks are silenced. If the tab was throttled, the scheduler
skips ahead instead of firing a backlog. A Screen Wake Lock keeps the screen on
while playing, because Android throttles timers once the screen sleeps.

Settings (tempo, meter, level, toggles) persist in `localStorage`.
