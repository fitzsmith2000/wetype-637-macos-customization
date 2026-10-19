---
name: wetype-637-macos-customization
description: "Use when WeType 637 color, V-panel, or settings patches."
metadata:
  version: "1.1.0"
---

# WeType 637 color customization

Version-locked color work on a user-owned WeType 2.2.1 (637) macOS bundle.
This skill keeps the official bundle untouched, patches a work copy, signs
every nested component from the inside out, verifies the result, and only then
installs it as a macOS input method.

## Compatibility

| Skill version | WeType version | Support level |
| --- | --- | --- |
| v1.1.0 | 2.2.1 (637) | Fully validated color patches, build, sign, install, rollback, and input-source recovery |
| v1.1.0 | Other or later builds | Workflow is reusable, but the 637 binary offsets and resource counts are unverified |

A later WeType build may retain enough of the same layout for this workflow to
apply, but native offsets, Flutter AOT instructions, asset records, signing
targets, or entitlements can change. The bundled 637 scripts intentionally
stop on a different version or unexpected bytes; do not remove those guards.
Re-derive a compatibility profile when they fail.

## Ownership and sharing boundary

- Use a legally obtained official WeType package and a machine/account the
  operator is authorized to modify.
- Share this skill, patch source, offset map, and generic scripts. Do not share
  Tencent's app binary, user dictionaries, private keys, exported `.p12`
  files, provisioning profiles, or a signed personal bundle.
- Apple Development signing is for local development/testing. It is not the
  Tencent Developer ID identity and does not grant distribution or notarization
  rights. Full original host trust requires the original signing identity.
- Do not claim that one successful TextEdit or Edge test proves every protected
  host will accept the modified input method.

## Read first

Load the references that match the task:

- `references/v-panel-and-settings.md`: V-panel content cells and settings
  accent work.
- `references/flutter-assets.md`: PNG-based UI controls (radio buttons,
  checkboxes) that require asset replacement rather than binary patching.
- `references/637-xcode-signing.md`: Apple Account, Xcode, certificate/private-key
  recovery, entitlements, and signing pitfalls.
- `references/637-installation-and-troubleshooting.md`: installation, enabling,
  TIS/LaunchServices diagnosis, missing input-menu entries, global switching
  failures, cross-app failures, and rollback.
- `references/637-deep-customization.md`: how to add a customization or adapt
  a later build while keeping patches guarded and reproducible.

## Canonical workflow

### 1. Prepare and identify the package

Set paths in the operator's shell. Never point a patcher at the installed app:

```bash
ROOT="/Users/fitz/Documents/202607/wetype-637-work"
OFFICIAL="$ROOT/WeType-official.disabled"
CUSTOM="$ROOT/WeType-637-custom.bundle"
ENTITLEMENTS="$ROOT/signing-entitlements"
SCRIPTS="$ROOT/white-selected-black-text-tools"
BACKUPS="$ROOT/backups"
```

Check the version and bundle shape before any patch:

```bash
/usr/bin/plutil -extract CFBundleShortVersionString raw -o - \
  "$OFFICIAL/Contents/Info.plist"
/usr/bin/plutil -extract CFBundleVersion raw -o - \
  "$OFFICIAL/Contents/Info.plist"
/usr/bin/file "$OFFICIAL/Contents/MacOS/WeType"
/usr/bin/lipo -info "$OFFICIAL/Contents/MacOS/WeType"
/usr/bin/codesign --verify --deep --strict "$OFFICIAL"
```

The bundled 637 profile expects `2.2.1`, `637`, and a universal
`arm64 x86_64` main executable. Keep a separate untouched copy of the official
package and record its hash.

### 2. Build from the official copy

The parameterized builder creates a fresh custom output and fails closed on
unexpected bytes:

```bash
/bin/bash "$SCRIPTS/build_wetype_637.sh" "$OFFICIAL" "$CUSTOM"
```

The build phase needs the macOS Command Line Tools (`lipo`, Perl, and clang),
not a working Swift toolchain. The PNG patcher is compiled from Objective-C.
Full Xcode is needed only when the operator must create or recover an Apple
Development signing identity.

