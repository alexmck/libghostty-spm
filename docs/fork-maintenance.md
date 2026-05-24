# Termside Fork Maintenance

This fork exists for Termside packaging stability.

Upstream `Lakr233/libghostty-spm` currently ships the macOS `libghostty.framework` inside `GhosttyKit.xcframework` as a shallow framework:

```text
libghostty.framework/
  Info.plist
  libghostty
  Headers/
  Modules/
```

That layout is valid for iOS-style shallow frameworks, but Xcode's macOS app validation expects a versioned macOS framework:

```text
libghostty.framework/
  Versions/
    A/
      libghostty
      Resources/
        Info.plist
      Headers/
      Modules/
    Current -> A
  libghostty -> Versions/Current/libghostty
  Headers -> Versions/Current/Headers
  Modules -> Versions/Current/Modules
  Resources -> Versions/Current/Resources
```

Without this layout, macOS app builds can fail with:

```text
Framework .../libghostty.framework contains Info.plist, expected Versions/Current/Resources/Info.plist since the platform does not use shallow bundles
```

## Fork Policy

- Keep the fork as close to upstream as possible.
- Prefer packaging-only changes over Swift API changes.
- Keep iOS and Mac Catalyst framework slices shallow.
- Keep macOS framework slices versioned.
- Use exact tags from Termside rather than floating package versions.

## Current Change

`Script/merge-xcframework.sh` stages the macOS framework slice using `Versions/A` and root symlinks before creating `GhosttyKit.xcframework`.

The iOS and Mac Catalyst slices still use the original shallow framework layout.

## Release Tag Convention

Use tags of the form:

```text
<upstream-version>-termside.<n>
```

Example:

```text
1.1.6-termside.1
```

## Release Process

1. Fetch upstream changes.

```sh
git remote add upstream https://github.com/Lakr233/libghostty-spm.git # only needed once
git fetch upstream --tags
```

2. Merge or rebase upstream into the fork branch.

```sh
git checkout main
git merge upstream/main
```

3. Build a fresh XCFramework when the local Ghostty/Zig toolchain is working.

```sh
./build.sh --skip-tests
```

4. If rebuilding is not practical, repack the upstream release artifact only when the Swift sources and binary artifact are otherwise unchanged.

```sh
curl -L -o build/upstream-GhosttyKit.xcframework.zip \
  https://github.com/Lakr233/libghostty-spm/releases/download/<upstream-storage-tag>/GhosttyKit.xcframework.zip

rm -rf build/repack
mkdir -p build/repack
ditto -x -k build/upstream-GhosttyKit.xcframework.zip build/repack
```

5. Normalize only the macOS framework slice if repacking manually.

```sh
framework="build/repack/GhosttyKit.xcframework/macos-arm64_x86_64/libghostty.framework"
mkdir -p "$framework/Versions/A/Resources"
mv "$framework/libghostty" "$framework/Versions/A/libghostty"
mv "$framework/Info.plist" "$framework/Versions/A/Resources/Info.plist"
mv "$framework/Headers" "$framework/Versions/A/Headers"
mv "$framework/Modules" "$framework/Versions/A/Modules"
ln -s "A" "$framework/Versions/Current"
ln -s "Versions/Current/libghostty" "$framework/libghostty"
ln -s "Versions/Current/Headers" "$framework/Headers"
ln -s "Versions/Current/Modules" "$framework/Modules"
ln -s "Versions/Current/Resources" "$framework/Resources"
```

6. Create the release zip.

```sh
ditto -c -k --sequesterRsrc --keepParent \
  build/repack/GhosttyKit.xcframework \
  build/GhosttyKit.xcframework.zip
```

7. Compute the SwiftPM checksum.

```sh
swift package compute-checksum build/GhosttyKit.xcframework.zip
```

8. Update `Package.swift` to point at the fork release asset and checksum.

```swift
.binaryTarget(
    name: "libghostty",
    url: "https://github.com/alexmck/libghostty-spm/releases/download/<tag>/GhosttyKit.xcframework.zip",
    checksum: "<checksum>"
)
```

9. Validate the manifest.

```sh
swift package dump-package
```

10. Commit the changes.

```sh
git add Package.swift Script/merge-xcframework.sh docs/fork-maintenance.md README.md
git commit -m "Prepare Termside libghostty artifact <tag>"
```

11. Tag and push.

```sh
git tag -a "<tag>" -m "Termside libghostty artifact <tag>"
git push origin main
git push origin "<tag>"
```

12. Create the GitHub release and upload the zip.

```sh
gh release create "<tag>" build/GhosttyKit.xcframework.zip \
  --repo alexmck/libghostty-spm \
  --title "<tag>" \
  --notes "Termside rebuild of the upstream binary artifact with the macOS libghostty.framework packaged as a versioned framework for Xcode validation."
```

13. Update Termside's `project.yml` to the new exact fork tag.

```yaml
libghostty-spm:
  url: https://github.com/alexmck/libghostty-spm.git
  exactVersion: <tag>
```

14. Regenerate and verify Termside.

```sh
xcodegen generate
xcodebuild -project Termside.xcodeproj -scheme Termside -destination 'platform=macOS' build
xcodebuild -project Termside.xcodeproj -scheme Termside -configuration Release -destination 'platform=macOS' clean build
codesign --verify --deep --strict --verbose=2 <path-to-release-Termside.app>
```

## Public Fork Notes

This fork can be public, but every release should preserve clear provenance:

- Keep upstream license files intact.
- Do not remove notices from Ghostty, libghostty-spm, or bundled third-party code.
- State in release notes when an artifact is repacked from an upstream binary rather than rebuilt from source.
- Do not publish private signing identities, credentials, or local paths in release notes or scripts.
- Keep the binary artifact source URL and checksum reproducible in release notes when practical.

The fork currently publishes the same wrapper source with a packaging fix for the macOS binary framework layout.
