# Deep customization method (637)

## Choose the correct layer

The 637 package has three different code/resource layers:

1. Native Swift/AppKit/InputMethodKit code in `Contents/MacOS/WeType`.
2. Flutter AOT machine code in `WeTypeSettings.app/.../App.framework`.
3. Flutter assets and the top-level `Contents/Resources/Assets.car`.

A screen can combine more than one layer. A green settings preview does not
prove that the candidate or voice view uses the same constant.

## Discovery loop

Work from a fresh official copy and record a hash before patching:

```bash
/usr/bin/shasum -a 256 "$OFFICIAL/Contents/MacOS/WeType"
/usr/bin/file "$OFFICIAL/Contents/MacOS/WeType"
/usr/bin/lipo -thin arm64 -output /tmp/wetype-arm64 \
  "$OFFICIAL/Contents/MacOS/WeType"
/usr/bin/lipo -thin x86_64 -output /tmp/wetype-x86_64 \
  "$OFFICIAL/Contents/MacOS/WeType"
/usr/bin/strings -a /tmp/wetype-arm64 \
  | /usr/bin/grep -i -E 'green|voice|mode|gradient'
/usr/bin/otool -arch arm64 -tV "$OFFICIAL/Contents/MacOS/WeType" \
  | /usr/bin/grep -A12 -B4 'VModeMenuItemView'
```

For Flutter AOT, thin `App.framework` separately. The virtual address printed
by `otool` is not automatically a file offset in a fat Mach-O. Use the thin
file offset in the patcher and inspect the bytes with `xxd -s OFFSET`.

Use `blutter` or another Flutter AOT disassembler only when needed to map a
closure/function to a screen. Keep the disassembler output as evidence; do not
guess an offset from a screenshot.

## 637 surface map

The exact offsets and old/new bytes are documented in `v-panel-and-settings.md`
and `flutter-assets.md`. The key principle for 637:

| Target | 637 source | Customization approach |
| --- | --- | --- |
| Candidate selected background | native + `Assets.car` | white fill (`bg03`/`bc16`) |
| Candidate selected text | native + V-panel cells | black text (`bc17` → `blackColor`) |
| V-panel content cells | Flutter AOT `VModeItemCell` | split highlight branch, retarget foreground |
| Settings sidebar selected | Flutter AOT snapshot object | white fill + black text, shared with switches |
| Settings switches | Flutter `CustomSwitch` | dark recessed track + round thumb (OFF `#2A2A2A`/`#B3B3B3`, ON `#3A3A3C`/`#FFFFFF`) |
| Settings buttons | Flutter AOT MOVZ groups | white fill + zero nearby `Color(0xFFFFFFFF)` to black |
| Radio buttons | PNG assets | Replace all resolution/theme variants |

Keep the "黑白" distinction: white fill + black text for highlights and selected
states; dark recessed for switches (not white fill). Do not invert the whole
dark settings window.

## Encoding rules

### ARM64 immediates

Some short ASCII literals are assembled through `MOVZ`/`MOVK` instructions.
Patch only the immediate bits while preserving the destination register and
instruction shape. For 637 V-menu and settings colors, record the anchor
offset and the following instruction words that encode the replacement color.

### x86_64 immediates and literals

x86_64 may store the same value as an ASCII literal or as an immediate in an
instruction. The 637 patcher uses the instruction start and writes the
immediate at `offset + 1` where appropriate. Do not assume ARM64 offsets match
x86_64 offsets.

### Normalized doubles

Native voice and `Assets.car` RGB values may be stored as IEEE-754 doubles:

```text
value = channel / 255
```

The bytes are little-endian on disk. Use a structured packer such as Perl's
`pack('d', $channel / 255)` and assert the exact old bytes before replacing
them. A decimal string replacement cannot reach these values.

### Flutter AOT snapshot objects

Settings sidebar and `CustomSwitch` share the AOT snapshot palette. Tokens like
`7f7501b8` (brand blue) and `7f6317bf` are 4-byte records with the last byte
as a tag. Do not rewrite these shared objects to white or dark until
`CustomSwitch` has its own Color, or both controls will change together.

