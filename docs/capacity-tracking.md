# Capacity Tracking: DVR & Live Playback

> **Architecture note (Sep 10):** Upstream (sparkison) merged their own pooled-provider
> work — live via merged playlists, profile stream-ID mapping, and DVR drawing from
> the pool via `getChannelUrl` with a "DVR always wins" raw-URL fallback. This branch
> is now a **small enhancement subset on top of upstream's code** instead of a
> replacement. The original full implementation (selectAndReserveProfile-based DVR
> mapping + `channel_profile_map`) is preserved on the fork under
> `feat/capacity-original-approach` in case upstream's approach needs revival.

## Goal

Enforce per-provider stream capacity so that:

- A DVR recording is rejected cleanly (422 at schedule, or refused at start) when the provider/playlist is full — **never** evicting an existing recording.
- A live stream is rejected cleanly (503 with a JSON message shown in the TV app) when capacity is exhausted — instead of failing with a generic "Source error" or silently evicting.
- Same-client channel switches still work (the only sanctioned eviction is a same-client live stream during a switch).

Capacity is enforced **per provider account** (pool). The pools in the current setup:

| Pool | Playlist | `available_streams` | DVR setting | `max_concurrent_recordings` |
|------|----------|--------------------:|------------:|-----------------------------:|
| Antenna (HDHomeRun, 4 tuners) | 7 | 4 | 6 | 4 |
| 2 Step 2 Provider | 79 | 2 | 5 | 2 |
| Primary (Ganja) | 73 | 1 | — | — |
| Secondary (Ganja) | 77 | 1 | — | — |
| Merged "Strong + HDHomerun" | 6 (merged) | 0 (unlimited) | — | — |

DVR and live streams share the same pool — both count against `available_streams`.

## Key Architectural Facts

- **The merged playlist must have `enable_proxy = true`.** Live requests through a proxy-disabled merged playlist redirect directly to the provider, bypassing `getChannelUrl()`'s capacity checks entirely. This was a silent bypass discovered mid-session: source playlists had proxy enabled, the merged playlist did not.
- Streams created via the proxy carry metadata keyed by the **provider playlist UUID** (e.g. `a1016255-…` for Primary), not the 2-step or merged UUID. Capacity counting must therefore be aware of which UUID a stream belongs to. Comparing a merged/2-step UUID to a provider UUID never matches — a bug class that bit twice.
- DVR and live use the **same counting primitives** (distinct active channels, piggyback-aware), so a channel that is both watched and recorded consumes one provider connection.

## Server-Side Work (m3u-editor — enhancement subset on upstream)

Upstream (dev) already provides: pooled-provider playback via merged playlists
(`$needsProxy` forced when the source playlist pools profiles), profile
stream-ID mapping, and DVR drawing from the pool via `getChannelUrl()` with a
raw-URL fallback ("DVR always wins"). On top of that, this branch adds:

- **Eviction notification** — `DvrRecorderService::start()` snapshots active
  live streams before resolving the URL through the pool; any stream a
  DVR-wins eviction stopped gets a TV notification to its owner's devices:
  "A DVR recording has taken precedence and stopped your live stream."
- **Bounded start retries** — `StartDvrRecording` retries up to 3× with 15s
  delays while the programme is on air (transient provider/tuner failures
  often clear within seconds).
- **Stale reservation pruning** — `ProfileService::pruneStaleReservations()`
  runs inside `selectAndReserveProfile()`: pending `reservation:xxx` entries
  whose reverse channel mapping (30s TTL) expired are phantom slots and are
  removed before capacity is counted.
