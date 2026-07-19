# qBittorrent Monitor — iCUE Widget

A Corsair iCUE custom widget for the XENEON Edge that shows live
download/seed activity from qBittorrent and lets you pause or resume
individual torrents without leaving the panel.

This widget is display-only — all the actual polling and qBittorrent
API calls happen in a separate local relay process (the same relay
used by the TrueNAS and Minecraft widgets). See that project's README
for setup; this one covers the widget itself.

## What it shows

- **Status** — online/offline pill, with the qBittorrent icon
  recoloring to match
- **Downloading / Seeding counts**, each with its live transfer speed
- **Session totals** — total torrent count, and cumulative session
  download/upload data
- **Per-torrent rows** — name, direction/status icon, speed or state,
  percent complete, ETA (for active downloads), and a progress bar
- **Color-coded by status**:
  - Green — actively downloading
  - Purple — completed (100%), whether it's actively seeding or has
    been stopped — completed always wins over the "stopped" color
  - Amber/yellow — queued or stalled (still incomplete, still
    intends to run on its own)
  - Red — explicitly paused/stopped, incomplete
- **Tap-to-pause/resume** — every row with a known torrent hash gets a
  small circular button. Halted torrents show a resume (▶) button;
  everything else shows pause (⏸). This reflects the torrent's real
  running state, independent of its display color — a stopped torrent
  that happens to be 100% complete still shows purple text *and* a
  resume button.

## Layout

Built and sized for the **Small** XENEON Edge slot (840×344px): a
fixed-width info column on the left (status, counts, speeds) and a
scrollable torrent list on the right. Like the Minecraft widget, this
isn't currently responsive across slot sizes — Medium/Large/XL will
need layout adjustments.

## Files

```
QbitWidget/
├── manifest.json   # widget metadata, device targeting, interactive:true
├── index.html      # all UI + logic (CSS and JS inlined)
└── icon.svg        # preview icon shown in iCUE's widget picker
```

## How it gets its data

The widget polls `http://127.0.0.1:8787/qbit-stats` on the relay
(configurable via the widget's own settings panel, default port
8787). It never talks to qBittorrent directly — button taps `POST` to
the relay's `/qbit-action` endpoint (`{"hash": "...", "action":
"pause"}` or `"resume"`), and the relay handles login and the actual
qBittorrent Web API call.

If the relay isn't reachable, the widget shows an offline status with
the specific reason (can't reach relay vs. relay reports qBittorrent
itself is offline) rather than failing silently.

## Packaging

From inside this folder:
```
icuewidget validate QbitWidget
icuewidget package QbitWidget
```

## Limitations (by design, not oversight)

- **No delete/add-torrent controls** — pause/resume only. Deleting a
  torrent is destructive enough that it didn't seem like a good fit
  for a tap-only touchscreen control with no confirmation step (the
  Minecraft widget's Stop/Restart buttons get a "tap again to
  confirm" pattern for the same reason — delete would need something
  similar, and it isn't built yet).
- **Row cap** — both the relay (`qbittorrent.max_rows` in
  `config.json`) and the widget itself (its own "Torrent rows to
  show" setting in iCUE) independently cap how many torrents are
  listed. Whichever is smaller wins. If torrents you know exist don't
  show up, check both.
- **"Queued/stalled" vs. "stopped" is a best-effort split** —
  qBittorrent doesn't expose an intent flag ("will auto-resume" vs.
  "user meant to halt this"), so the widget infers it from the state
  string alone (`queuedDL`/`stalledDL`/etc. → amber "paused";
  `pausedDL`/`stoppedDL`/etc. → red "stopped"). This is a reasonable
  read of the state names but isn't authoritative.

## Things we learned building this one

A few of these cost real debugging time and are worth knowing before
touching this widget or building something similar:

- **Direct widget → qBittorrent doesn't work, and no amount of
  qBittorrent configuration fixes it.** The widget runs in a
  browser-based context (QtWebEngine), so it's subject to two
  independent browser rules that a relay is simply immune to:
  - **CORS** — qBittorrent sends no `Access-Control-Allow-Origin`
    header by default, and the only way to add one requires knowing
    the widget's exact origin string in advance.
  - **Cookie `SameSite` policy** — even *with* correct CORS headers,
    qBittorrent's session cookie won't be sent on a cross-origin
    `fetch()` from a browser context, because modern browsers default
    cookies to `SameSite=Lax`, which blocks it for non-navigation
    requests. This isn't something qBittorrent's settings can fix.

  The widget's own origin, for reference, turned out to be `file://`
  — an opaque origin that's a dead end for both of the above. The
  relay pattern (plain server-to-server HTTP, no browser involved)
  sidesteps both problems entirely, which is why every widget in this
  set works this way.

- **qBittorrent's login response shape isn't consistent across
  versions.** Older builds return `200 OK` with a body of `Ok.`.
  Newer builds (5.0+, confirmed via `curl -v` against a real
  instance) return `204 No Content` with an *empty* body — the
  session cookie is still set correctly either way, so the real
  signal of success is the cookie, not the response body. Checking
  for `200` + `"Ok."` specifically will falsely treat a working login
  as a failure.

- **qBittorrent 5.0+ (WebAPI v2.11+) renamed the pause/resume
  endpoints** to `/torrents/stop` and `/torrents/start`. The relay
  tries the new path first and falls back to the old
  `/torrents/pause` / `/torrents/resume` on a 404, so it works against
  either generation without needing to know the version up front.

- **Never classify torrent state by exact-set membership without a
  catch-all.** An earlier version of the relay only recognized a
  fixed list of state strings per group (downloading/seeding/paused);
  anything outside those lists — including, at one point, a torrent
  that had just been manually stopped — was silently dropped and
  never sent to the widget at all, rather than showing up miscolored.
  The fix was a classifier where every branch that isn't explicitly
  matched falls into a final catch-all group, so an unrecognized or
  future state string still shows up (as "stopped") instead of
  vanishing.

- **CSS Grid items (not just flex items) need explicit `min-width: 0`
  to truncate properly.** This bit us twice in the same widget. The
  well-known flexbox gotcha — a flex child defaults to `min-width:
  auto`, refusing to shrink below its content's natural width even
  inside a `flex: 1` container — has a Grid equivalent that's easy to
  miss: a CSS Grid item has the same default, so a `1fr` track can
  still get pushed wider than intended by content that "wants" more
  room. The first fix pass only addressed the flex chain *inside*
  each torrent row; the actual widget-breaking overflow (buttons and
  even plain text getting clipped off the right edge on the Small
  slot) turned out to be the grid column itself never having
  `min-width: 0`. Long torrent names or long status text (like the
  word "stopped") were enough to trigger it. Worth checking every
  level of a layout — grid tracks and flex items alike — when
  truncation isn't behaving.

- **A restarted scheduled task doesn't always mean a dead process.**
  More than once, a config change appeared to have no effect after
  restarting via `schtasks`, because the old `pythonw.exe` hadn't
  actually exited. `restart-relay.ps1` force-kills any lingering
  `pythonw.exe` before restarting the task, specifically to rule this
  out — if a config edit ever "doesn't seem to take," check that the
  process ID actually changed before assuming the code is wrong.

