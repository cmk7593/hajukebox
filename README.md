# JukeBox — Home Assistant + Music Assistant + SyncLyrics

A single-file HTML/JS "jukebox" UI: album art with a spinning-vinyl idle
state, transport controls, a speaker picker, search-and-queue for a
music library, and synced lyrics. Originally built as one view inside a
larger custom Home Assistant dashboard; this export pulls out just the
jukebox-related code so it can be dropped into a new project.

## What it talks to

There are three separate connections, and it's worth understanding them
as three different systems before you start editing:

1. **Home Assistant WebSocket API** (`/api/websocket` on your HA host)
   — used for auth, reading `media_player` entity state, and generic
   `call_service` calls (play/pause/skip/volume/join-speakers). This is
   the same connection pattern any custom HA dashboard uses.

2. **Music Assistant's own WebSocket API** (default port `8095`,
   separate from HA's). Music Assistant (MA) is a HACS/HA add-on that
   sits in front of your actual music sources (Spotify, local library,
   etc.) and exposes a richer search/queue API than HA's generic
   `media_player` services support. The jukebox connects to it directly
   for library search and queue manipulation — `jbMaWsOpen`,
   `jbMaWsSend`, and everything under `// ── Search ──` /
   `// ── Add to queue ──` in the script.

3. **[SyncLyrics](https://github.com/AnshulJ999/SyncLyrics)** — a small
   self-hosted server that fetches time-synced lyrics for whatever is
   currently playing and serves a minimal embeddable HTML view. The
   jukebox just points an `<iframe>` at it (`jb-lyrics-frame`); all the
   actual lyric-fetching and sync logic lives in that separate project,
   not in this code. You need to run that server yourself and point
   `SYNCLYRICS_URL` at it.

## Setup

1. **Home Assistant long-lived token** — Profile → Security →
   Long-Lived Access Tokens. Paste it into `haToken` near the top of
   the `<script>` block.
2. **Music Assistant token** — MA issues its own separate token (not
   your HA token). Paste it into `jbMaToken`. If you're serving this
   file from the same browser session as your HA frontend, you may be
   able to swap this for MA's normal session auth instead — this
   export uses the token approach the original dashboard used.
3. **Speaker roster** — edit the `SONOS` array (`entity_id`, display
   name pairs) to match your own `media_player.*` entities.
4. **Queue IDs** — `JB_QUEUE_IDS` maps each speaker to its Sonos
   `RINCON_...` queue id, which MA needs for some queue operations.
   The values in this export are placeholders
   (`RINCON_XXXXXXXXXXXX01400`) — you can read the real ones off the
   `active_queue` attribute of each `media_player` entity's state in
   HA's Developer Tools once things are connected, then fill them in.
5. **SyncLyrics** — stand up the SyncLyrics server (see its repo) and
   set `SYNCLYRICS_URL`, e.g. `http://homeassistant.local:9012`.
   Leave it blank and the code will try to auto-derive a URL from
   whatever host is serving this page (port 9012 for http, 9013 for
   https) — that's a convenience for the original same-host setup and
   you'll likely want to just hardcode your own URL instead.
6. Open the HTML file in a browser served from (or with network access
   to) your HA instance. This file makes no server-side calls of its
   own — everything is a client-side WebSocket connection from the
   browser.

## File map

- **CSS** — `:root` color variables, then the `.jb-*` classes: the
  cabinet/screen/neon-tube visual, now-playing card, transport
  buttons, speaker chips, volume slider, lyrics card, search results
  list, and queue list.
- **HTML** — one `<div>` tree: left column is the "machine" (art,
  now-playing, transport, volume, quick-group buttons), right column
  is lyrics iframe + search box + queue.
- **JS**, roughly top to bottom:
  - Generic HA WebSocket boot (`openWS`, `onMsg`, `onAuth`, `svc`) —
    this is shared plumbing any HA dashboard needs, kept here because
    the jukebox functions call `svc()` and read `states{}` directly.
  - `SONOS` / `JB_QUEUE_IDS` — your speaker config.
  - MA WebSocket (`jbMaWsOpen`/`jbMaWsSend`) — separate connection,
    separate auth, separate message protocol from HA's.
  - `jbRender` / `jbDiscover` — entry point that sets up speaker
    buttons and kicks off polling.
  - Now-playing polling + progress bar (HA state → UI, plus a local
    ticking clock between state updates so the bar doesn't visibly
    stutter).
  - Queue rendering, staging (batch-add multiple search results before
    sending), search (including a "library rescue" workaround for a
    Music Assistant quirk where combined artist+title queries don't
    always match well), and voice search via the Web Speech API.
  - Lyrics iframe wiring (`jbInitLyrics`) — mostly just pointing the
    iframe `src` at your SyncLyrics server with the right query params
    for the current track.

## Known rough edges (carried over from the source)

- `jbStartRecognition`'s result handler looks for an element id
  (`jb-search-input`) that doesn't actually exist in the markup — it's
  guarded (`if (inp) inp.value = ...`) so it's harmless, but voice
  search won't visibly populate the search box even though it still
  fires `jbSearch()` with the transcript. Worth an id fix if you want
  the visual feedback.
- Two secrets were stripped from this export (`haToken`, `jbMaToken`)
  — you must supply your own for either system to authenticate.
- There are queuing issues if you try to use another service directly,
  Spotify, for example, so I would recommend only using queueing related
  to Music Assistant.