- **Schedule-time accuracy (fixed-tuner pools)** — `scheduleDvr` resolves the
  DVR setting through merged/custom channels (`resolveDvrSetting`) and, when
  the source playlist has a fixed `available_streams` (e.g. Antenna/HDHomeRun
  tuners), refuses at schedule time when the pool is full. The count includes
  recording rows + **running broadcasts** (whose rows may have left Recording
  during stop drain) + **imminent scheduled recordings** (starting within 10
  minutes, so rapid-fire taps can't slip past).
- **`DvrSetting::isAtCapacity`** counts only `Recording` status —
  PostProcessing recordings have already released the tuner.
- **JSON errors on stream routes** — `bootstrap/app.php` adds the live/vod/
  series/timeshift stream routes to `shouldRenderJsonWhen` so the TV client
  can read capacity errors instead of an HTML page.
- **Helpers** — `M3uProxyService::getActiveLiveChannelIds()`,
  `getRunningDvrRecordingIds()`, `getActiveLiveStreams()`.

> The profile-capacity *refusal* (fail-the-recording-when-pool-full) that the
> original approach implemented was dropped in favor of upstream's "DVR always
> wins" policy — the recording evicts an older live stream instead of failing.
> The notification makes the eviction visible to the affected viewer.

### The `failoverSelected` abort fix (live path — superseded by upstream)

The original branch fixed a live-path abort bug (merged UUID vs source UUID
comparison). Upstream's pooling rework keys pool lookups on the SOURCE playlist
UUID instead, which addresses the same class of bug — so this fix is no longer
carried here.

### Profile reservation leak fixes (from the original approach)

The original branch's explicit reservation lifecycle
(`releaseProfileReservation`, delete-path releases, crash-recovery broadcast
stops) is NOT carried in the reduced branch — upstream's webhook-based
decrement handles reservations for proxy streams. The prune (above) covers the
phantom-slot leak class. The full lifecycle code remains on
`feat/capacity-original-approach`.

### JSON errors on stream routes

`bootstrap/app.php` — stream routes added to `shouldRenderJsonWhen`, so capacity/abort responses are JSON (`{"message": "…"}`) instead of HTML. The native player cannot parse JSON, which is why the client-side pre-flight exists (below).

### Provider-profile mapping for DVR (the "2-step" fix)

DVR recordings previously used the channel's **raw URL**, which for 2-step channels always points at **Primary** (`available_streams = 1`). Two 2-step recordings therefore both piled onto Primary: the 2nd stayed stuck at "Scheduled" and live playback of 2-step channels failed with "Source error" (Primary full, no Secondary fallback). DVR now uses the same profile mechanism as live playback:

- `DvrRecorderService::start()` — when the channel's source playlist has `profiles_enabled`, calls `ProfileService::selectAndReserveProfile()` (atomic select + increment, TOCTOU-safe) and transforms the URL with `PlaylistUrlService::getChannelUrl($channel, $selectedProfile)`. Selection is verified to distribute: recording #1 → Primary, #2 → Secondary, #3 → refused.
- Reservation lifecycle: `provider_profile_id` + `provider_reservation_id` are stored in the recording's `metadata` JSON column. Released via `ProfileService::cancelReservation()` on:
  - manual stop (`stop()`)
  - cancel (`cancel()`)
  - natural programme end (`programme_ended` callback → `DvrCallbackController::handleRecordingStopped`)
  - broadcast failure (`broadcast_failed` callback → `handleBroadcastFailed`)
  - crash recovery (`recoverFromCrash()`)
  - `startDvrBroadcast()` throwing (slot released before rethrowing)
  - release is idempotent (metadata cleared on first release)
- If no profile has capacity, the recording is marked Failed with a clear message instead of evicting anything.

### Zombie-broadcast crash recovery fix

`recoverFromCrash()` previously only marked stale `RECORDING` rows as Failed in the DB. The proxy-side FFmpeg broadcast kept running, holding the provider/tuner slot **for hours** — this was the root cause of "my other recording got deleted": the pool was silently occupied by zombie broadcasts from before a container restart, so every new recording (and 2-step/HDHomeRun playback) hit 503 → "FFmpeg exited with code 8" → cascading failures.

Now `recoverFromCrash()` calls `stopDvrBroadcast()` + `cleanupDvrBroadcast()` for each stale recording's `proxy_network_id` **before** clearing it.

### Config

- Antenna: `available_streams = 4`, DVR setting max = 4
- 2-step: `available_streams = 2`, DVR setting max = 2
- Primary / Secondary: `available_streams = 1` each
- Merged playlist: `available_streams = 0`, `enable_proxy = true`
- Orphaned/duplicate DVR settings and stale schedule rules cleaned up

## Client-Side Work (m3u-tv Flutter app)

### Self-healing DVR status poll

The app previously relied entirely on the Reverb push channel for recording-status updates — when the WebSocket was down (or never connected), the red dots and status labels went stale forever ("recording dot stays on a failed recording", "stop button says 'being scheduled'").

- `_ensureActiveDvrPoll()` — a 30-second poll of the server's authoritative `status=recording` **and** `status=scheduled` lists, started from **every** DVR touchpoint (notification connect, `refreshDvrRecordings`, `scheduleDvrAiring`) so a failure in any one path can't leave the app without its self-healer. The dot set is recomputed from the two server lists each tick — a recording that started (or failed) moments after a refresh corrects itself within 30s.
- The poll reconciles against **both** lists: previously it preserved channels whose *local* row said "scheduled", so a recording that FAILED while its row was stale kept its dot forever.
- Stop-button fallback: if the red dot shows but no in-progress row is found locally, the app fetches the authoritative list once and shows the stop dialog if the server confirms the recording — instead of the misleading "Recording is being scheduled…" snackbar.

### Pre-flight live stream check (`playback_orchestrator.dart`)

The native player (ExoPlayer/mpv) can't render a JSON error body — a 503 with `{"message": …}` surfaces as a generic "Source error" after seconds of retrying. Two changes:

1. `_preflightLiveStream(url)` — before opening the player, GET the live URL with redirects disabled. If the server returns ≥ 400 with a JSON `message`, emit that message immediately and skip playback entirely (fast, deterministic — mirrors how DVR scheduling reports errors instantly).
2. `_fetchStreamError(url)` — fallback: on any live playback failure, fetch the server's JSON message and surface it instead of the generic player error. Previously this only ran for probe-style failures, which "Response code: 503" doesn't match.

Both are only used for live sources (`source.isLive`); VOD playback is unaffected.

## Current Behavior (verified)

- **Antenna**: 4 concurrent recordings OK; 5th channel → immediate capacity message at schedule time. Live playback of a 5th channel while 4 are recording → capacity message (pre-flight).
- **2-step**: two recordings distribute across Primary + Secondary; 3rd channel (record or play) → immediate "No provider profiles available." at schedule time.
- Capacity errors surface immediately (422 at schedule) with the server's own message; a transient start failure retries up to 3× before failing.
- Red dots and status labels self-heal within 30s from the server's authoritative lists, even with the push channel down.
- Zombie broadcasts no longer block pools after a restart (crash recovery stops them) or after a delete (delete stops them).

## Known Transient Issue

- **Network flakes between the TV and server** ("Connection closed before full header was received" on the WebSocket and on schedule requests) can make a single record tap fail with an HTTP error while the next tap succeeds. The poll self-healing now survives these, but the underlying Wi-Fi/server connection instability remains a separate networking investigation.

## Limitations & Next Steps

- **VOD / Series episodes were not touched.** The episode path (`M3uProxyService::getEpisodeUrl`) mirrors the live path and likely still has the old UUID-comparison abort logic and count semantics that caused the live bugs. Capacity enforcement for VOD/episode streaming should be audited and brought in line with the live path (distinct-channel counting, `failoverSelected`-style abort, JSON error surfacing, profile mapping).
- **External consumers of the HDHomeRun are invisible to the server.** A tuner held by another application (or a lingering HDHomeRun session from a failed broadcast) can make a 4th recording fail at start even though the server counted 3. The failure is clean (existing recordings untouched) and the start-retry usually succeeds once the tuner frees. The server cannot enumerate hardware tuner state.
- **Zombie broadcasts after DB-row deletion without a restart**: if a recording row is deleted through a path that bypasses `releaseProxyResources`, the broadcast keeps running. The crash-recovery fix covers rows at boot; a periodic reconciliation that stops broadcasts whose `recording_db_id` has no matching DB row would close the remaining gap.
- **`proxy_stop_oldest_on_limit` semantics**: only same-client live eviction is sanctioned; any future change must keep DVR recordings non-evictable.
- **Network flakes** (TV↔server "Connection closed before full header") cause occasional one-off HTTP errors on record taps and push drops; the app now self-heals via polling, but the connectivity issue itself is worth investigating.

## Files Changed

Server (`m3u-editor`) — branch `feat/capacity-tracking` (on `banshee2005/m3u-editor`):
- `app/Services/DvrRecorderService.php` — eviction notification on upstream's DVR-wins flow
- `app/Jobs/StartDvrRecording.php` — bounded start retries
- `app/Services/ProfileService.php` — `pruneStaleReservations()` in `selectAndReserveProfile`
- `app/Http/Controllers/XtreamApiController.php` — `resolveDvrSetting`, `isAtCapacity` check, fixed-tuner schedule accuracy (running broadcasts + imminent scheduled)
- `app/Models/DvrSetting.php` — `isAtCapacity()` counts only Recording
- `app/Services/M3uProxyService.php` — `getActiveLiveChannelIds()`, `getRunningDvrRecordingIds()`, `getActiveLiveStreams()`
- `bootstrap/app.php` — stream routes render JSON errors

Original full implementation preserved on `feat/capacity-original-approach` (same fork):
- selectAndReserveProfile-based DVR mapping, `channel_profile_map` in `PlaylistProfile::resolveInternalUrl()`, explicit reservation lifecycle (release on stop/cancel/failure/delete), `failoverSelected` abort fix, schedule-time profile refusal, zombie-safe crash recovery, delete-allows-in-progress.

Client (`m3u-tv`) — branch `feat/tv-ui-and-dvr-capacity` (pending fork of `m3ue/m3u-tv`):
- `flutter_client/lib/playback/playback_orchestrator.dart` — `_preflightLiveStream()`, `_fetchStreamError()` on all live failures, skip retries on server rejection
- `flutter_client/lib/services/app_state_controller.dart` — `_ensureActiveDvrPoll()` (30s poll of recording+scheduled lists from every DVR touchpoint), poll reconciliation against server truth
- `flutter_client/lib/app/app_shell.dart` — stop-button refresh fallback before showing "being scheduled"

DB/config:
- Merged playlist `enable_proxy` f→t on the user's server (upstream now forces the proxy path for pooled playlists automatically, so this is no longer strictly required)
- Antenna / 2-step / Primary / Secondary `available_streams`, DVR `max_concurrent_recordings`