# ImmersiveReloading — mod context for the agent

**First read `harness/AGENTS.md`**: the shared rules, tester workflow, pipeline and game knowledge for every
Stalker 2 mod. They apply here. This file holds only what is specific to this mod.

Purpose: Sprint and reload at the same time, with an optional reload-speed setting.

Repo `noahsemus/stalker2-immersive-reloading`, Zone Kit plugin `ImmersiveReloading` (`/ImmersiveReloading/`), asset
prefix `ImmReload`. Created from the harness on 2026-09-30.

## Current state (one line per checkpoint / release: date, what changed, what the tester confirmed in game)
- 2026-09-30: repo created; research done, plan written (`PLAN.md`). Waiting on the tester: pick `ImmersiveReloading`
  in the editor's toolbar mod selector once (GameFeatureData), the three decisions in PLAN.md, vanilla behaviour
  (Q1) and the `ApplyEffectComponent` Details check (Q2). Then build 1.

## Key facts (verified; details and quotes in `zonekit/research/2026-09-30-sprint-reload-research.md`)
- Sprint vs reload is native: no cfg field; `IA_Sprint` / `IA_Reload` (`/Game/_Stalker_2/data/input/InputActions/Delayable/`)
  have no triggers or modifiers (editor Python, 2026-09-30). A sprint-pose reload is impossible pak-only; the plan is
  a "combat sprint" (run pose at sprint speed while reloading).
- Reload in progress: `PC.Is Action Active(ActionType::RELOAD_WEAPON)`; `Obj.Get Current Reload State`.
- Speeds `ObjPrototypes.cfg` Player `MovementParams`: run 370, jog 625, sprint 820 (`:716-727`); sprint stamina
  `StaminaPerAction.Sprint = 5.25` (`:666`), threshold 16.1 (`:694`).
- Sprint block by data: `EEffectType::BlockAnimationActionType` + `EActionType::Sprint` (`EffectPrototypes.cfg:24719`
  `BlockSprintEffect`, `:25029` `ConcussionBlockSprint` 3 s).
- Reload speed by data: `EEffectType::ReloadingTime`, negative % = faster (`EffectPrototypes.cfg:5556`
  `ReloadingTimeDecBy25`). Per-weapon `WeaponReloadTimePerAttachment` (`WeaponData/WeaponGeneralSetupPrototypes.cfg`)
  is avoided: OXA patches it and it likely affects NPCs.
- Applying an effect from Blueprint is untested: route A `ApplyEffectComponent` (`bp_api_dump2.txt:1582`), route B
  `CppMediator.start_quest_node` (`:2233`) on a `SetCharacterEffect` node.
- MCM 2.0 is installed on the dev box; provider = any actor implementing `BPI_MCM_SettingsProvider`
  (`harness/docs/compatibility.md` § MCM).

## Assets this mod may override
- None planned. Never `AnimCollection_fp_*` (OXA overrides 38), `AnimBP_Player` (ZST), the pawn (ImmersiveDialogue).

## Files
- `PLAN.md` plan, builds and test matrix · `BUILD.md` every edit asset by asset · `zonekit/README.md` engineering
  log · `zonekit/research/` research reports · `zonekit/ImmersiveReloading/` plugin mirror · `zonekit/tools/`
  mod-specific generators and classifier lists · `zonekit/builds/` checkpoint and release paks · `mod.json`.
- `CLAUDE.md` / `GEMINI.md` only import `harness/AGENTS.md` and this file, for tools that don't read `AGENTS.md`.

## Release (only when the tester says "cut a release")
Follow `harness/docs/pipeline.md` § Release. Nexus page: _(record the mod id here once it exists)_.
