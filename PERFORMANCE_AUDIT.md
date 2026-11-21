# m3u-tv Performance Audit — onn. Streaming Device 4K Pro

**Date**: 2026-09-08  
**Device**: onn. Streaming Device 4K Pro (Amlogic SC2, Mali-G31, 3GB RAM, Android 14)  
**App version**: dev.sparkison.tv (Flutter, Impeller Vulkan)  
**Connection**: Wireless ADB 192.168.2.93:44547

---

## Executive Summary

The app drops **83.5% of frames** on this low-end Android TV device. The primary bottleneck is **image oversampling** that floods the GPU with 258 MB of textures on a 3 GB device with only 47 MB free RAM, triggering swap thrashing. Secondary issues include cascading widget rebuilds and a navigation architecture that keeps all 10 branches mounted simultaneously.

---

## Device Profile

| Spec | Value |
|------|-------|
| Device | onn. Streaming Device 4K Pro |
| SoC | Amlogic SC2 (Cortex-A55, quad-core) |
| GPU | Mali-G31 |
| Max CPU freq | 2.0 GHz |
| Android | 14 (SDK 34) |
| RAM | 2.99 GB (47 MB free, ~1 GB available) |
| Renderer | Impeller Vulkan (gfxinfo misreports "Skia OpenGL") |
| Thermal | 48.4°C (below 65°C throttle threshold) |

---

## Frame Rendering (gfxinfo)

| Metric | Value | Target |
|--------|-------|--------|
| Total frames rendered | 1,739 | — |
| Janky frames | **1,454 (83.6%)** | <5% |
| 50th percentile | **27 ms** | <16.6 ms |
| 90th percentile | **57 ms** | <16.6 ms |
| 95th percentile | **61 ms** | <16.6 ms |
| 99th percentile | **150 ms** | <33 ms |
| Missed Vsync | **794** | 0 |
| Slow UI thread | **1,120** | 0 |
| Slow draw commands | 303 | — |
| GPU p50/p90 | 6 ms / 10 ms | — |

**Conclusion**: GPU is not the bottleneck. The Dart/Flutter UI thread is the limiting factor — 1,120 slow UI thread frames vs. 303 slow draw commands.

### Frame Time Histogram

```
 5ms:   0     20ms:  91     35ms:  25     57ms: 320  ← 3× frame budget cluster
10ms:   5     22ms:  97     40ms:  12     61ms:  43
12ms:  20     25ms:  62     45ms:  11     81ms:   2
15ms:  34     27ms:  69     48ms:  41    150ms:  13  ← severe stutter
17ms:  56     30ms:  35     53ms:  33    200ms:   5
19ms:  36     32ms:  65     57ms: 320    450ms:   2
```

Bimodal distribution: ~500 frames at 20-27ms (1 VSync miss) and 320 frames at exactly 57ms (2 VSync misses — the UI thread completes just as the next VSync arrives).

---

## Memory

### Process Memory (dumpsys meminfo)

| Component | PSS (KB) | Private Dirty |
|-----------|----------|---------------|
| **GL mtrack** | **258,104** | **258,104** |
| Native Heap | 22,047 | 21,092 |
| Unknown | 74,550 | 74,276 |
| .apk mmap | 14,688 | 140 |
| .so mmap | 6,250 | 160 |
| Dalvik Heap | 4,687 | 2,256 |
| **TOTAL PSS** | **402,980** | — |
| **TOTAL RSS** | **498,800** | — |
| **SwapPss** | **12,599** | — |

### System Memory

| Metric | Value |
|--------|-------|
| MemTotal | 2,991,892 KB |
| MemFree | 47,804 KB |
| MemAvailable | 1,072,408 KB |
| kswapd0 | **Active** — actively reclaiming pages |
| lmkd | **Woken up** — borderline OOM |

### GPU Memory

| Metric | Value |
|--------|-------|
| GL mtrack | **258 MB** |
| Skia GPU cache | 0 bytes (6 image entries — textures not recycled) |
| Max Skia resource budget | 99.5 MB (x48 entries) |
| Max surface area | 2,073,600 px (1080p) |

---

## Scheduling & CPU

### CPU Frequency

