# Xcode signing and private-key recovery (637)

This workflow is for local testing of a modified WeType 2.2.1 (637) input method.
It does not recreate the original Tencent Developer ID signature and it does not
make a free Personal Team eligible for distribution or notarization.

## What is required

- macOS with Xcode installed and accepted license.
- The operator's Apple Account signed in to Xcode under Xcode > Settings >
  Accounts.
- An `Apple Development` identity whose certificate and private key are paired
  in the login keychain.
- A Team ID belonging to that account. Do not infer it from the short code in
  parentheses after the certificate name.

An Apple Account is enough for a local Xcode Personal Team test identity; a
paid Apple Developer Program membership is required for Developer ID signing,
distribution, and notarization. The macOS Passwords app is not the certificate
store. Certificates and private keys are managed by Keychain Access, Xcode, and
the `security` command.

## Inspect the current machine

```bash
/usr/bin/xcode-select -p
/usr/bin/xcodebuild -version
/usr/bin/security find-identity -v -p codesigning
/usr/bin/security find-certificate -a -c "Apple Development" \
  "$HOME/Library/Keychains/login.keychain-db"
```

If `/Applications/Xcode.app` is missing, install the full Xcode application,
launch it once, accept its license, and sign in under Xcode > Settings >
Accounts. Command Line Tools alone cannot create the automatic-signing project.

After Xcode is installed, select its developer directory:

```bash
sudo /usr/bin/xcode-select --switch \
  /Applications/Xcode.app/Contents/Developer
/usr/bin/xcodebuild -version
```

The `sudo` command and any Apple Account, Keychain, or certificate prompts must
be completed interactively by the operator. Do not put those credentials in a
script or repository.

The important distinction is:

```text
certificate visible in Keychain + private key paired = usable identity
certificate visible without private key = cannot sign
```

`security find-identity` must report at least one valid codesigning identity.
Use the printed SHA-1 or complete common name as the `IDENTITY` argument to the
generic signer. A certificate name such as `Apple Development: person@example`
may show a short code in parentheses; do not treat that display as authoritative
Team ID metadata. Read the Team ID from Xcode's team selector or, after a sample
app builds successfully, from its `TeamIdentifier` in `codesign -dv` output.

## Recover a missing private key with Xcode

When Xcode shows a development certificate but `find-identity` reports zero
valid identities, create a minimal macOS app and let Xcode perform automatic
signing. This recovery path has been validated for 637 workflow: the sample
build creates a paired identity, exposes its Team ID, and the same identity can
sign the patched bundle successfully.

Create a temporary project with XcodeGen if it is installed. Replace
`TEAM_ID_HERE` with the operator's own Team ID; never copy a friend's value.

```yaml
name: SigningRepair
options:
  bundleIdPrefix: com.example
settings:
  base:
    CODE_SIGN_STYLE: Automatic
    DEVELOPMENT_TEAM: TEAM_ID_HERE
targets:
  SigningRepair:
    type: application
    platform: macOS
    deploymentTarget: "14.0"
    sources:
      - Sources
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.example.signingrepair
        GENERATE_INFOPLIST_FILE: YES
        SWIFT_VERSION: 5.0
```

Add a minimal SwiftUI app at `Sources/SigningRepairApp.swift`:

```swift
import SwiftUI

@main
struct SigningRepairApp: App {
    var body: some Scene {
        WindowGroup {
            Text("Signing repair")
                .padding(32)
        }
    }
}
```

Then build it while signed in to Xcode:

```bash
xcodegen generate
xcodebuild -project SigningRepair.xcodeproj \
  -scheme SigningRepair \
  -configuration Debug \
  -destination 'platform=macOS' \
  -derivedDataPath DerivedData \
  -allowProvisioningUpdates \
  build
```

If XcodeGen is unavailable, create the same macOS App project in Xcode, enable
Automatically manage signing, choose the Personal Team, and build once.

Recheck the keychain:

