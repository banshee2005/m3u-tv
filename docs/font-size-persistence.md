# Font Size Persistence

## Problem
The font size setting (Normal / Large / Very Large) did not persist across app restarts. Every time the app launched, it reset to Normal.

## Root Cause
`main.dart` pre-loaded `optimizeFor` and `defaultStartPage` into the in-memory cache before `runApp()`, but not `fontSize`. The `fontSizeSync` getter reads from the in-memory `_memory` map, which was empty on first build — so it always returned `AppFontSize.normal`.

## Fix
Added `await appState.viewSettingsService.fontSize()` in `main.dart` before `runApp()` to populate the memory cache on boot.

## File Changed
- `flutter_client/lib/main.dart` — line ~52

## Behavior After Fix
1. User sets font size to Large in Settings
2. App is closed and reopened
3. Font size remains Large