The builder does the following in order:

1. Check the version and reject `/Library/Input Methods/WeType.app` as input.
2. Copy the official bundle to a temporary build directory.
3. Thin the universal `WeType` executable into arm64 and x86_64 files.
4. Patch native V-menu and voice constants in each thin file, then rebuild the
   universal executable.
5. Patch the Flutter AOT `App.framework` arm64 and x86_64 slices separately.
6. Patch SVG and PNG settings assets.
7. Patch named colors in `Assets.car`.
8. Run resource/version checks and emit the custom bundle only at the end.

Do not use broad `sed`, `perl -pi`, or string replacement across a Mach-O file.
Every binary patch must assert the expected old bytes, accept the exact target
bytes when rerun, and identify the architecture and thin-file offset.

### 3. Create or recover a local signing identity

Check the identity and its private key:

```bash
/usr/bin/security find-identity -v -p codesigning
/usr/bin/security find-certificate -a -p -c "Apple Development" \
  | /usr/bin/openssl x509 -noout -subject -nameopt RFC2253 \
  2>/dev/null || true
```

If `/Applications/Xcode.app` is absent, install Xcode and sign in under Xcode >
Settings > Accounts before attempting identity recovery. Point `xcode-select`
at the full Xcode developer directory after installation; Command Line Tools
alone cannot perform the automatic-signing recovery build.

Use the full identity name or SHA-1 printed by `find-identity`, not the short
code in parentheses in the certificate display. Read the Team ID from Xcode's
team selector or the `TeamIdentifier` of an automatically signed sample app.
If the certificate appears in Keychain but `find-identity` reports zero valid
identities, the private key is missing or not paired. Follow
`references/637-xcode-signing.md`; the verified recovery path is a minimal
automatically signed macOS Xcode project built with
`xcodebuild -allowProvisioningUpdates`.

Create generic entitlements using the operator's Team ID:

```bash
/bin/bash "$SCRIPTS/create_entitlements.sh" "$ENTITLEMENTS" "$TEAM_ID"
```

Do not copy another person's Team ID into a new signature. Do not sign with
`codesign --sign -`; ad-hoc signatures commonly make input work in one host but
fail in another and discard the original trust boundary.

### 4. Sign inside out

Sign with the operator's identity:

```bash
/bin/bash "$SCRIPTS/sign_wetype_637.sh" \
  "$CUSTOM" "$IDENTITY" "$ENTITLEMENTS"
```

The script signs Settings and Feedback subtrees, Sparkle leaves and framework,
root frameworks, `BinaryDelta`, helper executables and apps, FinderSync, and
the top-level WeType app last. Use `--timestamp=none` for local testing and
preserve the appropriate entitlements for each nested target.

### 5. Verify before installation

```bash
/bin/bash "$SCRIPTS/verify_wetype_637.sh" "$CUSTOM"
/usr/bin/codesign --verify --deep --strict --verbose=2 "$CUSTOM"
/usr/bin/codesign -d --verbose=4 "$CUSTOM" 2>&1 \
  | /usr/bin/grep -E 'Authority|TeamIdentifier|Identifier'
```

Verification must cover version, both main architectures, native voice/V-menu
bytes, both Flutter slices, `Assets.car`, the connect icon, FinderSync, and a
valid deep signature. A successful build is not yet an installed result.

### 6. Install and enable from Terminal

Install only the verified custom bundle, using the user's Terminal:

```bash
sudo /bin/bash "$SCRIPTS/install_wetype.sh" \
  "$CUSTOM" "/Library/Input Methods/WeType.app" "$BACKUPS"
```

The installer refuses non-root execution, verifies the source, creates a dated
backup of the installed bundle, stages the new bundle, verifies the staged
copy, restores `root:wheel`, replaces the app, and restarts input-method
related processes. It does not require a full computer reboot.

Then enable/select WeType in System Settings > Keyboard > Input Sources. Leave
exactly one WeType Pinyin row. Run `$SCRIPTS/diagnose_tis.sh` to inspect the
registry; the System Settings editor is the reliable way to remove duplicates.
An enabled or selected TIS row does not prove that WeType appears in the input
menu or that a physical shortcut can activate it. Check those separately.