```bash
/usr/bin/security find-identity -v -p codesigning
/usr/bin/codesign -dv --verbose=4 \
  DerivedData/Build/Products/Debug/SigningRepair.app 2>&1 \
  | /usr/bin/grep -E 'Authority|TeamIdentifier|Identifier'
```

## Exporting an identity from another Mac

Only the original owner should export a signing identity, and the export must
include the private key. A bare `.cer` is not enough. Xcode's certificate
export or Keychain Access can create a password-protected `.p12` file.

Import it on the target Mac, then verify with `security find-identity`. Never
put a `.p12`, password, private key, or Keychain database in the shareable
skill. Never ask a friend to use another person's private key.

## Entitlements

Signing a nested app with the wrong entitlements can make input work in one
host and fail in another. Use the generic `create_entitlements.sh` script from
the work directory to generate the known 637 local-test set with the current
Team ID. The two helper executables with application identifiers must be
prefixed with that Team ID:

```text
TEAM_ID.com.tencent.WeTypeRelaunch
TEAM_ID.com.tencent.WeTypeAccessibilityChecker
```

The 637 `Contents/Helpers/WeTypeUpdater.app` has no custom entitlement blob in
the official package. Sign that app with hardened runtime and no entitlement
file.

Keep the app-specific capabilities from the 637 package:

- WeType: non-sandbox, Apple Events, audio input, Downloads read/write,
  network client/server.
- Settings: non-sandbox, audio input.
- Feedback: non-sandbox, user-selected read-only.
- FinderSync: sandboxed, user-selected read-only.

Inspect the result before signing:

```bash
/usr/bin/plutil -p "$ENTITLEMENTS/wetype.plist"
/usr/bin/plutil -p "$ENTITLEMENTS/settings.plist"
```

Do not blindly copy entitlements from an unrelated app. Do not add
`get-task-allow` to a production-like local bundle unless you explicitly need
debugging and understand the trust impact.

## Correct signing order

Sign the deepest code first and the containing bundle last:

1. Settings `App.framework` and `FlutterMacOS.framework`, then Settings.app.
2. Feedback Swift runtime, then Feedback.app.
3. Sparkle Autoupdate, Updater, Downloader, Installer, then Sparkle.framework.
4. Root frameworks, Swift runtime, and the `Contents/Resources/BinaryDelta`
   executable leaf.
5. WeTypeRelaunch, `Contents/Helpers/WeTypeUpdater.app`, and the accessibility
   checker.
6. FinderSync extension.
7. Top-level WeType.app.

Use `--timestamp=none` for a local test signature. Use `codesign --verify
--deep --strict` only for verification; `codesign --deep` is not a substitute
for an explicit inside-out signing order.

## Common signing failures (637)

| Symptom | Likely cause | Correct response |
| --- | --- | --- |
| `0 valid identities found` | certificate has no paired private key | recover through Xcode automatic signing or import the original `.p12` |
| `errSecInternalComponent` | keychain/private-key access or wrong identity | unlock login keychain, verify identity, avoid ad-hoc signing |
| top-level signature valid but deep verify fails | nested component was not signed after its parent changed | rerun inside-out signer |
| input works in TextEdit only | ad-hoc/trust/entitlement boundary or duplicate TIS row | inspect signature and TIS before changing data |
| `spctl` rejects a local development bundle | Personal Team is not Developer ID | treat as local test only; do not claim notarized distribution |
| `Access Unavailable` on developer portal | Personal Team not enrolled in paid program | use Xcode local signing or enroll separately; portal certificates are not required for this local test |

## 637-specific notes

The work directory at `/Users/fitz/Documents/202607/wetype-637-work/` contains:

- `signing-entitlements/` - Pre-configured entitlements for Team ID
- `signing-entitlements-c4w6c3s5p4/` - Team-specific entitlements backup
- `white-selected-black-text-tools/` - Full toolchain including
  `create_entitlements.sh` and `sign_wetype_637.sh`

Use these existing scripts rather than creating new ones. Verify the Team ID
matches before signing.
