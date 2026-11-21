# Record Button: First-Press Issue & Controls Bar Behavior

## Problem 1: Record Button Requires Two Presses
The record button didn't work on the first press — users had to tap it twice.

### Root Cause
The `_refreshActiveDvrRecordings()` method (called on WebSocket reconnect and periodically) was **overwriting** `_recordingChannelIds` with only actively-recording channels from the server. A just-scheduled channel (still in `scheduled` status) was stripped out before the server transitioned it to `recording`.

### Fix
In `_refreshActiveDvrRecordings()`, after computing the server's active recording set, also include channels that have a `scheduled` recording in `_dvrRecordings`. This preserves the optimistic state until the server's push transitions the recording.

Additionally, `scheduleDvrAiring()` now:
1. Adds the channel to `_recordingChannelIds` **before** the network call (optimistic)
2. If the network call fails, rolls back the optimistic update
3. The background refresh preserves the channel in `_recordingChannelIds` even if the server hasn't transitioned yet

### Files Changed
- `app_state_controller.dart` — `scheduleDvrAiring`, `scheduleDvrAirings`, `_refreshActiveDvrRecordings`

## Problem 2: Controls Bar Disappears Before Recording Status Is Known
After tapping record, the controls bar would hide before the user could see whether the recording succeeded or failed.

### Root Cause
`_dvrSchedulePending` was cleared in `didUpdateWidget` when `isRecordingCurrentChannel` changed. With the optimistic fix, this happened immediately — before the network call completed.

### Fix
1. Changed `onRecordProgram` callback to return `Future<void>` (was `void`)
2. PlayerScreen now `await`s the callback
3. After the callback completes, waits 1 second (so user sees stop icon on success or error SnackBar on failure), then clears `_dvrSchedulePending` and hides the overlay
4. Removed the `didUpdateWidget` auto-clear of `_dvrSchedulePending`

### Behavior After Fix
- **Success**: Button shows stop icon → 1 second hold → overlay hides
- **Failure**: Error SnackBar appears → 1 second hold → overlay hides
- **User taps stop during pending**: Shows "Recording is being scheduled..." message

### Files Changed
- `player_screen.dart` — `onRecordProgram` signature, `onRecordNow` callback, `didUpdateWidget`
- `app_shell.dart` — `_handleRecordButtonTap` returns `Future<void>`, `_scheduleDvr` returns result
- All 5 ARB files — new `appRecordingScheduling` key
