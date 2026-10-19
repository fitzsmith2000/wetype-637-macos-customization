# WeType 637 macOS Customization

A Hermes/Claude skill for customizing WeType 2.2.1 (Build 637) on macOS — candidate bar, V-panel, and settings interface "黑白" (white-on-black) theme.

## What This Skill Does

- **Candidate bar**: White selected background + black text
- **V-panel** (clipboard/常用语/符号): Black text on white highlight
- **Settings interface**: Dark recessed switches + white fill buttons with black text
- **Complete workflow**: Build → Sign → Verify → Install → Test → Rollback

## Version

- **Skill version**: v1.1.0
- **WeType target**: 2.2.1 (Build 637)
- **Compatibility**: arm64 + x86_64 universal binary

## What's Included

### Core Skill
- `SKILL.md` — Complete workflow, pitfalls, and troubleshooting (20KB)

### References
1. `v-panel-and-settings.md` — V-panel and settings color mapping
2. `flutter-assets.md` — PNG asset replacement (radio buttons, etc.)
3. `637-installation-and-troubleshooting.md` — TIS diagnosis, input menu recovery, rollback
4. `637-xcode-signing.md` — Apple signing, certificate recovery, entitlements
5. `637-deep-customization.md` — Adding new customizations, adapting future builds

## Quick Start

### Prerequisites
- macOS with Xcode installed
- Hermes Agent or Claude desktop app
- WeType 2.2.1 (637) official package
- Apple Development signing identity

### Installation

1. Load this skill in Hermes:
   ```bash
   # From Hermes profile directory
   cd ~/.hermes/profiles/YOUR_PROFILE/skills/software-development/
   git clone https://github.com/fitzsmith2000/wetype-637-macos-customization.git
   ```

2. Or use it directly with Claude:
   - Copy the skill directory to your project
   - Reference `SKILL.md` when working on WeType customization

### Usage

Ask your AI assistant:
```
使用 wetype-637-macos-customization skill 帮我定制 WeType 637 的候选栏颜色
```

The skill will guide you through:
1. Identifying your WeType bundle
2. Building the customized bundle
3. Signing with your Apple identity
4. Verifying the result
5. Installing safely with backup
6. Testing in multiple apps
7. Rollback if needed

## Key Features

### Canonical 7-Step Workflow
Clear procedure from package identification to testing

### Comprehensive Bug Triage
Structured diagnosis for common failures:
- Missing input menu
- White-on-white text
- Signature failures
- Cross-app compatibility

### Battle-Tested Pitfalls
20+ real-world issues and solutions, organized by category:
- Architecture and encoding
- V-panel and candidate bar
- Flutter settings colors
- System blue handling
- LSUIElement preview
- Asset-only changes

### Safety First
- Never modifies installed app directly
- Always creates timestamped backups
- Verifies signatures before installation
- Provides rollback instructions

## What's NOT Included

This repository intentionally excludes:
- WeType binary files or bundles
- Personal signing identities or certificates
- Team IDs or entitlements with personal info
- User dictionaries or caches
- Scripts (refer to your local work directory)

See `references/637-deep-customization.md` for adding your own patches.

## Compatibility Notes

- **Fully validated**: WeType 2.2.1 (637)
- **Workflow reusable**: Other builds (offsets need re-derivation)
- **macOS**: Tested on macOS 14+
- **Architecture**: Universal arm64 + x86_64

## Legal & Distribution

- This skill is for **local testing only**
- Apple Development signatures are **not** for distribution
- Original Tencent Developer ID signature provides full host trust
- Share the skill, not the binaries

## Contributing

Improvements and new findings welcome:
1. Test your changes thoroughly
2. Document new offsets in references
3. Update pitfalls with evidence
4. Submit PR with verification steps

## Related Projects

- [WeType 657 skill](https://github.com/LKRCharon/kairong-skills/tree/main/skills/wetype-macos-customization) (parent workflow)
- [Hermes Agent](https://hermes-agent.nousresearch.com/)

## License

Skill source and documentation: MIT

WeType is property of Tencent. This skill is for personal customization only.

---

**Status**: Maintained | **Last Updated**: 2026-10-19
