# Game data (.cfg) files

All gameplay data (items, weapons, NPCs, effects, abilities, input contexts, core variables) lives in text `.cfg`
files under `<kit>\Stalker2\Content\GameLite\GameData\` in GSC's struct DSL (`Name : struct.begin {refkey=...}`
... `struct.end`, inheritance through `refurl` / `refkey`). Grep them first for any gameplay number.

## Rules
- **Never ship a whole-file override** of a game `.cfg`. It silently replaces every other mod's edits to the same
  file (ImmersiveDialogue 2.0.0 was re-cut once because a full `CoreVariables.cfg` had slipped into the pak).
- Change data with **patch files** that touch only the fields you need (the mechanism grEdit, Better Vaulting and
  ImmersiveDialogue's talk-distance plan use). Syntax and placement: see "Patch files" below.
- Offer different values as optional pak variants (e.g. `Optional-Fast/`), or at runtime through Blueprint if a
  setter exists (then an MCM slider becomes possible).
- A mod's own cfg folder comes from its GameFeatureData action AddConfigsPath:
  `Content/GameLite/ModGameData/<Mod>/`.
- Data reached through `refkey` inheritance: check that a patch on the base prototype actually reaches the
  children that override the same field (they keep their own value).

## Patch files
_(to be filled in with verified syntax and placement; see the Immersive Reloading research, 2026-09-30)_
