# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project context

SensibleSideButtons is a macOS menu bar utility that remaps the M4/M5 (side) buttons on third-party mice to fire 3-finger swipe gestures, so they trigger system-wide history navigation instead of being ignored by macOS. Original author is Alexei Baboulevitch (archagon); `upstream` remote points to his repo, `origin` is the `samalone` fork.

The codebase has been untouched since 2018 (deployment target macOS 10.10/10.12, Objective-C only, no ARM slice). The motivation for working on it is to modernize the build (Apple Silicon native, current SDK/deployment target, current code-signing/notarization requirements) so macOS will keep running it.

## Build & run

There is no command-line build script — open `SwipeSimulator.xcodeproj` in Xcode. The project name is historical; the main app target is **SensibleSideButtons** (the `swipesiml` / `swipesimr` targets are tiny standalone CLI test binaries that fire a single left/right swipe via `LEFT` preprocessor define).

Build from the command line with:

```
xcodebuild -project SwipeSimulator.xcodeproj -target SensibleSideButtons -configuration Debug
xcodebuild -project SwipeSimulator.xcodeproj -target SensibleSideButtons -configuration Release
```

Current build settings worth knowing before changing them:
- `DEVELOPMENT_TEAM = R4MX2B96J2` — this is archagon's team ID and will need to be replaced (or removed for ad-hoc signing) before the app will build/sign on this machine.
- `MACOSX_DEPLOYMENT_TARGET = 10.10` for the app, `10.12` for the swipe CLI tools.
- No `ARCHS` override → built with the SDK default, which on a current Xcode is arm64 + x86_64. The 2018 release binary on disk is x86_64-only because it was built with an older Xcode.

The app must be granted Accessibility permission in System Settings → Privacy & Security to receive global mouse events; without it the menu enters `MenuModeAccessibility` and disables the toggles. After replacing the binary, macOS often keeps the *old* path in the Accessibility list — you usually need to remove and re-add the new build.

Runtime defaults live under `net.archagon.sensible-side-buttons`:
```
defaults read net.archagon.sensible-side-buttons
defaults delete net.archagon.sensible-side-buttons   # reset state during testing
```

## Architecture

Three targets, one shared private SPI header set.

**SensibleSideButtons** (`SideButtonFixer/`) is the actual app — an `LSUIElement` menu-bar-only app with no main window despite the storyboard. `AppDelegate.m` is the whole app and does three things:

1. Installs a `CGEventTap` on `kCGHIDEventTap` for `kCGEventOtherMouseDown`/`Up` (`startTap:`). The callback `SBFMouseCallback` inspects `kCGMouseEventButtonNumber` — button 3 → left swipe, button 4 → right swipe (swappable via `SBFSwapButtons`), gated on mouse-down vs. mouse-up by `SBFMouseDown`. Returning `NULL` from the callback swallows the original button event so apps never see it.
2. Synthesizes a 3-finger swipe via `tl_CGEventCreateFromGesture` (see below). Each swipe is two events: a `kTLInfoSubtypeSwipe` with `phase=1` (begin) followed by one with `phase=4` (end) and a direction. These are posted with `CGEventPost(kCGHIDEventTap, …)`.
3. Drives a status-bar menu that has three modes (`MenuModeAccessibility` / `MenuModeDonation` / `MenuModeNormal`) selected by `updateMenuMode` based on `AXIsProcessTrustedWithOptions` and the `SBFDonated` default. `refreshSettings` re-applies item state/visibility every time; menu items are indexed by the `MenuItem` enum and `assert`-checked at construction, so **adding/removing/reordering menu items requires updating the enum in lockstep**.

User defaults keys: `SBFWasEnabled`, `SBFMouseDown`, `SBFSwapButtons`, `SBFDonated`.

**External/TouchEvents.{h,c}** is third-party (Calf Trail Software, 2010) and exposes `tl_CGEventCreateFromGesture(info, touches)` plus the `kTLInfoKey…` / `kTLInfoSubtype…` constants. This is the load-bearing private-SPI shim that lets the app synthesize multitouch gesture events — there is no public API for this. If a future macOS breaks the app, this file is the most likely failure point and the place to look first.

**SwipeSimulator** (`SwipeSimulator/main.m`) is a debugging CLI. It builds twice, once as `swipesiml` (with `LEFT` defined → fires left swipe) and once as `swipesimr` (right swipe), so you can test the gesture synthesis pipeline in isolation from the event tap.

## Working in this repo

- Pure Objective-C with manual `CFRetain`/`CFRelease` around the Core Foundation event objects; ARC is on for Objective-C objects but CF lifetimes are managed by hand.
- `techsteps.txt` is archagon's personal release-notes file (archive → Developer ID export → DMG repack → `codesign --verify --deep --strict`). Useful as a reference but pre-notarization; modern releases need `notarytool` + stapling, which isn't documented anywhere in the repo yet.