### 7. Test in layers

Test in this order:

1. TextEdit: select WeType Pinyin and type Chinese.
2. Microsoft Edge: test a normal web text field.
3. The user's real target host, especially if it is protected, sandboxed, or
   uses a special text field.
4. Settings, candidate/V menu, voice window, appearance previews, and any
   newly patched asset.

Record which host, input source, process, signature, and surface failed. Do
not infer a global fix from a single successful host.

After install, tell the user to fully quit settings, reselect WeType, and
press V. Keep the rollback backup path in the report.

If the user rejects the result (撤回 / obviously wrong / blank controls):
restore that timestamped backup with the same installer before any further
patching. Do not iterate on a live broken install.

## Deep customization routing

Start with the surface map rather than guessing where a green pixel came from:

| Surface | Layer | Correct method |
| --- | --- | --- |
| Candidate selected background | native ARM64/x86_64 plus `Assets.car` | patch main thin slices and named color records (`bg03`/`bc16`) |
| Candidate selected text | native + V-panel cells | patch to black text (`bc17` → `blackColor`) |
| V-panel content cells | Flutter AOT `VModeItemCell` | split highlight branch, retarget foreground |
| Native voice ripple | `Contents/MacOS/WeType` | patch normalized floating-point channel constants in both slices |
| Settings sidebar selected | Flutter AOT snapshot object | white fill + black text, shared with switches |
| Settings switches | Flutter `CustomSwitch` | dark recessed track + round thumb (OFF `#2A2A2A`/`#B3B3B3`, ON `#3A3A3C`/`#FFFFFF`) |
| Settings buttons | Flutter AOT MOVZ groups | white fill + zero nearby `Color(0xFFFFFFFF)` to black |
| Radio buttons | PNG assets | Replace all resolution/theme variants |

"黑白" is two separate looks, not one shared accent:
- **Highlight** (sidebar selected pill, selected radio): white fill + black text,
  matching the candidate selected state.
- **Switches**: dark recessed track + round thumb, both states designed — OFF
  track `#2A2A2A` / thumb `#B3B3B3`; ON track `#3A3A3C` / thumb `#FFFFFF`.
  ON is thumb position and slight track lift, never blue/cyan/white fill.

Do not invert the whole dark settings window to light theme unless asked.
Never ship a fill-only white patch — if you cannot retarget the matching
selected foreground in the same build, stop and restore.

For any new surface:

1. Capture the exact screen and identify which child app/window owns it.
2. Search text/color strings in resources first.
3. Inspect both architectures with `otool`, `strings`, and `xxd`.
4. Thin the relevant universal file before using virtual addresses or offsets.
5. Confirm the old bytes or literal occur at the intended count.
6. Patch one layer at a time and preserve gradient hierarchy.
7. Rebuild, sign, verify, and test the actual host.
8. Add the finding to the reference before sharing the skill.

## Bug triage

Use `references/637-installation-and-troubleshooting.md` and follow this split:

| Symptom | Likely cause | Correct response |
| --- | --- | --- |
| Build fails with unexpected byte | wrong version, wrong slice, or already patched/unrelated package | stop; do not force the patch |
| Signature fails | first failing nested path, missing private key, wrong entitlements | inspect identity, restore entitlements, sign inside out |
| Settings is blue but candidates are green | candidate rendering is native or `Assets.car` | patching Flutter alone cannot fix it |
| Control-Space fails everywhere and WeType missing from input menu | TIS registration / source switching | compare actual menu with System Settings and TIS; follow input-source recovery |
| Input works in TextEdit but not another app | TIS duplicates, process paths, LaunchServices, or host trust | inspect before deleting dictionaries/caches |
| WeType absent from `pgrep` | lazy-started process | absence alone is not evidence of stale process |
| Voice remains green | native `VoiceRippleView` constants missed | App.framework changes do not reach that view |
| A gradient becomes flat pale blue | several stops flattened | restore proper gradient hierarchy |
| Post-reboot failure | TIS registration or signature | verify before changing data |
| V-panel content blank | `VModeItemCell` / `VModeSymbolsItemView` still call `colorNamed:bc17` | retarget `bc17` → `blackColor` for V-panel cells |
| Sidebar and switches together change | shared snapshot token `7f7501b8` / `7f6317bf` | split `CustomSwitch` colors first |
| White-on-white | white fill without matching black foreground | retarget both fill AND foreground together |

