# PDFium for iOS 15 (Notavus)

This repository holds the build scripts of
[bblanchon/pdfium-binaries](https://github.com/bblanchon/pdfium-binaries) at the commit
that built its release [`chromium/8066`](https://github.com/bblanchon/pdfium-binaries/releases/tag/chromium%2F8066)
(`f2e9a1c`), with two changes for the iOS packages of [Notavus](https://notavus.com), an
app whose minimum is iOS 15. The packages of bblanchon require iOS 17.

1. **iOS 15 as the minimum.** `steps/05-configure.sh` sets `ios_deployment_target = "15.0"`
   instead of `"17.0"` (builds without V8). Builds without V8 link only Foundation,
   CoreFoundation, CoreGraphics and libSystem.
2. **Apple's linker.** The SDKs of Xcode 27 list `arm64e.x1` targets in their stubs
   (`.tbd`), which the lld of chromium/8066 cannot read ("unknown target"): the iOS device
   SDK for the library, the macOS SDK for host tools such as nasm of the x64 simulator.
   iOS builds therefore link with Apple's linker: `use_lld = false` in
   `steps/05-configure.sh` and, for the host toolchain, which Chromium sets to lld,
   `patches/ios/build.patch` (applied by `steps/03-patch.sh`).

Nothing else is changed. The packages have the same layout, headers, exported symbols,
`LICENSE`, `licenses/`, `VERSION` and `PDFiumConfig.cmake` as those of bblanchon's release
`chromium/8066`; `args.gn` differs in `ios_deployment_target` and `use_lld`.

## Releases

| Release | PDFium | Packages |
|---|---|---|
| [`chromium/8066-ios15`](https://github.com/Schwarzbierbaron/notavus-pdfium/releases/tag/chromium%2F8066-ios15) | 156.0.8066.0, branch `chromium/8066` (`fc46361c`) | `pdfium-ios-device-arm64.tgz`, `pdfium-ios-simulator-arm64.tgz`, `pdfium-ios-simulator-x64.tgz` |

SHA-256 of `chromium/8066-ios15`:

```
d4d18fabae4a16cc93a9360f2f3cdf597057b01dcdacb54dc9e82d089335fc46  pdfium-ios-device-arm64.tgz
7ce8fba070008efe34ebca35c017b8a516d7f5be4d8118b075fe716106da230f  pdfium-ios-simulator-arm64.tgz
844dade84a24cb07b2b9fefa1cef29ec8fa5a5100e653a680e3653570efdb6af  pdfium-ios-simulator-x64.tgz
```

## Building

Built on a Mac mini (Apple M6, macOS 27.0.1) with Xcode 27.1 (iOS SDK 27.1) from the tag
of the release:

```bash
export PDFium_VERSION=156.0.8066.0
steps/00-environment.sh
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git depot_tools
echo "$PWD/depot_tools" > .path
./build.sh -b chromium/8066 -g 2 ios arm64 device
./build.sh -b chromium/8066 -g 5 ios arm64 simulator
./build.sh -b chromium/8066 -g 5 ios x64 simulator
```

Steps 0 and 1 run by hand because `steps/01-install.sh` selects
`/Applications/Xcode_26.0.app` of GitHub's runners with sudo; the build uses the selected
Xcode. The second and third build start at the configure step: checkout and patches are
the same for every iOS target. Each run writes `pdfium-ios-<environment>-<cpu>.tgz`.

## Checks

Before a release, for each package:

- `vtool -show-build lib/libpdfium.dylib`: `minos 15.0`, platform `IOS` or
  `IOSSIMULATOR`.
- `nm -gU`: the same exported symbols as bblanchon's package of the same branch (624, of
  them 432 `FPDF*`); `nm -um`: the same imported symbols (Apple's linker needs no
  `dyld_stub_binder`), none of them weak; `otool -L`: Foundation, CoreFoundation,
  CoreGraphics and libSystem.
- The build log has no availability warning (`-Wunguarded-availability` is on).

## Licenses

The scripts are under the MIT license of bblanchon/pdfium-binaries (`LICENSE`). Each
package carries the license of PDFium and those of its components under `licenses/`,
collected as upstream by `steps/08-licenses.sh`. HarfBuzz (MIT-Modern-Variant) and
dragonbox (Apache-2.0 with LLVM exception or BSL-1.0) are compiled into the library too,
but that step does not collect their texts (as upstream); they are
`third_party/harfbuzz/src/COPYING` and `third_party/dragonbox/src/LICENSE-*` of PDFium's
checkout.
