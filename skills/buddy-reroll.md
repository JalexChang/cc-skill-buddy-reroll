---
name: buddy-reroll
description: Reroll your Claude Code /buddy companion to Legendary ★★★★★ by patching the buddy salt in the binary. Interactive gacha-style rolling with safe fallback and auto-restore on failure.
---

# Buddy Reroll

Reroll your Claude Code `/buddy` companion to Legendary (★★★★★).

## Important

- **Verify every assumption** before acting. If anything doesn't match expectations, stop and explain — never proceed blindly.
- **Safe fallback** on any failure — restore backup, report what went wrong.
- Every step must be confirmed before proceeding to the next.
- Known values are used as a fast path. If they fail, fall back to dynamic discovery. If both fail, guide the user to report the issue.
- **Never hardcode values derived from earlier phases** (lengths, offsets, counts). Always compute them programmatically from the source data — manual counting is error-prone.

## Phase 1: Preflight

### 1.1 Tool Check

Verify these tools are available. For any missing tool, help the user install it before continuing.

| Tool | Check | Install hint |
|------|-------|-------------|
| `bun` | `bun --version` | `mise install bun` or `curl -fsSL https://bun.sh/install \| bash` |
| `python3` | `python3 --version` | Usually pre-installed on macOS |
| `codesign` | `which codesign` | Part of Xcode Command Line Tools: `xcode-select --install` |

If any tool is missing and user declines to install, abort with explanation of why it's needed.

### 1.2 Locate Binary

```bash
readlink "$(which claude)" 2>/dev/null
```

- The result should be a Mach-O arm64 executable under `~/.local/share/claude/versions/`
- Verify with `file <path>` — must show `Mach-O 64-bit executable arm64`
- If not found or wrong format, abort: "Cannot locate Claude Code binary."

### 1.3 Extract Current Salt

Locate the salt dynamically from the binary using code context, not a hardcoded value (the salt changes every release).

The salt is assigned to a variable right before the buddy roll function. In the minified binary, the pattern looks like:

```
<SALT_VAR>="<salt_value>",<roll_function_start>
```

Use two structural markers to find it (try both, cross-validate):

**Anchor A — `oauthAccount` (near userId resolution):**

```bash
perl -ne 'while (/(.{0,200}oauthAccount.{0,5}accountUuid.{0,200})/g) { print "$1\n" }' <binary>
```

The salt variable is assigned near the `companionUserId` function (reads `oauthAccount?.accountUuid`, falls back to `userID`, then `"anon"`).

**Anchor B — Rarity array (near buddy roll code):**

```python
import re
with open('<binary>', 'rb') as f:
    data = f.read()
for m in re.finditer(rb'common.*?uncommon.*?rare.*?epic.*?legendary', data):
    # Extract a wide region around the match to find the salt
    chunk = data[m.start():m.start()+3000].decode('ascii', errors='replace')
    print(chunk[:3000])
```

The salt is a quoted string constant assigned just before the buddy roll function returns. In minified code it looks like `VAR="<salt>"`.

**From either anchor's output**, identify the salt: a string literal in the same module scope. It must:

1. Appear exactly 3 times in the binary (JS source is embedded twice + 1 in the string table)
2. Be near the `mulberry32` / buddy roll code
3. NOT be a known keyword (not `"anon"`, `"common"`, `"legendary"`, etc.)
4. If both anchors find a candidate, they must agree — if they don't, abort

If a candidate is found, verify by running a test roll (Phase 1.5) with that salt against the user's current buddy. If the roll matches, the salt is correct.

**Discovery failed?**

If no valid salt is found, abort:

```
❌ Cannot locate buddy salt in the binary.

This skill may no longer work due to changes in the buddy system by Anthropic.
It is NOT recommended to continue using this skill until it has been updated.

If you'd like to help, please report this at:
  https://github.com/VdustR/cc-skill-buddy-reroll/issues

Include: Claude Code version (<version>), OS, and this error message.
```

### 1.4 Read User Identity

```python
import json
with open(os.path.expanduser('~/.claude.json')) as f:
    config = json.load(f)
uuid = config.get('oauthAccount', {}).get('accountUuid')
companion = config.get('companion')
```