- Max: 2004 MHz (2.0 GHz)
- Governor: `sugov` (schedutil) — fluctuating 1.4–1.8 GHz under load
- Currently running at max during trace capture

### Thread Preemption (atrace)

The Flutter main thread (PID 5300, prio=110) is **regularly preempted** by:

| Thread | Priority | Impact |
|--------|----------|--------|
| `RTW_RX_CB_THREA` (WiFi RX) | **98 (RT)** | Preempts UI 5+ times in burst — WiFi interrupt handler starves Dart |
| `surfaceflinger` | 97 (RT) | Normal compositing |
| `android.hardwar` (HWC) | 97 (RT) | Display hardware |
| `lmkd` | 98 (RT) | Memory pressure wakeups |
| `kswapd0` | 115 | Page reclaim — causes I/O stalls |

### HwBinder D-State (ExoPlayer/MediaCodec)

`HwBinder:5300_1` (PID 5586) enters uninterruptible sleep (D-state) **10+ times** in the 10-second trace — blocked waiting for page-in from swap. This directly causes video decode stalls.

---

## Source Code Analysis

### 1. Image Oversampling (CRITICAL — 258 MB GPU)

**Files**: `lib/shared/cached_media_thumbnail.dart:36`, `lib/shared/media_browsing_widgets.dart:480-481`

```dart
// cached_media_thumbnail.dart:36
static const double _oversample = 2;

// media_browsing_widgets.dart:480-481
final devicePixelRatio =
    MediaQuery.devicePixelRatioOf(context) * 2 * TvZoomScale.of(context);
```

Both multiply `devicePixelRatio × 2 × TvZoomScale (1.4×)` = **2.8× effective multiplier**.

A 120dp thumbnail decodes to **~672px** instead of ~240px. Each visible card on the home screen holds a ~672×576 RGBA texture (~1.5 MB). With ~48 visible cards + backdrops + player surface = **258 MB GPU**.

### 2. No ImageCache Size Limit

**File**: `lib/main.dart` — no calls to `PaintingBinding.instance.imageCache.maximumSize` or `maximumSizeBytes` anywhere in the codebase.

Flutter defaults to 1000 images / 100 MB. With 2.8× oversampling, decoded images accumulate unbounded.

### 3. IndexedStack Keeps 10 Branches Mounted

**File**: `lib/navigation/go_router_config.dart:55`

```dart
StatefulShellRoute.indexedStack(
  branches: [
    // 10 branches: Home, Search, Live TV, VOD, Series, AIOStreams, DVR, Requests, Notifications, Settings
  ],
)
```

All visited branches stay **alive and mounted** simultaneously, holding focus nodes, scroll controllers, local state, and GPU textures.

### 4. AppStateController Cascading Rebuilds

**File**: `lib/providers/app_providers.dart:45-64`

Every `AppStateController.notifyListeners()` fires **all ~20 providers** derived from `appStateControllerProvider`. A single progress update triggers rebuilds of `liveChannelsProvider`, `vodItemsProvider`, `seriesListProvider`, `progressListProvider`, `dvrRecordingsProvider`, etc.

### 5. HomeScreen Rebuilds 8 Providers

**File**: `lib/app/app_shell.dart:2329-2340`

```dart
ref.watch(isBootstrappingProvider);
ref.watch(isConfiguredProvider);
ref.watch(progressListProvider);
ref.watch(liveChannelsProvider);
ref.watch(vodItemsProvider);
ref.watch(seriesListProvider);
ref.watch(epgServiceProvider);
ref.watch(dvrRecordingsProvider);
```

During playback, progress reporter fires every 10s → full HomeScreen rebuild mapping entire channel/VOD catalogs into `MediaPreviewItem` lists.

### 6. PlayerScreen 500ms Timer Full Rebuilds

**File**: `lib/features/player/player_screen.dart:404-416`

```dart
_positionTimer = Timer.periodic(const Duration(milliseconds: 500), (_) {
  setState(() { _currentPosition = next; });
  _checkComskip();
  _checkIntroDb();
  _checkUpNext();
});
```

2 full `Stack` rebuilds/sec during playback, each with 7+ conditional `Positioned` children.

