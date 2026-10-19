# V-panel and settings color lessons

## V-panel blank rows

`VModeMenuItemView` is only the category row. Clipboard / 常用语 / 符号 content
uses `VModeItemCell` and `VModeSymbolsItemView`, which still call
`colorNamed:bc17` after the candidate-bar patch. White `bg03` plus leftover
`bc17` is white-on-white.

Arm64 (thin-file): retarget those two `colorNamed:` selector loads to
`blackColor` (same page `0x1031a3000`, LDR `#0x130`). Do not retarget a
shared x86 `colorNamed` that selected and unselected join into.

## Settings 黑白

Do not invert the whole dark window to light theme. Highlight and switches
are different looks:

- Sidebar selected pill and selected radio: white fill + black text.
- `CustomSwitch` OFF: track `#2A2A2A`, thumb `#B3B3B3`. ON: track `#3A3A3C`,
  thumb `#FFFFFF`. ON is thumb position and slight track lift — never blue,
  cyan, or a white track.

Independent color sources. Do not install a build that only hits some of
them:

1. Brand MOVZ fills (`#007AFF` / `#0A84FF` / gradient ends) -> white. Keep
   alpha. Sparse groups omit red — use R=`#ff` `lsl #16` plus a `0xffff`
   green-blue immediate, not G/B=`#ff` alone (`#00FFFF`). White MOVZ fills
   without black labels blank 隔空传送 / 本机-style buttons.
2. Selected/button *text* is not `#F7F7F7`. On arm64 the contrast builders
   next to appearance / phone-download accents are at `0x286304` and
   `0x2a33c4` (`Color(0xFFFFFFFF)`). Zero RGB and drop the blue `orr` with
   `x0=#ff` so labels stay visible on white fills. Remapping `#F7F7F7`
   punches holes in unrelated chrome.
3. Sidebar selected pill and Switch ON share the v7 static snapshot tokens
   `7f7501b8` / `7f6317bf` (often as 5-byte `…c0` records), not the MOVZ
   groups. v7 green-to-blue worked because it wrote codec-correct encodings
   of the same object kind. The `7f…b8` family is R≈0 (blue/cyan): rewriting
   `7f7501b8` to `7f7f7fb8` keeps R at 0 and paints `#00FFFF`. Do not replace
   with `7f7f7f7f` or copy another cluster record (`7f7f7f6fc0` included) —
   those SIGSEGV WeTypeSettings. Do not retarget that shared object to white
   or dark until `CustomSwitch` on/off colors are split from the sidebar pill.
   `jsonToColor` in `switch/index.dart` is a converter, not a second palette.

Launch-test: sign the copy's `App.framework`, `FlutterMacOS.framework`, then
`WeTypeSettings.app`; `open -n` the nested settings app; require a live
process on the copy path and no new WeTypeSettings ips. Pixel-sample the
highlight if it is not obviously white: low R with G=B=255 means cyan from
the `7f…b8` family, not a white encoding.

Do not trampoline into trailing `__text` padding to rewrite Color extractors
or `Color.lerp` — SIGILL. Isolate snapshot data starts at
`_kDartIsolateSnapshotData` (`0x425a00`); hash `ee1eb666c76a5cb7746faf39d0b97547`
is Flutter 3.19.x. Settings toggles are still `CustomSwitch`, not
`material/switch.dart`.

SVG `#007AFF` / `#64D2FF` in flutter_assets can go `#FFFFFF`. Leave error
red `#FF5252`. Official-green patch scripts will not apply to an already-blue
installed build — use live bytes as `old`.

On reject/rollback: restore the timestamped pre-install backup with
`install_wetype.sh` before another color pass. After an install that drops
the input method from the menu, TIS-enable both
`com.tencent.inputmethod.wetype` and `com.tencent.inputmethod.wetype.pinyin`.