- If no `accountUuid`, check `userID`, then fall back to `'anon'`
- If no existing `companion`, note that this is a first hatch (no validation possible, skip 1.5)

### 1.5 Validate Algorithm

Write a temporary Bun script that reproduces the buddy roll algorithm (Mulberry32 PRNG + Bun.hash + weighted rarity pick). The script must:

1. Implement `mulberry32(seed)` — the PRNG
2. Implement `hashString(s)` using `Bun.hash(s)` — NOT the FNV-1a fallback
3. Implement the weighted rarity pick: common=60, uncommon=25, rare=10, epic=4, legendary=1
4. Implement species/eye/hat/shiny/stats roll in the exact same order as the source
5. Roll using `hash(userId + discoveredSalt)` and compare against the user's current companion

**Cross-check:** The rolled `species` must match the companion's system-reminder context (e.g., "A small robot named X" → species is `robot`). If it doesn't match:

- The algorithm, salt, or userId resolution may have changed
- Abort with the same message as salt discovery failure:

```
❌ Algorithm validation failed — rolled [species] but expected [other].

This skill may no longer work due to changes in the buddy system by Anthropic.
It is NOT recommended to continue using this skill until it has been updated.

If you'd like to help, please report this at:
  https://github.com/VdustR/cc-skill-buddy-reroll/issues

Include: Claude Code version (<version>), OS, expected species, and rolled species.
```

The PRNG and roll algorithm should be derived from the source code at the time of execution. If the source repos are available, read the latest version. The known algorithm as of v2.1.89:

```
seed = Bun.hash(userId + salt) & 0xFFFFFFFF
rng = mulberry32(seed)
rarity = weightedPick(rng)          // common:60 uncommon:25 rare:10 epic:4 legendary:1
species = pick(rng, 18 species)     // duck,goose,blob,cat,dragon,octopus,owl,penguin,turtle,snail,ghost,axolotl,capybara,cactus,robot,rabbit,mushroom,chonk
eye = pick(rng, 6 eyes)             // ·, ✦, ×, ◉, @, °
hat = rarity=='common' ? 'none' : pick(rng, 8 hats)  // none,crown,tophat,propeller,halo,wizard,beanie,tinyduck
shiny = rng() < 0.01
stats = rollStats(rng, rarity)      // DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
```

If validation succeeds, display the current buddy info.

**Legendary/Shiny warning:** If the current buddy is already legendary and shiny, include a prominent warning in the roll prompt:

```
⚠️ Your current buddy is already Legendary ★★★★★ ✨SHINY!
   Rerolling will replace your current companion and you may lose your shiny status.
```

### 1.6 Roll Confirmation

After displaying the current buddy info (and the legendary/shiny warning if applicable), ask the user to choose a roll mode:

```
Ready to roll?

  1. ✨ Shiny Legendary (default 3 candidates per batch — takes longer)
  2. 🎰 Legendary (default 10 candidates per batch)
  3. Cancel

Pick a mode (1/2/cancel), or specify a custom batch size (e.g., "shiny 5", "legendary 20").
```

**Batch size limits:**

- If the user requests a batch size > 50, display a warning before proceeding:
  ```
  ⚠️ Batch size [N] is very large.
     Finding this many candidates may take a while, especially in Shiny mode (~0.01% chance).
     Continue? (yes / cancel)
  ```
- The user may adjust batch size at any time during rolling (e.g., "roll 5 more", "next 20").

Wait for explicit confirmation before continuing to Phase 2. If the user says "cancel" / "nevermind", abort cleanly.

## Phase 2: Gacha Roll

### 2.1 Generate Candidates

Write a Bun script that searches for replacement salts. Requirements:

- New salt must be **exactly the same byte length** as the discovered salt — derive the length dynamically from the salt found in Phase 1.3 (do NOT hardcode it)
- Characters must be ASCII printable and binary-safe: `[a-z0-9\-_]`
- Only collect `legendary` rarity results
- **Batch size** depends on the mode chosen in Phase 1.6:
  - **Legendary mode:** default 10 candidates per batch
  - **Shiny mode:** default 3 candidates per batch (only collect results where `shiny === true`)
  - The user may override these defaults (e.g., "shiny 5", "legendary 20")
