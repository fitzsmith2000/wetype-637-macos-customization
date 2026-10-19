# Installation, enablement, and troubleshooting (637)

## Install safely

The installed input method lives at:

```text
/Library/Input Methods/WeType.app
```

The color customization keeps this app name and location. It does not create a
separate `WeTypeBlue.app` or an app in `/Applications` unless renaming was
explicitly requested. Check the installed path before treating a missing
Applications or Spotlight entry as an uninstalled input method.

Always keep the official source and a dated pre-install backup. Use the
provided installer from the user's Terminal, not an AppleScript administrator
shell:

```bash
sudo /bin/bash scripts/install_wetype.sh \
  "$CUSTOM" "/Library/Input Methods/WeType.app" "$BACKUPS"
```

The installer should:

1. Require root and reject an unsigned source.
2. Copy the installed bundle to a dated backup.
3. Verify the backup before replacement.
4. Stop WeType helper processes.
5. Stage and verify the new bundle.
6. Replace the installed path and restore `root:wheel`.
7. Restart `TextInputMenuAgent` and verify the installed signature.

It should not delete WeType dictionaries, `~/Library/Application Support`, or
`~/Library/Caches` as part of a normal install.

## Enable the input source

Open System Settings > Keyboard > Input Sources, add or select WeType Pinyin,
and remove duplicate WeType rows one at a time. Leave one parent input method
and one `com.tencent.inputmethod.wetype.pinyin` mode.
The parent and pinyin mode are distinct, expected TIS records. A System Settings
row labelled `微信输入法、微信输入法` can contain the name and subtitle of one
entry; that label alone does not establish a duplicate.

Inspect the actual TIS registry with:

```bash
/bin/bash scripts/diagnose_tis.sh
```

The app is lazy-started. No `WeType` process while another input source is
selected is normal; it is not proof of a stale process. The TIS result and an
actual text-entry test are more important. Conversely, a running process or
`enabled=true` / `selected=true` does not prove that the menu lists WeType or
that a physical shortcut works.

For a one-time selection after TIS is clean, compile the optional selector from
the work directory or skill. It enables/selects WeType and therefore belongs to a
repair or activation task, not a read-only diagnostic check:

```bash
# From work directory
/usr/bin/clang -framework Carbon \
  /Users/fitz/Documents/202607/wetype-637-work/white-selected-black-text-tools/select_wetype_tis.c \
  -o /tmp/select_wetype_tis
/tmp/select_wetype_tis
```

Read back the selected source with `scripts/diagnose_tis.sh`, then verify the
menu and actual input. The helper's success message only confirms that the API
accepted its request.

If macOS blocks an unsigned temporary helper, select WeType manually in System
Settings or compile/run it from a trusted local development directory.

## Test hosts in order

Use TextEdit first, then Edge, then the real target application. For each host
record:

- selected source ID;
- whether the candidate window appears;
- whether committed Chinese text arrives;
- installed bundle path and signature;
- whether the host is sandboxed/protected;
- whether voice input and auxiliary windows work.

TextEdit success only proves the basic InputMethodKit path. A protected host
may reject a personal Apple Development signature even when the input method
is otherwise correct.

## Triage matrix

### Control-Space fails everywhere and WeType is missing from the input menu

First distinguish source switching from WeType's internal Chinese/English
mode. Ask whether the failure affects every app and whether WeType appears in
the menu bar's input menu. If it appears, check whether selecting it manually
allows Chinese input. If it is already selected but types English, inspect
WeType's own language-toggle setting (often Shift).

Check the installed version and signature with the matching versioned verifier,
and inspect TIS with `scripts/diagnose_tis.sh`. Inspect System Settings >
Keyboard > Text Input > Input Sources > Edit
(`系统设置 > 键盘 > 文字输入 > 输入法 > 编辑…`) and the actual menu separately.
Stored preferences are supporting evidence:

```bash
/usr/bin/defaults read com.apple.symbolichotkeys AppleSymbolicHotKeys
/usr/bin/defaults read com.apple.HIToolbox
```

For the standard macOS bindings, symbolic hotkey 60 is the previous input
source (Control-Space), and 61 is the next input source
(Control-Option-Space). Check both their enabled state and actual binding.
`AppleEnabledInputSources` in `defaults` can disagree with the live TIS list
and System Settings; its omission of WeType alone is not proof that WeType is
disabled. Use the visible menu and a physical switching test to establish the
symptom.

When repair is in scope and the installed bundle verifies, recover a mismatch
between the enabled source and the menu as follows:

1. Optionally refresh the input-menu service once. Resolve current PIDs for
   `TextInputMenuAgent` and `TextInputSwitcher`, verify their system executable
   paths, and send TERM only to those processes. Read back their PIDs before
   claiming a restart: a successful signal does not prove a process exited.
   If the menu is still missing WeType, continue to the source editor.
