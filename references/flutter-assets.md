# Flutter assets structure

WeTypeSettings.app uses Flutter with PNG assets for many UI controls, not just
binary color constants.

## Asset location

```
/Library/Input Methods/WeType.app/Contents/MacOS/WeTypeSettings.app/Contents/
  Frameworks/App.framework/Versions/A/Resources/flutter_assets/
```

Manifest: `AssetManifest.json` lists all assets.

## Setting UI images

### Radio buttons (单选按钮)

- Light theme selected: `assets/modules/setting/images/icon_singlechoice_sel_light.png`
- Dark theme selected: `assets/modules/setting/images/icon_singlechoice_sel_dark.png`
- Dark theme normal: `assets/modules/setting/images/icon_singlechoice_nor_dark.png`

Each has 2.0x and 3.0x variants in matching subdirectories.

Base size: 36×36 pixels, RGBA PNG.

Official light-theme selected is system blue RGB(0, 122, 255). To change:
replace blue pixels (where B > max(R,G) + threshold) with target color,
preserving alpha channel.

### Checkboxes (多选按钮)

- `icon_multiplechoice_sel_dark.png` (and variants)
- `icon_multiplechoice.png`

Same structure as radio buttons.

### Toggle switches

- `icon_switch_open.png`: 68×36px, shows green when ON (not blue)
- `icon_switch_close.png`: 68×36px, gray when OFF

If switches appear blue, that's a code-level color override or tint, not the
PNG itself.

## Testing PNG changes

1. Create test PNG with obvious color (white/black/red)
2. Backup original: `sudo cp <original> <original>.original-backup`
3. Replace: `sudo cp <test> <original>`
4. Close WeTypeSettings: `killall WeTypeSettings`
5. Relaunch: `open "/Library/Input Methods/WeType.app/Contents/MacOS/WeTypeSettings.app"`
6. Verify UI change
7. Restore: `sudo cp <original>.original-backup <original>`

No re-signing needed for asset-only changes — the framework signature covers
the whole Resources directory but macOS does not enforce per-file hashes for
resource bundles in practice.

## Batch replacement for production

When a test confirms the right PNG:

1. Modify all variants: base, 2.0x, 3.0x
2. Modify both themes if applicable: light + dark
3. Test on a signed copy first
4. Script the replacements so installs are repeatable

Do not replace only one variant — the OS picks resolution based on display
scaling and will show mismatched colors.

## Other common assets

- Candidate bar: likely in `assets/modules/candidate/`
- V-panel: `assets/modules/v_mode/` or similar
- Main settings chrome: `assets/modules/setting/images/`

Exhaustive asset search:
```bash
find flutter_assets/assets -type f -name '*.png' | grep -i <keyword>
```

Use `sips -g all <file>` for quick PNG inspection on macOS.