Keep the 4-byte length and last-byte tag when rewriting. `7f7f7fb8` keeps R≈0
and paints `#00FFFF` cyan (not white). Copying another 5-byte cluster or
setting `7f7f7f7f` causes SIGSEGV. Launch-test signed copies with `open -n`
and check DiagnosticReports before installing.

### PNG and SVG

SVG color literals can be replaced only in the intended asset root. For PNG,
decode to RGBA, preserve every alpha byte, and recolor non-transparent pixels;
do not flatten antialiasing. Radio buttons require replacing all resolution
variants (1.0x, 2.0x, 3.0x) and both theme versions (light/dark).

## Adding a new patch

Use this low-level contract:

1. Name the exact version and file in the patcher output.
2. Verify the old value/count at the intended location.
3. Accept an already patched target only if the target bytes are exact.
4. Fail on any other value, short read, architecture mismatch, or resource
   dimension mismatch.
5. Keep arm64 and x86_64 code paths separate.
6. Rebuild from the official copy rather than stacking patches on a prior
   custom output.
7. Extend the verifier with the new expected value.
8. Test the actual UI surface in at least TextEdit and one target host.

Never use `codesign --sign -`, binary-wide color substitution, unguarded
offsets, or a patch that silently succeeds with an unknown old value.

## Known 637 pitfalls

From battle-tested experience:

- **V-panel content blank**: `VModeItemCell` / `VModeSymbolsItemView` still call
  `colorNamed:bc17` after candidate-bar patch. Retarget `bc17` → `blackColor`
  for V-panel cells specifically.

- **Shared snapshot tokens**: Do not retarget `colorNamed` that both selected
  and unselected paths join into (common on x86_64). Split the highlight branch
  or skip that arch.

- **Settings sidebar and switches together**: `7f7501b8` / `7f6317bf` snapshot
  objects are shared. Changing them affects both sidebar selected pills and
  CustomSwitch ON tracks. Split `CustomSwitch` colors first.

- **MOVZ-only patches**: Changing only MOVZ groups may leave snapshot-based
  controls (sidebar, switches) unchanged. Verify both instruction-level and
  snapshot-level colors.

- **Cyan not white**: Median R≪G=B=255 is the R≈0 snapshot family (`#00FFFF`
  cyan). Sparse groups that omit red need R=`#ff` `lsl #16` plus `0xffff`
  green-blue immediate for white, not just G/B=`#ff`.

- **System blue not used**: `#007AFF` at arm64 App.framework `0x444ada` is in
  isolate data but not used by sidebar or switches. Test patches proved no UI
  change.

- **White-on-white**: White fill without matching black foreground recreates
  the original problem. V-panel and settings need both fill AND foreground
  retargeted together.

- **Flutter asset cache**: Asset-only changes (PNG replacement) do not require
  re-signing, but may need App.framework re-opened or settings fully quit to
  flush cache.

- **Test binary crashes**: Do not BL into trailing `__text` zeros of
  App.framework for color remapping — that tail is not executable (SIGILL).
  Keep patches to existing code/data sections.

- **LSUIElement preview**: WeType and WeTypeSettings are `LSUIElement` — no
  Launchpad icon is expected. For visible previews, copy to Desktop with
  `LSUIElement` false and distinct `CFBundleName`, then re-sign.

- **Frida/lldb blocked**: macOS SIP protects system input method processes.
  Runtime hooking fails with permission-denied. Use static binary patches and
  test-signed copies instead.

## Updating beyond 637

Treat every new WeType build as a new reverse-engineering target. Re-check:

- version and architectures;
- symbol/function locations;
- thin-file offsets;
- AOT closure groups;
- resource dimensions and named-color records;
- nested bundle layout and entitlements;
- FinderSync/helper additions;
- input-source identifiers.

Do not reuse the 637 patchers on any other build without re-deriving the
offsets and updating both the reference and verifier first.
