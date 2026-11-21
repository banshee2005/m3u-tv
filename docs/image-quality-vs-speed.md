# Image Quality vs Speed Setting

## Overview
The "Optimize for" setting in View Settings controls image rendering quality:

- **Quality**: Full oversampling (2x), high filter quality, 100MB image cache
- **Speed**: No oversampling, medium filter quality, 50MB image cache

## Persistence
Both settings persist via `ViewSettingsService` using the `PersistentJsonStore`:
- `m3ue_tv_optimize_for` key stores "quality" or "speed"

## Widget Tree Propagation
The setting flows through:
1. `ImageQualityScope` InheritedWidget — provides `oversample` and `filterQuality` to child widgets
2. `main.dart` listener — adjusts `PaintingBinding.instance.imageCache` limits when the setting changes
3. `TimelineEpgView` — uses `OptimizeFor.speed` to simplify EPG rendering on low-end devices

## Image Cache Behavior
| Setting | Max Cache Size | Max Entries | Oversample | Filter Quality |
|---------|---------------|-------------|------------|----------------|
| Quality | 100MB | 1000 | 2x | High |
| Speed | 50MB | 200 | 1x | Medium |

When the setting changes, cached images are cleared so they re-decode at the new quality level.
