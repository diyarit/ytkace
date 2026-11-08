# AGENTS.md

Persistent project memory for YTKACE. Read this before changing the tweak or packaging an IPA.

## Project

- iOS YouTube enhancement (Theos tweak + sideloaded IPA build).
- Source: `https://github.com/diyarit/ytkace` (`main`).
- Current fix commit: `224486f` — "Add Open in YouTube extension and iPhone fixes".
- Full session details and issue triage: [`docs/HANDOFF.md`](docs/HANDOFF.md).
- `origin` still points at `itzzace/ytkace`. The working push remote is `diyarit`; push there unless told otherwise. Never force-push `main`.

## Build

```bash
export THEOS="$HOME/theos"
make clean all
```

- Output: `dist/YTKACE.dylib` (arm64).
- Uses the local Xcode iOS SDK and the static FFmpeg libs in `Vendor/FFmpeg/lib`.
- `make package` is not needed for IPA work.
- Syntax-check changed Objective-C++ files with:

```bash
SDK=$(xcrun --sdk iphoneos --show-sdk-path)
xcrun --sdk iphoneos clang++ -fsyntax-only -fobjc-arc -std=c++17 \
  -Wall -Wextra -Werror=return-type -Wno-unused-parameter \
  -Wno-module-import-in-extern-c -isysroot "$SDK" \
  -target arm64-apple-ios15.0 -I Tweak <file.mm>
```

## Package an IPA

```bash
bash Scripts/sideload-repack.sh <input.ipa> dist/YTKACE.dylib <output.ipa>
bash Tools/verify-artifacts.sh dist/YTKACE.dylib <output.ipa>
```

- The output IPA is ad-hoc signed for TrollStore; Sideloadly/AltStore/SideStore can re-sign it.
- `*.ipa` is gitignored. Do not commit generated IPAs.
- The last validated IPA was based on YouTube 21.44.4 (`com.google.ios.youtube`), output on the Desktop, SHA256 `d59b818f8175c0b6ac1421afb8d8a31dd7ccc3cbda6abc8e19e98ec2394ea164`. Regenerate rather than relying on that path.

## Important implementation notes

- **`PlugIns`, not `Plugins`**: YouTube's app extension directory is `Payload/YouTube.app/PlugIns`. Use that casing when adding appexes or build scripts.
- **Safari extension**: `Resources/OpenYouTubeSafariExtension.appex` is vendored from `BillyCurtis/OpenYouTubeSafariExtension` at commit `c87b5d5`.
  - Its executable must be mode `100755`; upstream stores it `100644`. `sideload-repack.sh` chmods it.
  - `sign-bundle.sh` signs `.appex` bundles as units so the code-signature identifier is `com.google.ios.youtube.OpenYouTube.Extension` and resources are sealed.
  - The upstream repo has no licence file. Publishing IPAs containing it may need permission.
- **Hooks** are installed by name through `Tweak/Runtime/Hooking.h`. Missing classes/selectors fail quietly; do not introduce a hooking framework.
- **Watch Later was removed from addable tabs** (issues #195/#204). It is a playlist page and freezes the pivot bar on YouTube 21.44. Do not re-add it without opening the playlist outside the pivot bar.
- **Legacy quality path** (#202): `YTVideoQualitySwitchRedesignedController` must get a real service registry scope and parent responder; nil values are an error case. Keep the exception fallback.
- **Filter chips** (#203): call the original chip-bar/subheader hooks before hiding, and return an empty array from `framesForChipViewsWithWidth:` when hiding.
- **Duo brightness** (#199): use `view.window.windowScene.screen`, not `UIScreen.mainScreen`.
- **Shorts line** (#131/#143): hide YTKACE's own progress layer in pinch fullscreen and keep it above the floating tab bar otherwise.
- **Localization**: user-facing strings go through `YTKACELocalizer` and `Resources/YTKACE.bundle/en.lproj/Localizable.strings`; other languages can follow.

## Validation gates

- `python3 -m unittest discover -s Tests` (currently 5 tests).
- `git diff --check`.
- `bash -n Scripts/sideload-repack.sh Scripts/sign-bundle.sh Tools/verify-artifacts.sh`.
- End-to-end repack + `verify-artifacts.sh` before delivering an IPA.

## Known unverified / open

- #202, #203, #131, #143 are code-hardened but were not tested on a physical device.
- #205 (iPhone Duo landscape scroll), #193 (Dynamic Island), #150 (History lag), #167 (fullscreen dismiss drag), and the glass/layout issues need a device or screenshots.
- iPad-only: #177, #135.
- #61 looks like a sideloading limitation.
