# AutoHunt v2 (MQ Lua) - README & Command Guide

## Overview

`autohunt.lua` is an advanced auto-target / semi-pull hunter for EverQuest (MacroQuest Lua).

It provides:

- Weighted multi-factor target scoring (instead of simple rigid tiers).
- Priority and avoid lists.
- Optional pull mode with MQ2Nav integration.
- Group-aware target selection (MA assist and defensive behavior).
- Safety automation (PC/GM suspend, adds check, timeouts).
- Profile save/load system.
- Themed ImGui control panel (Maui theme bridge compatible).

## Important

- Use only where automation is allowed by your server rules.
- This script can issue targeting/navigation/combat commands automatically.

## Files Used

- Script:
  - `C:\Users\E9ine\AppData\Local\VeryVanilla\Emu\Release\lua\autohunt.lua`
- Settings:
  - `<MQ configDir>/autohunt_settings.lua`
- Priority list storage:
  - `<MQ configDir>/autohunt_lists.lua`
- Profiles:
  - `<MQ configDir>/autohunt_profiles.lua`
- Error log:
  - `<MQ configDir>/autohunt_error.log`
- Optional list export/import text:
  - `<MQ configDir>/autohunt_export.txt` (default if no path provided)

The script also writes timestamped `.bak.lua` backups before overwriting settings/lists/profiles.

## Quick Start

1. Run script:
   - `/lua run autohunt`
2. Open UI:
   - `/autohunt ui`
3. Add priorities:
   - In `Priority` tab or via command `/autohunt add <pattern>`
4. Enable pulling if needed:
   - `/autohunt pull on`
5. Save config:
   - `/autohunt save`

## How Target Selection Works

AutoHunt scans nearby NPCs, filters invalid candidates, then scores each candidate with weighted factors.

### Base Candidate Filters

A spawn is rejected if:

- Not NPC.
- Dead, pet, charmed, mezzed, fleeing.
- HP outside `min_hp_pct` / `max_hp_pct`.
- Has a valid `Master` (pet-like ownership).
- Fails LOS check when `check_collision` is enabled.

### Weighted Scoring Factors

Each factor contributes points with configurable weights:

- Priority list rank.
- Named tier (rare named patterns can score higher).
- Distance (non-linear closer bonus).
- Level delta.
- Current HP%.
- Difficulty estimate (from MaxHP heuristic).
- Aggro relevance (xtarget/aggressive/recent damage cache).
- Body type bonus/penalty.
- Recent-kill penalty (weak anti-repull cache).
- Loot heuristic (basic named/level bias).
- Group defense logic (MA assist, low-HP member pressure, defensive mode).
- Optional custom Lua scoring rules.

Avoid list applies a massive negative score and effectively blacklists those mobs.

### Why a target was chosen

The script tracks and shows reason text:

- UI `Status` tab (`Reason` line)
- Live best line in toolbar/status
- Chat log when targeting

## Pull Mode

When enabled and idle:

- AutoHunt searches using `pull_radius`.
- Selects best candidate from pull range.
- If MQ2Nav is loaded, navigates toward `pull_approach_distance`.
- Fires `pull_command` when in range.
- Falls back to `/face` + `/stick id ... loose 50` when nav is unavailable and fallback is enabled.

Pull auto-cancels on:

- Timeout
- Missing/dead pull target
- Zone change
- Combat entering

## Group-Aware Features

Options available in settings:

- `assist_group_ma`
- `only_aggro_on_group`
- `prioritize_attacker_on_lowest_hp`
- `defensive_mode`
- `defensive_hp_threshold`
- `squishy_classes` map

Behavior examples:

- Bonus when candidate is MA’s target.
- Bonus when candidate is attacking lowest-HP member.
- Stronger defensive bonus for squishy/healer pressure under threshold.

## Safety Features

Implemented safety controls:

