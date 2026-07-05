# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Open-source live (video) wallpaper app for macOS 14+. Menu-bar app with a SwiftUI configuration window; actual wallpaper playback is done by a separate helper binary (`wallpaperdaemon`), one process per display.

## Build

The project builds with Xcode (requires full Xcode, not just Command Line Tools — `xcodebuild` fails if `xcode-select` points at CommandLineTools).

```sh
git submodule update --init            # vendor/yaml-cpp is required
xcodebuild -project LiveWallpaper.xcodeproj -scheme LiveWallpaper build
```

Two targets: `LiveWallpaper` (the app) and `wallpaperdaemon` (helper binary). The app target embeds the daemon binary into the bundle at `Contents/MacOS/wallpaperdaemon` via a CopyFiles phase, so building the `LiveWallpaper` scheme is normally all you need.

There are no tests and no linter configured.

Note: the README's CMake build instructions apply to the legacy `ObjectiveC` branch. This branch (`SwiftUI`, the default) has no CMakeLists.txt — use Xcode.

To debug a running install, launch the binary directly to see logs: `/Applications/LiveWallpaper.app/Contents/MacOS/LiveWallpaper`.

## Architecture

Mixed Swift / Objective-C++ codebase. Swift sees the ObjC++ layer through `LiveWallpaper-Bridging-Header.h` (exposes `WallpaperEngine`, `DisplayObjc`, `SharedConstants.h`).

**Two-process design:**

- **App process** (`LiveWallpaperApp.swift` + `ContentView.swift` + `WallpaperEngine.mm`): menu-bar accessory app (`NSStatusItem`, activation policy `.accessory`). `AppDelegate` hosts `ContentView` in an `NSWindow`. All SwiftUI UI lives in `ContentView.swift` (~1000 lines: `ContentView`, `VideoGridView`, `SettingsView`, `WallpaperViewModel`, `ThumbnailCache`, `LanguageManager`, `UserDefaultsKeys`).
- **Daemon process** (`wallpaperdaemon/daemon.mm`): standalone binary, one instance per display. Creates a borderless window at desktop level playing the video with `AVQueuePlayer`/`AVPlayerLooper`. Handles auto-pause on battery/low-power/screen-lock/occlusion and display reconfiguration. CLI: `wallpaperdaemon <video> <frame_output.png> <volume> <scale_mode> [display_uuid]`.

**Engine → daemon lifecycle:** `WallpaperEngine.mm` (ObjC++ singleton, `WallpaperEngine.shared()` from Swift) spawns one daemon per display with `posix_spawn`, tracks PIDs, and kills/replaces daemons when the wallpaper changes. Runtime settings changes are pushed to daemons via Darwin notifications named `com.live.wallpaper.*` (volumeChanged, autoPauseChanged, spaceChanged, terminate, scaleModeChanged) — the daemon registers observers in its `main()`.

**Shared display code:** `DisplayManager.h` is a header-only C++/ObjC++ layer included by both the app and the daemon: the `Display` struct, display UUID ↔ `CGDirectDisplayID` conversion (displays are identified by UUID so they survive reconnection), `ScanDisplays()`, `KillProcessByPID()`. It uses the private `CoreDisplay_DisplayCreateInfoDictionary` API via `dlopen` for display names. `DisplayObjc` is the ObjC wrapper that exposes `Display` to Swift.

**Persistence:**
- Settings: `NSUserDefaults`. Swift-side keys are centralized in the `UserDefaultsKeys` enum in `ContentView.swift`; the ObjC++ side also uses raw string keys (`WallpaperFolder`, `wallpapervolume`, `scale_mode`) — keep them in sync when touching either.
- Per-display wallpaper assignments: `SaveSystem.mm` serializes the display list to `~/Library/Preferences/LiveWallpaper.yaml` using yaml-cpp (the reason the submodule is compiled into the app target).
- Wallpaper folder defaults to `~/Library/Caches/LiveWallpaper`; the engine also maintains thumbnail and static-wallpaper caches (`WallpaperEngine` `thumbnailCachePath` / `staticWallpaperCachePath`).

**Constraints baked into the code:**
- Video filenames must contain exactly one dot (the extension one); only `.mp4` and `.mov` are supported.
- Thumbnail dimensions are defined once in `SharedConstants.h` and used from both Swift and ObjC.
- Localization: `en.lproj` and `zh-Hans.lproj`, switched at runtime by `LanguageManager` in `ContentView.swift`. User-facing strings go through `NSLocalizedString` / `localizedString(_:)`.

**Dead code:** `LiveWallpaper.mm` (~3000 lines, has its own `main()`) is the old pre-SwiftUI app. It is not referenced by any Xcode target — do not add features there. The `ObjectiveC` branch holds that legacy version.
