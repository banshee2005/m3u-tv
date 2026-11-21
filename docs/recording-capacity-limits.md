# DVR Recording Capacity & Stream Limits

## Overview
m3u-editor enforces stream capacity at multiple levels. All limits must pass for a new stream (DVR or live) to be created.

## Capacity Hierarchy

### 1. Playlist-Level Limit (`available_streams`)
- Set per playlist in the admin panel
- `0` = unlimited (no enforcement)
- Represents the provider's total connection cap for that account
- **DVR and live streams share this pool** — both count against it

### 2. Merged Playlist Limit
- Merged playlists can have their own `available_streams`
- The effective limit is `min(merged_playlist, source_playlist)` — the tighter of the two
- Example: merged=3, source=2 → effective limit=2

### 3. Per-User Limit (`PlaylistAuth.max_connections`)
- Constrains individual users within a playlist
- Separate from the global pool

### 4. DVR-Specific Limit (`max_concurrent_recordings`)
- Default: 2 (configurable via `DVR_MAX_CONCURRENT_RECORDINGS` env var)
- Checked at schedule time AND at trigger time

### 5. Provider-Level Limit
- The Xtream API provider's own `max_connections` per account
- Enforced when the stream actually connects
- If exceeded, the provider rejects the connection

## How Limits Are Checked

### When Scheduling a DVR Recording (`schedule_dvr` API)
1. Per-user DVR limit (`hasReachedConcurrentLimit`)
2. Global DVR limit (`isAtCapacity`)
3. **Global stream limit**: active DVR + active live streams ≥ `available_streams`

### When Starting a Live Stream (`getChannelUrl`)
1. Source playlist `available_streams` (active live + active DVR)
2. Merged playlist `available_streams` (if set, tighter limit applies)
3. Per-auth `max_connections`
4. Provider profile limits

### When the Provider Rejects
- Server returns JSON error on stream routes
- Flutter client fetches and displays the actual error message

## Error Messages

| Scenario | HTTP Status | Message |
|----------|------------|---------|
| Concurrent recording limit (per-user) | 422 | "Concurrent recording limit reached" |
| Concurrent recording limit (global) | 422 | "Concurrent recording limit reached" |
| Playlist at capacity | 503 | "Playlist has reached maximum stream limit." |
| Per-auth limit exceeded | 503 | "You have reached your maximum allowed concurrent streams." |
| No provider profiles available | 503 | "No provider profiles available." |
| Provider rejects (max_connections) | 500 | Provider's error message surfaced to user |

## Files Changed
- `app/Http/Controllers/XtreamApiController.php` — scheduleDvr global capacity check
- `app/Services/M3uProxyService.php` — merged playlist min(merged, source) logic
- `bootstrap/app.php` — stream routes return JSON errors
- `flutter_client/lib/playback/playback_orchestrator.dart` — fetch and display server errors
