# cc-skill-buddy-reroll

A Claude Code skill for rerolling your `/buddy` companion to Legendary (★★★★★).

## How It Works

Claude Code's buddy system uses a deterministic PRNG (Mulberry32) seeded by `hash(userId + salt)` to generate companion attributes. This skill finds alternative salt values that produce Legendary rolls for your account, then patches the Claude Code binary with the new salt.

## Usage

```
/buddy-reroll
```

The skill will:

1. **Preflight** — Verify tools (`bun`, `codesign`, `python3`), locate the binary, extract the current salt, and validate the algorithm against your existing buddy
2. **Gacha Roll** — Generate Legendary seeds and present them for selection. Roll again until you find one you like
3. **Apply** — Backup the binary, patch the salt, re-sign with ad-hoc codesign, and clear the companion for re-hatch
4. **Verify** — Confirm the patched binary runs correctly. Auto-restore on failure

## Tested Version

- Claude Code **v2.1.89** (Mach-O arm64, Bun-compiled)
- macOS (Apple Silicon)

The algorithm (Mulberry32 + `Bun.hash`) and salt pattern have been stable across versions. However, future updates may change these internals.

## Disclaimer

> **USE AT YOUR OWN RISK.** This skill modifies the Claude Code binary. It is provided for **research and educational purposes only**. The author assumes no responsibility for any consequences arising from its use, including but not limited to broken installations, lost data, or account issues. Always ensure you have a backup before proceeding.

## License

[MIT](LICENSE)
