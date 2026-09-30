# Compatibility with other mods

## Principle
Per asset (and per whole `.cfg` file), the last-mounted pak wins outright. Nothing merges. Every override is a
conflict with every other mod shipping that asset, and whichever loses loses *all* its edits to it. So:
- Put logic in NewContent (`runtime-host.md`); override only data the mod cannot work without.
- cfg: patch files only (`config-files.md`), never whole files.
- When an override is unavoidable, rename the override pak `zzz_<Mod>_20_P` so it beats the usual `_P` / `_10_P`
  mods, and document the edit so other authors can merge it.
- Never hard-import another mod's assets from the main mod (it then fails to load without that mod). Optional
  integrations go in a separate optional plugin/pak.

## Checking a conflict
- `tools/pak/scan_mods.py <AssetName> ...` lists every installed container that mentions it and its mount order.
- Extract a container (`UnrealPak <x>.utoc -Extract <scratch>`) and read `zen_names.py` (names) / `--imports`.
- Shared assets seen in the wild: `AnimBP_Player` (ZoneWatch/ZST, WRP-type animation mods, S-Watch),
  `BP_Stalker2Character` (Slop yCam, ImmersiveDialogue ≤ 2.1), `IMC_Exploration` (Immersive HUD, ZST),
  `DA_InputElementsModels` (Controls menu entries: IHUD, ZST), `AnimBP_PlayerWeaponLayer` / `AnimBP_Player_Knife` /
  weapon AnimCollections (Slop yCam), `W_GameHUD` (IHUD), `CoreVariables.cfg` (FOV mods like No Dialogue Zoom).

## Known mods
- **ZST / ZoneWatch** (dannicroax, Nexus 2721): `BP_ZoneWatchSubsystem` → `BP_ZoneWatch` polls keys from
  `QueryKeysMappedToAction`; overrides `AnimBP_Player`, `IMC_Exploration`, `DA_InputElementsModels`; ships a combined
  `AnimBP_Player` as `_30_P` and once gated another mod's block on a per-cook anchor asset
  (`/Game/__ModKitWwiseCookAnchor_<Mod>_<timestamp>__`), which only exists in one build. Lesson: offer a
  **permanent marker asset** with an interface revision in its name (`tools/editor/create_marker.py`) and freeze the
  shared block; never tell others to detect cook anchors.
- **Immersive HUD** (Nexus 1895): NewContent under `/ImmersiveModePlus/` (own IAs, `IMC_ImmersiveHUD`, world
  subsystem `BP_ModWorldImp` → `BP_ModActorImp` with `EnableInput` and EnhancedInputAction events); overrides
  `IMC_Exploration`, `DA_InputElementsModels`, `W_GameHUD`, stat panel assets; **hard-imports MCM**.
- **Slop yCam**: override pak replaces `BP_Stalker2Character` and weapon anim assets, plus a UE4SS part polling
  `PC:IsInStaticDialog`. Hard conflict with any pawn override.
- **UltraPlus / UltraPlusExtensions** (UE4SS Lua): hooks sequence players, toggles DLSSG in cutscenes; suspect in
  soft hangs during sequences (sleep).
- **UObjectCacheMod** (UE4SS Lua): caches objects, rebuilds on transitions; probes must load before it.
- **Better Vaulting, grEdit**: examples of cfg `_patch_` style mods.

## MCM (Mod Configuration Menu, Nexus 2225)
Community settings menu used by several Zone Kit mods. Interfaces seen imported: `/ModConfigurationMenu/BPI_MCM_API`,
`BPI_MCM_SettingsProvider`, `E_MCM_SettingType`; functions `RegisterMCMSettings`, `RegisterModSetting`,
`RegisterDefaultModSetting`, `GetModSetting` (bool / float / int / keybind / combobox), `OnCheckStateChanged`,
`OnSliderValueChange`, `OnMCMButtonPressed`, `AddUniqueModID`. Rule: the main mod keeps settings as variables with
defaults and imports nothing from MCM; a separate optional `<Mod>MCM` plugin implements the provider interface and
writes the values into the main mod's actor. Compiling it needs MCM's interface assets available in the kit (from
its author guide / example mod). Details grow here as a mod implements it.
