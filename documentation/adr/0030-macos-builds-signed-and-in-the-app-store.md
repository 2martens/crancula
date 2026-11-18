# 30. macOS builds signed and notarized; a sandboxed package for the Mac App Store

Date: 2026-11-18

## Status

Accepted

## Context

The release workflow (#11, ADR 19) built an unsigned `.dmg`. Gatekeeper reported it as damaged, and the README
told owners to run `xattr -cr`. The owner has an Apple Developer Program membership and wants Crancula both as a
download and in the Mac App Store.

The two routes need different builds. A download needs a Developer ID signature, the hardened runtime and
notarization. The App Store does not accept a `.dmg`: it needs electron-builder's `mas` target, Electron's MAS
build, the App Sandbox, Apple Distribution and Mac Installer Distribution certificates, a provisioning profile and
App Review.

## Decision

- Bundle ID `de.twomartens.crancula`, App Store category Music.
- The `.dmg` is signed with Developer ID and notarized with an App Store Connect API key, in the existing macOS
  packaging job. It stays on GitHub Releases.
- A separate job builds the `mas` target into a signed `.pkg`, kept as a workflow artifact and uploaded to App
  Store Connect by hand with Transporter. Submission stays a human step.
- The App Store build gets only `app-sandbox`, its application group and `files.user-selected.read-write`;
  Crancula touches files only through dialogs and `userData`.
- Certificates, the API key, the provisioning profile and the Team ID live in GitHub secrets; the Team ID is put
  into the entitlements at build time. Without the secrets a pull request or manual run builds unsigned and skips
  the App Store package; a tag fails.
- Windows and Linux stay unsigned.

The setup steps are in [MACOS-SIGNING.md](../MACOS-SIGNING.md).

## Consequences

- macOS owners install without a Gatekeeper warning or `xattr`.
- Releases need the eleven secrets; certificates and the provisioning profile expire and must be renewed.
- Each release adds a manual Transporter upload and App Review; the release is published on GitHub before Apple
  approves it.
- A feature that needs files without a dialog, a remembered folder or the network must add an entitlement and
  be checked in the App Store build.
- The App Store build is Apple silicon only, like the `.dmg`.