Rollback only to a dated backup that was verified before replacement:

```bash
sudo /bin/rm -rf "/Library/Input Methods/WeType.app"
sudo /usr/bin/ditto "/path/to/verified-backup.bundle" \
  "/Library/Input Methods/WeType.app"
sudo /usr/sbin/chown -R root:wheel "/Library/Input Methods/WeType.app"
sudo /usr/bin/codesign --verify --deep --strict \
  "/Library/Input Methods/WeType.app"
```

## Pitfalls (637 battle-tested)

### Architecture and encoding

- Treat arm64 and x86_64 as separate targets; thin with `lipo` before applying
  recorded offsets. Patch each slice separately.
- For Flutter AOT channels, ARM64 values are often `MOVZ` fragments and x86_64
  values may be immediates or ASCII. For native voice and `Assets.car`, colors
  are normalized `channel / 255` IEEE-754 little-endian doubles.
- Do not convert a fat-file virtual address directly into a file offset.
- ARM64 immediates: patch only the immediate bits while preserving destination
  register and instruction shape. Record anchor offset and following instruction
  words that encode the replacement color.
- x86_64 may store the same value as ASCII literal or immediate in instruction.
  The 637 patcher uses instruction start and writes immediate at `offset + 1`.

### V-panel and candidate bar

- Treat `VModeMenuItemView` as the V-mode category row only — clipboard,
  hotwords, and symbols use `VModeItemCell` / `VModeSymbolsItemView` and still
  call `colorNamed:bc17` after the candidate-bar patch, which is why those
  panels go blank on white highlight.
- V-panel content needs `bc17` → `blackColor` retarget.
- Do not retarget a shared `colorNamed` that both selected and unselected
  paths join into (common on x86_64) — unselected light text on dark would
  become black-on-dark. Split the highlight branch or skip that arch.
- White `bg03`/`bc16` with leftover `colorNamed:bc17` or engine white text is
  white-on-white. Check all three candidate layers, then V-panel content cells,
  then settings fills+text together.

### Flutter settings colors

- Flutter settings colors are three MOVZ/ORRs (keep alpha, R `lsl #16`, G<<8, B).
- Sparse groups omit red; turning them white needs R=`#ff` `lsl #16` plus
  a `0xffff` green-blue immediate, not G/B=`#ff` alone (`#00FFFF` cyan).
- Snapshot tokens that start `7f` and end `b8` are the same R≈0 family:
  `7f7501b8` is brand blue, `7f7f7fb8` only raises G and paints cyan, not white.
- Settings sidebar selected and Switch ON share the AOT snapshot palette
  (`7f7501b8` / `7f6317bf`), not the MOVZ groups. Do not retarget that shared
  object to white or dark — it recolors sidebar and switches together.
- Split `CustomSwitch` on/off colors from the sidebar pill before changing
  snapshot bytes.
- `7f7f7f7f` or copying another 5-byte cluster record SIGSEGVs WeTypeSettings.
  Keep 4-byte length and last-byte tag if you must rewrite a token.
- Launch-test a signed settings copy (`open -n`) and confirm no new
  DiagnosticReports ips before asking to install.
- MOVZ-only patches may leave snapshot-based controls (sidebar, switches)
  unchanged. Verify both instruction-level and snapshot-level colors.
- Settings toggles are `CustomSwitch` in
  `package:my_desktop_app/modules/setting/views/components/switch/index.dart`
  (`jsonToColor` is the converter). `_trackColor@394083257` is Material
  scrollbar; `_getFillColor@430019562` is InputDecorator — neither is the
  settings switch.

### Settings buttons and foreground

- Changing accent fills to white without a matching selected-foreground
  retarget recreates white-on-white.
