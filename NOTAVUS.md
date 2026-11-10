# PDFium for iOS 15 (Notavus)

This repository holds the build scripts of
[bblanchon/pdfium-binaries](https://github.com/bblanchon/pdfium-binaries), merged at the
commit that built its release
[`chromium/8152`](https://github.com/bblanchon/pdfium-binaries/releases/tag/chromium%2F8152)
(`0dc8e95`), with two changes for the iOS packages of [Notavus](https://notavus.com), an
app whose minimum is iOS 15. The packages of bblanchon require iOS 17.

1. **iOS 15 as the minimum.** `steps/05-configure.sh` sets `ios_deployment_target = "15.0"`
   instead of `"17.0"` (builds without V8). Builds without V8 link only Foundation,
   CoreFoundation, CoreGraphics and libSystem.
2. **Apple's linker.** The SDKs of Xcode 27 list `arm64e.x1` targets in their stubs
   (`.tbd`), which the lld of chromium/8066 could not read ("unknown target"): the iOS
   device SDK for the library, the macOS SDK for host tools such as nasm of the x64
   simulator. iOS builds therefore link with Apple's linker: `use_lld = false` in
   `steps/05-configure.sh` and, for the host toolchain, which Chromium sets to lld,
   `patches/ios/build.patch` (applied by `steps/03-patch.sh`).

Nothing else is changed. The packages have the same layout (`lib/libpdfium.dylib` and
`lib/pdfium.framework`), headers, exported symbols, `LICENSE`, `licenses/`, `VERSION` and
`PDFiumConfig.cmake` as those of bblanchon's release of the same branch; `args.gn` differs
in `ios_deployment_target` and `use_lld`, the `Info.plist` of the framework in
`MinimumOSVersion`.

## Releases

| Release | PDFium | Packages |
|---|---|---|
| [`chromium/8152-ios15`](https://github.com/Schwarzbierbaron/notavus-pdfium/releases/tag/chromium%2F8152-ios15) | 159.0.8152.0, branch `chromium/8152` (`6688c576`) | `pdfium-ios-device-arm64.tgz`, `pdfium-ios-simulator-arm64.tgz`, `pdfium-ios-simulator-x64.tgz` |
| [`chromium/8066-ios15`](https://github.com/Schwarzbierbaron/notavus-pdfium/releases/tag/chromium%2F8066-ios15) | 156.0.8066.0, branch `chromium/8066` (`fc46361c`) | `pdfium-ios-device-arm64.tgz`, `pdfium-ios-simulator-arm64.tgz`, `pdfium-ios-simulator-x64.tgz` |

SHA-256 of `chromium/8152-ios15`:

```
6010eb847abd23cc8aef360eb122fead3e81d1f08e13564f3bd5897f429ca201  pdfium-ios-device-arm64.tgz
5636521fb512006566b5dd232aa5ec5d36258e53e933a96e1305d766e55e5898  pdfium-ios-simulator-arm64.tgz
98bb3acbad06d4c6646a3248fe7f02f586e62240ca3c5f1e4a8dbdc6a576ebda  pdfium-ios-simulator-x64.tgz
```

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
export PDFium_VERSION=159.0.8152.0
export PATH="/opt/homebrew/opt/gettext/bin:/opt/homebrew/bin:$PATH"
steps/00-environment.sh
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git depot_tools
echo "$PWD/depot_tools" > .path
./build.sh -b chromium/8152 -g 2 ios arm64 device
./build.sh -b chromium/8152 -g 5 ios arm64 simulator
./build.sh -b chromium/8152 -g 5 ios x64 simulator
```

Steps 0 and 1 run by hand because `steps/01-install.sh` selects
`/Applications/Xcode_26.0.app` of GitHub's runners with sudo; the build uses the selected
Xcode. `steps/07-stage.sh` writes the `Info.plist` of the framework with `envsubst`
(gettext, keg-only in Homebrew) and `steps/09-test.sh` builds the example with CMake, hence
the `PATH`. The second and third build start at the configure step: checkout and patches
are the same for every iOS target. Each run writes `pdfium-ios-<environment>-<cpu>.tgz`.

## Checks

Before a release, for each package:

- `vtool -show-build lib/libpdfium.dylib`: `minos 15.0`, platform `IOS` or
  `IOSSIMULATOR`; `MinimumOSVersion` 15.0 in `lib/pdfium.framework/Info.plist`.
- `nm -gU`: the same exported symbols as bblanchon's package of the same branch (625 for
  chromium/8152, of them 433 `FPDF*`); `nm -um`: the same imported symbols (Apple's linker
  needs no `dyld_stub_binder`), none of them weak; `otool -L`: Foundation, CoreFoundation,
  CoreGraphics and libSystem.
- Headers and `licenses/` byte for byte as in bblanchon's package.
- The build log has no availability warning (`-Wunguarded-availability` is on).

## Licenses

The scripts are under the MIT license of bblanchon/pdfium-binaries (`LICENSE`). Each
package carries the license of PDFium and those of its components under `licenses/`,
collected as upstream by `steps/08-licenses.sh`. HarfBuzz (MIT-Modern-Variant) and
dragonbox (Apache-2.0 with LLVM exception or BSL-1.0) are compiled into the library too,
but that step does not collect their texts (as upstream); they are
`third_party/harfbuzz/src/COPYING` and `third_party/dragonbox/src/LICENSE-*` of PDFium's
checkout.