### 7. PaletteGenerator Re-decodes on Every Detail Navigation

**File**: `lib/shared/dominant_backdrop_color.dart:17-33`

Each VOD/Series detail screen open triggers `PaletteGeneratorMaster.fromImageProvider` — redundant image decode at 220×124 for color extraction.

### 8. ExoPlayer Probe Only Supports H.264

**File**: `lib/navigation/app_router.dart:134-140`

`AndroidPlaybackProbe` configured with `hardwareCodecs: {VideoCodec.h264}` only — no H.265/HEVC. Amlogic SC2 supports HW HEVC decode, but falls back to mpv/software decode.

### 9. EPG Map Rebuilt Inside build()

**File**: `lib/features/live_tv/live_tv_screen.dart:688-690`

`_loadEpgForChannels(filtered, epgService)` iterates every channel inside `build()` — O(n) scan on every rebuild.

---

## Improvement Recommendations

### Tier 1 — Biggest Impact (Fix These First)

| Fix | Expected Impact | Effort |
|-----|-----------------|--------|
| **Remove `_oversample = 2`** — set to 1 or remove entirely | **-60% GPU memory** (258→~90 MB), reduces swap pressure | 1 line change |
| **Cap `imageCache.maximumSizeBytes = 50 * 1024 * 1024`** in `main.dart` | Prevents unbounded decoded image accumulation | 1 line addition |
| **Replace `StatefulShellRoute.indexedStack` with `StatefulShellRoute`** | Branches built on demand, not kept alive — **-50% baseline memory** | Router refactor |

### Tier 2 — High Impact

| Fix | Expected Impact | Effort |
|-----|-----------------|--------|
| Split `AppStateController` into domain-specific ChangeNotifiers | **-70% unnecessary rebuilds** | Medium refactor |
| Extract HomeScreen section widgets as independent ConsumerWidgets | Prevents full-home rebuilds on progress ticks | Medium refactor |
| Replace player position `setState` with `ValueNotifier` + `AnimatedBuilder` | **-90% player rebuilds** | Small refactor |
| Disable Impeller on Mali-G31 (`EnableImpeller=false`) | May improve frame times on this specific GPU | 1 XML change |

### Tier 3 — Medium Impact

| Fix | Expected Impact | Effort |
|-----|-----------------|--------|
| Enable H.265/HEVC in ExoPlayer hardware codecs | Eliminates software decode fallback | Small change |
| Cache PaletteGenerator results by URL | Avoids redundant decodes on revisit | Small change |
| Pre-build EPG map in listener callback, not build() | Eliminates O(n) scan per rebuild | Small refactor |

---

## Root Cause Summary

```
Image oversampling (2.8×)
    ↓
258 MB GPU textures on 3 GB device
    ↓
47 MB free RAM → kswapd active → swap thrashing
    ↓
HwBinder D-state (video decode stalls)
WiFi RT thread preempts UI thread
IndexedStack keeps 10 branches alive (all holding textures)
    ↓
83.5% janky frames, 57ms p90, 150ms p99
```

---

## Appendix: Thread List (PID 5300)

Key threads from `ps -T`:

| Thread | PID | Role |
|--------|-----|------|
| `ev.sparkison.tv` (main) | 5300 | Flutter UI thread |
| `RenderThread` | 5326 | Android HWUI render |
| `dart:io EventHa` | 5347 | Dart I/O event handler |
| `6.raster` | 22089 | Flutter raster thread (Impeller) |
| `6.io` | 22090 | Flutter I/O thread |
| `IplrVkFenceWait` | 22105 | Impeller Vulkan fence |
| `IplrVkResMgr` | 22106 | Impeller Vulkan resource manager |
| `ExoPlayer:Playb` | 22386 | ExoPlayer playback |
| `ExoPlayer:Loade` | 22397 | ExoPlayer loader |
| `MediaCodec_loop` | 22398 | MediaCodec decode loop |
| `HWC release` | 22460 | Hardware composer release |
| `GPU completion` | 22461 | GPU completion callback |
| `InteractionJank` | 5411 | Android Jank detector |
| `Sqflite` | 5433 | SQLite worker |
