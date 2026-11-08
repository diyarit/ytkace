# YTKACE handoff — 2026-11-08

This document records the state after the iPhone issue pass and the Open in YouTube Safari extension integration. It is meant to let the next person (or agent) resume without re-reading the whole session.

## Current state

- Repository: `https://github.com/diyarit/ytkace`
- Branch: `main`
- Source-fix commit: `224486f` — *Add Open in YouTube extension and iPhone fixes*
- Build: YTKACE 1.1.5, arm64, Theos
- `origin` still points at `itzzace/ytkace`; the working push remote is `diyarit`.

### Generated IPA

- Path: `/Users/diyarabbas/Desktop/ytkace/YTKACE_1.1.5_YouTube_21.44.4_fixed_20261108_011429.ipa`
- SHA256: `d59b818f8175c0b6ac1421afb8d8a31dd7ccc3cbda6abc8e19e98ec2394ea164`
- Base: `/Users/diyarabbas/Desktop/ytkace/com.google.ios.youtube_v21.44.4_und3fined.ipa`
- Bundle ID: `com.google.ios.youtube`
- YouTube: 21.44.4 (iOS 17+)
- Signing: ad-hoc `ldid` signature suitable for TrollStore; re-sign with Sideloadly/AltStore/SideStore for a developer certificate.

## Changes in this pass

### Open in YouTube Safari extension (#105)

Files:

- `Resources/OpenYouTubeSafariExtension.appex`
- `Scripts/sideload-repack.sh`
- `Scripts/sign-bundle.sh`
- `Tools/verify-artifacts.sh`
- `README.md`
- `THIRD_PARTY_NOTICES.md`
- `docs/ASSET_PROVENANCE.md`

What it does:

- Vendors the prebuilt appex from `BillyCurtis/OpenYouTubeSafariExtension` at upstream commit `c87b5d5`.
- Copies it to `YouTube.app/PlugIns/` in IPA builds. The real YouTube 21.44.4 layout uses `PlugIns` (capital I), not `Plugins`.
- Makes `OpenYouTubeSafariExtension` executable. Upstream stores it as mode `100644`, so without this the extension cannot launch.
- Signs every `.appex` as a bundle after signing its Mach-O files. This keeps the code-signature identifier at `com.google.ios.youtube.OpenYouTube.Extension` instead of the bare executable name, and seals the extension resources.
- `verify-artifacts.sh` fails if the appex `Info.plist` or executable is missing.

User action after install: **Settings > Safari > Extensions > Open in YouTube**.

Licence note: the upstream repository publishes no licence file. Shipping IPAs containing the appex may need the author's permission.

### iPhone-specific fixes

| Issue | Change | Status |
|---|---|---|
| #199 | Brightness uses the player window scene's screen instead of `UIScreen.mainScreen` | Fixed |
| #195 / #204 | Watch Later removed from addable tabs | Mitigated by removing the broken tab |
| #203 | Filter-chip hooks call the original first, then hide; frame list returns `@[]` instead of `nil` | Hardened; needs device confirmation |
| #202 | Legacy quality controller gets a real service scope/parent responder; exceptions fall back to the plain format list | Hardened; needs device confirmation |
| #131 | YTKACE Shorts progress line hidden during pinch fullscreen | Implemented; needs device confirmation |
| #143 | YTKACE Shorts progress line lifted 10 pt above the floating tab bar | Implemented; needs device confirmation |

Key files:

- `Tweak/Features/Gestures/PlayerGestures.mm`
- `Tweak/Features/Downloads/YTKACEDownloadPlayerController.mm`
- `Tweak/Features/Interface/ContentVisibilityHooks.mm`
- `Tweak/Features/Navigation/TabBarHooks.mm`
- `Tweak/Features/Shorts/ShortsHooks.mm`
- `Tweak/Features/Streaming/StreamingHooks.mm`
- `Tweak/Settings/YTKACESettingsPages.mm`
- `Tweak/Settings/YTKACETabEditorController.mm`

### Why Watch Later was removed

