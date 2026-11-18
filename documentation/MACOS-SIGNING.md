# macOS signing and the Mac App Store

How Crancula's macOS builds are signed ([ADR 30](adr/0030-macos-builds-signed-and-in-the-app-store.md)) and the
one-time setup the owner of the Apple Developer account does by hand. The release workflow
(`.github/workflows/release.yml`) does the rest.

| Build | Certificate | Where it goes |
|-------|-------------|---------------|
| `.dmg` (job "Package (macOS)") | Developer ID Application, then notarized | GitHub Release |
| `.pkg` (job "Package (Mac App Store)") | Apple Distribution (app) and Mac Installer Distribution (`.pkg`), sandboxed | App Store Connect, uploaded by hand |

Bundle ID: `de.twomartens.crancula` (`appId` in `electron-builder.json`). It never changes once the app is in
App Store Connect.

Without the secrets, a pull request or manual run builds an unsigned `.dmg` and skips the App Store package. A
tag `v*` fails instead, so no release goes out unsigned.

## One-time setup

All on a Mac, signed in to [developer.apple.com](https://developer.apple.com/account) with the account that pays
for the Apple Developer Program.

### 1. Team ID and App ID

1. **Membership details**: note the 10-character **Team ID**.
2. **Certificates, IDs & Profiles → Identifiers → +**: App IDs, type App, Bundle ID *Explicit*
   `de.twomartens.crancula`, description "Crancula". No capabilities needed.

### 2. Certificates

For each certificate: in Keychain Access, *Certificate Assistant → Request a Certificate From a Certificate
Authority…*, save to disk; upload that `.certSigningRequest` under **Certificates → +**; download the `.cer` and
double-click it so it lands in the login keychain next to its private key.

| Type to choose | Used for |
|----------------|----------|
| Developer ID Application | the `.dmg` |
| Apple Distribution | the App Store app |
| Mac Installer Distribution | the App Store `.pkg` |

Export each one as `.p12` with a password: Keychain Access, *My Certificates*, right-click the certificate
(it must show its private key underneath) → *Export*.

### 3. Provisioning profile

**Profiles → +**: *Mac App Store Connect* (Distribution), App ID `de.twomartens.crancula`, certificate Apple
Distribution. Download the `.provisionprofile`. It expires with the certificate; renew both together.

### 4. API key for notarization

[App Store Connect](https://appstoreconnect.apple.com) → **Users and Access → Integrations → App Store Connect
API → Team Keys → +**, access *Developer*. Note the **Key ID** and the **Issuer ID**, download the
`AuthKey_<KeyID>.p8` (only possible once).

### 5. App record

App Store Connect → **Apps → + → New App**: platform macOS, name "Crancula" (or what is free), primary language,
Bundle ID `de.twomartens.crancula`, SKU `crancula`. Fill in the listing before the first submission: description,
screenshots, category Music, privacy policy URL, App Privacy ("Data Not Collected"), age rating, price.

### 6. GitHub secrets

Repository → *Settings → Secrets and variables → Actions → New repository secret*. Encode files with
`base64 -i <file> | pbcopy` and paste.

| Secret | Value |
|--------|-------|
| `MAC_DEVELOPER_ID_P12_BASE64` | Developer ID Application `.p12`, base64 |
| `MAC_DEVELOPER_ID_P12_PASSWORD` | its export password |
| `APPLE_API_KEY_P8` | content of `AuthKey_<KeyID>.p8` (plain text, not base64) |
| `APPLE_API_KEY_ID` | Key ID |
| `APPLE_API_ISSUER` | Issuer ID |
| `MAS_APP_P12_BASE64` | Apple Distribution `.p12`, base64 |
| `MAS_APP_P12_PASSWORD` | its export password |
| `MAS_INSTALLER_P12_BASE64` | Mac Installer Distribution `.p12`, base64 |
| `MAS_INSTALLER_P12_PASSWORD` | its export password |
| `MAS_PROVISIONING_PROFILE_BASE64` | the `.provisionprofile`, base64 |
| `APPLE_TEAM_ID` | Team ID |

Then run the Release workflow by hand (*Actions → Release → Run workflow*) and check that the macOS job logs
`notarization successful` and that the `app-store-package` artifact exists.

## Each release

The `release` skill runs as before. After the workflow:

1. Download the `app-store-package` artifact from the tag's workflow run.
2. Upload the `.pkg` with Apple's **Transporter** app (Mac App Store), signed in with the developer account.
3. In App Store Connect, pick the build for a new version, fill in "What's New" from the changelog section, and
   submit for review.

Every upload needs a version number higher than the last one; the release's version bump takes care of that.

## Building locally

`npm run electron:build` signs the `.dmg` with a Developer ID identity from the login keychain and notarizes it
when `APPLE_API_KEY` (path to the `.p8`), `APPLE_API_KEY_ID` and `APPLE_API_ISSUER` are set; otherwise it
builds unsigned (`CSC_IDENTITY_AUTO_DISCOVERY=false` skips the keychain). An App Store package needs the
provisioning profile at `build/embedded.provisionprofile` (ignored by Git) and `TEAM_ID` replaced in
`build/entitlements.mas.plist` (don't commit that), then
`npx electron-builder --mac mas -c.mas.extendInfo.ElectronTeamID=<TeamID>`.

## Sandbox

The App Store build runs in the App Sandbox with `files.user-selected.read-write` only: files are read and
written through open and save dialogs (ADR 4, 21, 28), snapshots live in `userData` inside the container. A
feature that touches files without a dialog, remembers a folder across launches, or uses the network needs a
new entitlement here and a test of the App Store build.