- **Always reproduce the candidate list in your response text** — do not rely on tool output being visible to the user. After the script runs, re-print the results in your message.
- Display format must include **full stats for all 5 dimensions**:

```
Your current buddy: [species] [rarity] [stars]

🎰 Roll #1:
  1. 🐉 dragon ★★★★★ hat:crown
     DEBUGGING=80 | PATIENCE=55 | CHAOS=72 | WISDOM=100 | SNARK=63
  2. 🐙 octopus ★★★★★ ✨SHINY hat:wizard
     DEBUGGING=90 | PATIENCE=100 | CHAOS=45 | WISDOM=68 | SNARK=71
  3. 🤖 robot ★★★★★ hat:tinyduck
     DEBUGGING=55 | PATIENCE=62 | CHAOS=100 | WISDOM=78 | SNARK=50
  ...

Pick a number, or "roll" to roll again.
```

### 2.2 Interactive Selection

- User picks a number → proceed to Phase 3
- User says "roll" / "again" / "more" → generate another batch (same mode and batch size)
- User says "cancel" / "nevermind" → abort cleanly, no changes made
- User adjusts batch size mid-roll (e.g., "roll 5 more", "next 20") → use the new size for this and subsequent batches
- User switches mode mid-roll (e.g., "switch to shiny", "legendary mode") → switch mode and reset batch size to that mode's default unless user specifies otherwise
- If user asks for a specific species, adjust the search filter

Continue rolling until the user is satisfied. There is no limit.

## Phase 3: Apply

### 3.1 Backup

```bash
cp <binary> <binary>.bak
```

Verify backup:

```bash
md5 <binary> <binary>.bak  # must match
```

If a `.bak` already exists from a previous run, ask user: "Existing backup found. Overwrite or keep?"

### 3.2 Patch

Use Python for binary-safe replacement:

```python
with open(binary_path, 'rb') as f:
    data = f.read()
old_salt = discovered_salt.encode()
new_salt = chosen_salt.encode()
assert len(old_salt) == len(new_salt)
patched = data.replace(old_salt, new_salt)
assert len(patched) == len(data)  # size must not change
with open(binary_path, 'wb') as f:
    f.write(patched)
```

**CRITICAL:** Do NOT use `perl -pi -e`, `sed`, or any text-mode tool on the binary. They will corrupt it.

### 3.3 Re-sign

```bash
codesign --force --sign - <binary>
```

Verify:

```bash
codesign -v <binary> 2>&1
```

Must not show errors.

### 3.4 Verify Launch

```bash
timeout 5 claude --version 2>&1
```

- If it outputs a version string → success
- If it fails, crashes, or is killed → **auto-restore:**
  ```bash
  cp <binary>.bak <binary>
  ```
  Report: "Patched binary failed to launch. Restored from backup. [error details]"

### 3.5 Clear Companion

Remove the stored companion soul from config to trigger re-hatch:

```python
import json
config_path = os.path.expanduser('~/.claude.json')
with open(config_path) as f:
    config = json.load(f)
config.pop('companion', None)
config.pop('companionMuted', None)
with open(config_path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
    f.write('\n')
```

### 3.6 Done

```
✅ Patched! Restart Claude Code and run /buddy to hatch your new companion.

Expected: [species] ★★★★★ [shiny?] hat:[hat]

Restore command (if needed):
  cp <binary>.bak <binary>

⚠️ Note: Claude Code auto-updates will replace the binary.
   Re-run /buddy-reroll after updates to re-apply.
```

## Error Handling Summary

| Failure | Action |
|---------|--------|
| Missing tool | Help install, abort if declined |
| Binary not found | Abort with explanation |
| Known salt not found | Fall back to dynamic discovery |
| Dynamic discovery also failed | Abort + suggest filing issue |
| Algorithm validation mismatch | Abort + suggest filing issue |
| File size changed after patch | Abort + restore backup |
| Codesign failed | Restore backup |
| `claude --version` failed | Auto-restore backup |
| Config write failed | Report error, binary patch is still valid |

For any failure that suggests the buddy system internals have changed, direct the user to:

```
https://github.com/VdustR/cc-skill-buddy-reroll/issues
```

Include: Claude Code version, OS, error details.
