# adb-agent-skill

A Claude Code / Anthropic skill for controlling an Android device via ADB — UI navigation, app interaction, messaging, voice messages, file transfer, alarm management.

## Install

Drop into your skills directory:

```bash
git clone https://github.com/<you>/adb-agent-skill ~/.claude/skills/adb-automation
```

Or symlink from this checkout:

```bash
ln -s "$PWD" ~/.claude/skills/adb-automation
```

## Requirements

- `adb` (`brew install android-platform-tools`)
- `python3` (UI hierarchy parsing)
- `edge-tts` + `ffmpeg` (voice messages only)
- Android device with USB debugging enabled

## Contents

- `SKILL.md` — main skill reference (loaded by the agent on demand)
- `references/commands.md` — extended ADB command reference

## License

MIT