- Settings buttons need the nearby `Color(0xFFFFFFFF)` builders zeroed to
  black, not `#F7F7F7` (that is unrelated chrome and punches holes).
- When a settings highlight looks wrong, sample non-dark pixels. Median
  R≪G=B=255 is the R≈0 snapshot family (cyan `#00FFFF`), not white.

### System blue not used

- Do not treat arm64 input-settings MOVZ `#007AFF` at `0x235214` or the
  `#050505`/`#4D4D4D` Color() pair at `0x2a124c`/`0x2a1320` as sidebar or
  `CustomSwitch` — a copy that only changes those still paints blue selected
  pills and blue ON tracks.
- System blue (`#007AFF` light / `#0A84FF` dark) in isolate data at `0x444ada`
  (arm64 App.framework) is not used by sidebar pills or switches — changing
  that byte sequence to red/white/black left those controls unchanged.

### Code caves and runtime hooks

- Do not BL into trailing `__text` zeros of App.framework to remap Color
  channel extractors or `Color.lerp` — that tail is not a valid executable
  cave (SIGILL even with `bti c`) and is not a substitute for a codec-correct
  snapshot token or a widget-local Color on `CustomSwitch`.
- Frida/lldb attachment to WeTypeSettings fails with permission-denied under
  normal user even with sudo — macOS SIP protects system input method processes.
  Do not attempt runtime hooking; use static binary patches and test-sign
  copies instead.

### Settings preview and LSUIElement

- WeType and WeTypeSettings are `LSUIElement` — missing from Launchpad is
  expected, and `open -n` on the nested settings app can leave a PID with no
  window.
- Visible previews belong on a Desktop copy with `LSUIElement` false and
  distinct `CFBundleName`, then re-sign. Never change `LSUIElement` on the
  installed input method.
- For a human look after a diagnostic copy, wait for what the user sees
  (sidebar vs switches) before another color pass — do not stack guesses.

### Asset-only changes

- Radio buttons (单选按钮) and other UI widgets may use PNG assets in
  `flutter_assets/assets/modules/setting/images/` rather than binary color
  constants. Check `icon_singlechoice_sel_light.png` (light theme selected),
  `icon_singlechoice_sel_dark.png` (dark theme), plus 2.0x and 3.0x variants.
- Modify the PNG pixels directly (replace blue with target color while
  preserving alpha), then replace all resolution variants and both theme
  versions to avoid partial changes.
- Asset-only changes (PNG replacement) do not require re-signing, but may need
  App.framework re-opened or settings fully quit to flush cache.

### Testing and evidence

- When stuck on unreachable UI colors after exhausting known offsets, create
  test binaries with obvious replacement colors (red, white, black) at
  candidate positions, sign each one separately, launch as `open -n` copies,
  and observe which test changes the target UI. A no-op across all test colors
  proves that position is not the control path.
- Patch scripts that expect official green will not apply to an already-blue
  installed build — assert current bytes, never disable the assertion.
- Diagnose blank selected UI as missing foreground, not missing background.
- If the input method vanished from the menu after an install, TIS-enable both
  `com.tencent.inputmethod.wetype` and `com.tencent.inputmethod.wetype.pinyin`.

### Publishing and recovery

- Do not ship or commit Tencent binaries in the GitHub skill repo; a 300MB+
  Recovery zip is the wrong artifact even if it is the running build.
- Reuse existing patch/verify/sign/install scripts if a `wetype-637-work` tree
  exists. Do not reinvent offsets.
- GitHub for this project is not the source of truth if the default-branch tree
  is a stub README. Check Releases and local skill clones before calling the
  repo empty. Do not upload WeType/Tencent binaries, dictionaries, or signed
  bundles. Publish skill source + docs only.

## What the shareable package excludes

The package must not contain `WeType.app`, `.bundle` outputs, dictionaries,
Application Support data, Keychain exports, private keys, personal Team IDs,
or machine-specific absolute paths. Before sharing, run a search for email
addresses, SHA-1 identities, `/Users/`, `/Library/Input Methods/`, and private
certificate material; only generic examples should remain.