2. Keep ABC available. In the input-source editor, select the single WeType
   row and use the minus/remove button to remove that entry. This removes the
   source from the configured list, not the installed bundle or dictionaries.
3. Use plus/add, choose Simplified Chinese > WeType (`简体中文 > 微信输入法`),
   add it, and finish the editor. Confirm that one WeType entry is present.
4. Select WeType from the input menu, or use the one-time TIS selector above
   after rebuilding the entry. Read back its enabled/selected state.
5. Verify that the menu now contains WeType, physical Control-Space switches
   between ABC and WeType, and Chinese candidates/committed text work in an
   ordinary text field. Ask the user to test if the automation cannot exercise
   the real keyboard path. Stop once the user's switching works.

Some automation delivers text directly or overrides the keyboard layout;
injected letters or an app-targeted Control-Space event do not establish how
the physical shortcut or IME behaves. Do not report the fix as verified solely
from TIS flags or a successful API call.

If the menu remains missing after one remove/add/select cycle, retain the
installed app and gather current registration paths and relevant system logs
before escalating. Avoid repeated source removal, direct writes to protected
`com.apple.inputsources`, cache/dictionary deletion, or rebuilding/re-signing a
bundle that still verifies. If manual selection works but shortcuts fail,
investigate the shortcut path. A remote-control or clipboard app having an
event tap is not evidence that it intercepted this shortcut; establish whether
the user is operating locally or remotely before attributing the failure.

### Input works in TextEdit but not another app

Run:

```bash
/usr/bin/codesign --verify --deep --strict \
  "/Library/Input Methods/WeType.app"
/usr/bin/codesign -d --verbose=4 \
  "/Library/Input Methods/WeType.app" 2>&1 \
  | /usr/bin/grep -E 'Authority|TeamIdentifier|Identifier'
/bin/bash scripts/diagnose_tis.sh
```

If TIS shows duplicate pinyin rows, remove the duplicate rows in System
Settings. Direct writes to protected `com.apple.inputsources` preferences can
be rejected or silently reverted. Do not start by deleting caches.

If TIS is clean and the bundle is correctly signed, the remaining likely
boundary is the host's trust/permission policy. Full compatibility would
require the original Tencent Developer ID identity.

### Settings is blue but candidates or voice are green

These are separate layers:

- candidate selection: native main executable plus `Assets.car`;
- voice ripple: native main executable's floating-point constants;
- settings pages: Flutter `App.framework`, SVG, PNG resources.

Re-run the versioned verifier and inspect the exact layer. Do not patch only
Flutter when the green view is native.

### A gradient is pale or flat

Early patch attempts may have flattened gradients. For 637, verify the target
hierarchy maintains proper color stops for each surface. If a surface looks
wrong, identify its owning layer before changing another AOT group.

### The builder reports unexpected bytes

Treat this as a hard stop. It means one of:

- the source is not exactly 2.2.1 (637);
- a fat-file offset was used without thinning;
- the official package was already patched;
- an upstream update changed the AOT layout;
- the wrong resource was selected.

Make a fresh official copy, compare hashes, and re-disassemble. Never force a
patch by removing the assertion.

### Reboot or restart appears to undo the change

Check the installed bundle version, deep signature, TIS rows, selected source,
and process path. The system may have restored the official package, retained
a duplicate input-source registration, or selected a different input source.
Do not interpret an absent lazy-started process as cache residue.

## LaunchServices and stale work copies

If System Settings lists old work copies, inspect registration paths before
unregistering anything:

```bash
LSREGISTER=/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister
"$LSREGISTER" -dump | /usr/bin/grep -i -A5 -B2 'WeType'
```

Only unregister a path after confirming it no longer exists and is not the
installed `/Library/Input Methods/WeType.app`. Then re-register the installed
bundle and reopen System Settings. LaunchServices cleanup does not replace TIS
duplicate cleanup.

## Rollback

Use only the backup path printed by the installer and verify it first:

```bash
/usr/bin/codesign --verify --deep --strict \
  "/path/to/verified-backup.bundle"
sudo /bin/rm -rf "/Library/Input Methods/WeType.app"
sudo /usr/bin/ditto "/path/to/verified-backup.bundle" \
  "/Library/Input Methods/WeType.app"
sudo /usr/sbin/chown -R root:wheel \
  "/Library/Input Methods/WeType.app"
sudo /usr/bin/codesign --verify --deep --strict \
  "/Library/Input Methods/WeType.app"
```

Do not guess a backup filename. Do not remove user data to solve a signature or
input-source registration problem.

## CDP is not part of this workflow

WeType candidate, voice, and input-source surfaces are native InputMethodKit or
AppKit components. CDP is for Chromium/Electron pages and cannot bypass
codesigning, Team ID, TIS registration, or host trust. Runtime injection is not
a durable replacement for a correctly rebuilt and signed bundle.