Watch Later is a playlist page (`VLWL`), not a browse feed. On YouTube 21.44, adding it as a pivot tab leaves the pivot bar unable to select any tab. Other synthetic browse tabs (History, Playlists, etc.) still work. Removing the entry fixes the bricked tab bar for existing users because the missing token is simply ignored by the order sorter.

Do not re-add it without a way to open the playlist outside the pivot bar. The binary exposes `-[YTAppPivotBarController didTapItemWithPivotIdentifier:browseID:]`, which may be a starting point.

### Already fixed before this pass

These were verified as already fixed on `main` and were not changed:

- #194 Extra Dim breaking rotation — fixed in 1.1.5
- #118 downloaded-video rotate button — added in 1.1.5
- #125 Keep Progress Bar Visible — fixed in 1.1.5; the "Show Status Bar in Fullscreen" half is still unconfirmed
- #140 queue — true local queue landed on `main` after 1.1.5
- #170 open links directly — option added in 1.1.5
- #46 haptics — haptics toggle added in 1.1.5

## Build and package

```bash
cd /Users/diyarabbas/ytkace
export THEOS="$HOME/theos"
make clean all

bash Scripts/sideload-repack.sh \
  /path/to/decrypted/YouTube.ipa \
  dist/YTKACE.dylib \
  /path/to/output/YTKACE_fixed.ipa

bash Tools/verify-artifacts.sh dist/YTKACE.dylib /path/to/output/YTKACE_fixed.ipa
```

The Theos build takes several minutes. `make all` copies the dylib to `dist/YTKACE.dylib`.

## Validation performed

- All changed Objective-C++ files pass `clang -fsyntax-only` against the iOS 27.2 SDK with `-fobjc-arc`, `-std=c++17`, `-Wall -Wextra -Werror=return-type`.
- `python3 -m unittest discover -s Tests` passes (5 tests).
- End-to-end repack of the real YouTube 21.44.4 IPA succeeded.
- `verify-artifacts.sh` passed on the output IPA.
- Output appex checks:
  - executable mode `100755`
  - `codesign` identifier `com.google.ios.youtube.OpenYouTube.Extension`
  - resources sealed
- Output dylib is arm64 and has no CydiaSubstrate/MobileSubstrate dependency.

## Open items

### Needs a device or crash log

- **#205** iPhone Duo landscape: right panel (suggested videos/comments) cannot scroll or jumps to top.
- **#193** Dynamic Island shows the audio waveform while the app is in the foreground.
- **#150** History scrolling is janky on iPhone 17 Pro Max.
- **#167** Dragging down to exit fullscreen moves the video out of the player instead.
- **#128, #159, #100, #83, #133, #138, #139, #162, #196, #122, #129, #130** are glass/layout visual issues that need screenshots and device testing.

### Needs on-device confirmation

- #202 and #203 are crash-path hardening based on the private API behavior; the original reporters did not provide crash logs. If either still crashes, capture the iOS crash log requested in those issues.

### iPad-only

- #177 iPad Home layout
- #135 iPad volume HUD

### Other

- #61 (lock-screen player does not open the app) looks like a sideloading limitation, as noted in the issue comments.
- #60, #31, #64 are cast/network/DRM paths rather than iPhone-only regressions.
- The Safari appex has no upstream licence.

## Repository conventions

- Hooks are installed by name through `Tweak/Runtime/Hooking.h`; a missing class fails quietly.
- New user-facing strings go through `YTKACELocalizer` and `Resources/YTKACE.bundle/en.lproj/Localizable.strings`; other languages can follow later.
- Build IPAs with `Scripts/sideload-repack.sh`; do not hand-edit the app bundle.
- `*.ipa` is ignored and must not be committed.
- `Resources/` contains vendored assets/binaries; `Vendor/FFmpeg` contains static arm64 libraries.
- Git identity is set locally in this clone to `diyarit <diyarit@users.noreply.github.com>`.
- Push to the `diyarit` remote unless told otherwise. Do not force-push `main`.