- Suspend on nearby PC/GM (`safety_suspend_on_pc`, `safety_suspend_on_gm`).
- Adds gate (`safety_avoid_adds`, radius + threshold).
- Max combat duration guard (`safety_max_combat_time_sec`).
- No-target timeout action (`safety_no_target_timeout_sec`, command).
- Optional mana suspend behavior for caster workflows.
- Optional humanized action delays.
- Fast target-switch warning detection.

## Profiles

Profiles store both settings and list state.

Use cases:

- `named_farm`
- `trash_grind`
- `zone_specific`
- `dailies`

Commands:

- Save: `/autohunt profile save <name>`
- Load: `/autohunt profile load <name>`
- List: `/autohunt profile list`
- Delete: `/autohunt profile delete <name>`

## Priority / Avoid Lists

### Priority list

- Higher rows are preferred.
- Supports pattern matching (`string.find` plain match).
- Optional `min_hp` gate and `required` behavior per row.

### Avoid list

- Patterns in `avoid_actors` are heavily penalized (effectively skipped).

## Full Command Reference

### Core

- `/autohunt help`
- `/autohunt on`
- `/autohunt off`
- `/autohunt toggle`
- `/autohunt ui`
- `/autohunt save`
- `/autohunt load`

### Priority list

- `/autohunt list`
- `/autohunt add <pattern>`
- `/autohunt addtarget`
- `/autohunt del <index>`
- `/autohunt up <index>`
- `/autohunt down <index>`

### Avoid list

- `/autohunt avoid add <pattern>`
- `/autohunt avoid del <index>`
- `/autohunt avoid list`

### Mode and ranges

- `/autohunt mode zone`
- `/autohunt mode global`
- `/autohunt mode named <key>`
- `/autohunt radius <n>`
- `/autohunt zradius <n>`
- `/autohunt delay <seconds>`
- `/autohunt minhp <0-100>`
- `/autohunt maxhp <0-100>`

### Pull mode

- `/autohunt pull on`
- `/autohunt pull off`
- `/autohunt pull toggle`

### Profiles

- `/autohunt profile save <name>`
- `/autohunt profile load <name>`
- `/autohunt profile list`
- `/autohunt profile delete <name>`

### List export/import

- `/autohunt exportlist [path]`
- `/autohunt importlist [path]`

## UI Guide

Tabs:

- `Settings`
  - Core, Scoring, Puller, Group, Safety, Visuals sections.
- `Priority`
  - Search/filter, list row management, avoid add.
- `Profiles`
  - Save/load/delete/list profile controls.
- `Status`
  - Current mode, best target line, reason text, pull state, last target info.

Top toolbar:

- Enable/Disable
- Force Scan
- Add Current Target
- Save
- Reload
- Close

## Custom Scoring Rules

`custom_score_rules` supports expressions returning numeric score delta.

Example entry (in settings file):

```lua
custom_score_rules = {
  { name = 'undead bonus', expr = 'return (spawn.Body.Name() == "Undead") and 25 or 0', enabled = true },
}
```

Notes:

- Rules execute in a lightweight env with `spawn`, `candidate`, `mq`, `T`, `state`, `settings`.
- Keep rules simple for performance.

## Performance Notes

- Scan cadence is bounded by `rescan_delay`.
- Recent-damage cache updates are throttled.
- Scoring is done once per candidate per scan.
- Heavy logic should be avoided in custom rules.

## Troubleshooting

### Script exits on startup

Check:

- `<MQ configDir>/autohunt_error.log`

### Pull mode not navigating

Check:

- MQ2Nav plugin is loaded.
- Target is reachable.
- Pull radius and approach distance are reasonable.

### Over-targeting / target thrash

Tune:

- `switch_score_margin`
- `rescan_delay`
- safety and aggro-group options

## Suggested Baseline Presets

### Safe Trash Grind

- pull_mode: off
- only_aggro_on_group: true
- safety_avoid_adds: true
- switch_score_margin: 15

### Named Farm

- pull_mode: on
- higher `named_tier` weight
- moderate distance weight
- keep avoid list populated

### Box Defense

- assist_group_ma: true
- defensive_mode: true
- lower defensive threshold (40-50)
- high group_defense + aggro weights
