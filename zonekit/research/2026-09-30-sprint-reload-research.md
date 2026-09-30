# Immersive Reloading — research report (2026-09-30)

Read-only research. Sources: Zone Kit GameData cfgs (`G:\Epic Games\STALKER2ZoneKit\Stalker2\Content\GameLite\GameData\`, abbreviated `GD\`), `bp_api_dump.txt` / `bp_api_dump2.txt` (kit root), the installed `~mods` (listed/extracted into this scratchpad: `paklists\`, `utoclists\`, `ext\`, `zen\`), the kit's editor pak index (`fulleditor_pak_index.txt`, 143,048 entries), sibling repo logs, and the web (Nexus pages return 403 to the fetcher; search snippets only).

Confidence scale: **High** = read directly in a file; **Medium** = strong inference from several files; **Low** = plausible, untested.

---

## 1. What stops sprint + reload today

### 1.1 Observed behaviour (players)
- Steam thread "reload animations not interruptable??" (https://steamcommunity.com/app/1643320/discussions/0/4626981323674781944/): "Sprinting cancels them"; firing and opening the backpack also cancel a reload; an interrupted reload restarts from the beginning.
- Web search summary of the Nexus forum request "Sprint + Heal/Reload Mod for S.T.A.L.K.E.R. 2" (https://forums.nexusmods.com/topic/13503027-sprint-healreload-mod-for-stalker-2/): "if you are sprinting and press reload, it will slow you down to run, and if it's the other way around the reload will get cancelled." (page itself 403; snippet only.)
- No existing Nexus mod does sprint-while-reloading (searches found only faster-reload and stamina mods).

So, as far as the public reports go: **reload -> sprint pressed = reload interrupted, sprint starts; sprint -> reload pressed = sprint drops to run, reload plays.** To be confirmed by Noah (section 6). Confidence: Medium.

### 1.2 Cfg: no field governs it (High)
Keyword sweep over all of GameData (`CanReload`, `WhileSprint`, `DuringSprint`, `bCanSprint`, `StaminaSprint`, `ActionBlock`, `InterruptReload`, `ReloadInterrupt`, `BlockReload`, `SprintBlock`, `CancelReload`, `AllowSprint`, `AllowReload`): **0 files each**. `Interrupt`/`Cancel` fields that exist are all AI / contextual-action fields (`CanBeInterrupted`, `ContinueAfterInterrupt`, `CanInterruptByCombat`, ...). `Block*` fields are meshes/grooms/upgrades plus the effect lists below.

What the cfg *does* have about sprint is only gating of sprint itself:

`GD\ObjPrototypes.cfg:653` Player prototype, `:694-713`:
```
      StaminaDisableThresholds : struct.begin
         [0] : struct.begin
            // Threshold of stamina at which current settings are applied
            Threshold = 16.1
            RegenerationDelay = 1
            // Blocked actions during current state of configuration
            StateTags : struct.begin
               [0] = EStateTag::Sprint
```
`GD\ObjPrototypes.cfg:760-771` (overweight blocks movement tags incl. `EStateTag::Sprint`), `MovementParams` `SprintSpeed = 820`, `RunSpeed = 370`, `JoggingSpeed = 625` (`:716-727`), `StaminaPerAction.Sprint = 5.25` (`:666`).

Effect type that blocks an *action*, proven on the player (heavy exoskeletons without the sprint upgrade):
`GD\EffectPrototypes.cfg:24709-24728`
```
ArmorConditionalEffect : struct.begin {refkey=[0]}
   SID = ArmorConditionalEffect
   Type = EEffectType::Conditional
   ConditionSID = TargetHasAddSprintEffect
   TrueEffectSID =
   FalseEffectSID = BlockSprintEffect
   ...
BlockSprintEffect : struct.begin {refkey=[0]}
   SID = BlockSprintEffect
   Type = EEffectType::BlockAnimationActionType
   Positive = EBeneficial::Negative
   bIsPermanent = true
   DuplicationType = EDuplicateResolveType::KeepOld
   BlockAnimationTypes : struct.begin
      [0] = EActionType::Sprint
   struct.end
struct.end
```
Also `ConcussionBlockSprint` (`:25029`, Duration = 3, timed) and `BlockAnimationActionTypeEffect` (`:11810`, blocks HandleAimInput/Sprint/Crouch/Fall/Vault/Lean/AutoCover). `EActionType` values used anywhere in cfg: AutoCover, Crouch, Fall, HandleAimInput, Jogging, Lean, Sprint, ThrowItem, UseMainItem, Vault. **No `EActionType::Reload` appears in any cfg** (the native enum may still have one; the dumps do not list `ActionType` values).

Relevant only for fake-sprint stamina: `GD\CoreVariables.cfg:40-58` `StaminaRegenStateCoefs` (`EStateTag::SprintUnderRunSpeed` -0.65, `EStateTag::Sprint` -1.4, ...). Sprint input mode: `GD\SettingsVariablesPC.cfg:13` `SprintInputType = EPlayerActionInputTypeCustom::ToggleOrHold`.

### 1.3 Input assets: separate actions, nothing linking them (High for layout, Medium for triggers)
- `IA_Sprint` and `IA_Reload` are ordinary Enhanced Input actions: `Stalker2/Content/_Stalker_2/data/input/InputActions/Delayable/IA_Reload.uasset`, `.../Delayable/IA_Sprint.uasset` (editor pak index). Both mapped in `IMC_Exploration` (`GD\InputMappingContextPrototypes.cfg:10-17`, priority Lowest). No cfg mentions either except PDA tutorials (`GD\PDATutorialPrototypes.cfg:94,177,789,922`).
- The only asset-side veto class is `InputTriggerActionBlocker` (`bp_api_dump2.txt:3671`) with conditions `TriggerStateActionBlockerCondition` (`:6980`, `trigger_state (PlayerTriggerState)`), `UserSettingActionBlockerCondition` (`:7200`), `MotionAim...`, `JournalTransition...`, `MapTransition...`, `PlayGoInProgress...`. A `TriggerStateActionBlockerCondition(RELOAD_TRIGGER)` on `IA_Sprint`'s triggers would be exactly this rule — **unchecked for IA_Sprint/IA_Reload**. The campfires log (`stalker2-immersive-campfires\zonekit\README.md:26`) found no blockers on the IAs it checked, but that list was the PDA/inventory/quick-slot rows. There is no `ReloadIPU`; `SprintIPU` exists with no BP members (`bp_api_dump2.txt:6593`), i.e. native input processing.

### 1.4 Player state machine is native (High)
`PlayerTriggerState` (`bp_api_dump.txt:5084-5140`) includes `RELOAD_TRIGGER`, `SPRINT_TRIGGER`, `SPRINT_STARTED_TRIGGER`, `UNLOAD_TRIGGER`, `UNJAM_TRIGGER`, `LADDER_SPRINT_TRIGGER`. Anim notifies that drive it from montages:
```
bp_api_dump.txt:252  ==== AnimNotify_PlayerAction
  [prop] player_action_result  -- (PlayerActionResult):  [Read-Only]      (END / INTERRUPT, :4614)
  [prop] player_action_type  -- (ActionType):  [Read-Only]
  [prop] player_action_type_to_override  -- (ActionType):  [Read-Only]
  [prop] player_trigger_to_override  -- (PlayerTriggerState):  [Read-Only]
bp_api_dump.txt:259  ==== AnimNotify_PlayerActionTrigger
  [prop] action_trigger  -- (PlayerTriggerState):  [Read-Only]
  [prop] trigger_new_state  -- (PlayerActionTriggerState):  [Read-Only]   (ACTIVATE / DEACTIVATE, :4618)
bp_api_dump2.txt:1185 ==== AnimNotifyState_ForceReloadingEnd   [prop] is_unloading
bp_api_dump2.txt:1487 ==== AnimNotify_WeaponAction  [prop] magazine_reload_state, reload_transition_rules, weapon_action, ...
```
Anim-side flags are native-computed, read-only: `AnimPlayerTransitionData.can_enter_sprint` (`bp_api_dump.txt:513`), `sprinting_override` (`:499`).

Relevant native members on the player (all Python-exposed, i.e. BlueprintCallable/Pure or BP events; `Obj` = base of `PC`):
```
bp_api_dump2.txt:4828  Obj.can_enter_to_sprint() -> bool
bp_api_dump2.txt:4829  Obj.can_reload_with_transition_rules() -> bool
bp_api_dump2.txt:4853  Obj.finish_reload()
bp_api_dump2.txt:4875  Obj.get_current_reload_state() -> MagazineReloadState
bp_api_dump2.txt:4926  Obj.interrupt_reload()
bp_api_dump2.txt:4958  Obj.is_reload_available(unloading=False) -> bool
bp_api_dump2.txt:4963  Obj.is_should_sprint() -> bool
bp_api_dump2.txt:4967  Obj.is_sprinting() -> bool
bp_api_dump2.txt:5025  Obj.on_reload_bp()                 (BP event: only an override of BP_Stalker2Character could use it)
bp_api_dump2.txt:5030  Obj.on_sprint_pressed() / :5031 on_sprint_pressed_bp()
bp_api_dump2.txt:5042  Obj.perform_sprint_jogging_pause(should_update_movement_speed)
bp_api_dump2.txt:5049  Obj.reload() / :5050 reload_weapon()
bp_api_dump2.txt:5081  Obj.set_speed_multiplier(speed_multiplier) / :4912 get_speed_multiplier()
bp_api_dump2.txt:5095  Obj.update_movement_speed()
bp_api_dump2.txt:4861  Obj.force_set_sp(sp) / get_sp() / get_max_sp()
bp_api_dump2.txt:5210  PC.is_action_active(action_type) -> bool
bp_api_dump2.txt:5295  PC.on_sprint_released() / on_sprint_released_bp()
bp_api_dump2.txt:5328  PC.process_player_action_notify(action_result, action_type)
bp_api_dump2.txt:5350  PC.set_forbidden_movement_types(...) / set_allowed_movement_types(...)   (PlayerMovementType has no SPRINT value, bp_api_dump.txt:5050)
bp_api_dump2.txt:5380  PC.velocity_multimplier (float) [Read-Write]
bp_api_dump2.txt:909   Agent.is_reload_animation_active()  (NPC class, not the player)
```

### 1.5 Assets: ~540 first-person reload montages (High)
The editor pak index lists 541 player first-person reload montages (`MG_*reload*` under `_STALKER2/Animations/Player/AnimSequences/<type>/<weapon>/Reload/` and attach folders), per-weapon `AnimCollection_fp_*` data assets (`PlayerFirearmAnimCollection`, `bp_api_dump.txt:4887`), and the player layer graph `Animations/Player/AnimLayers/AnimBP_PlayerWeaponLayer.uasset` + `AnimLI_PlayerWeapon*.uasset`, plus 7 per-weapon `AnimBP_Player<Weapon>Layer` assets. Any notify-level edit (e.g. stripping an `AnimNotify_PlayerAction(INTERRUPT)`) would be ~540 montage overrides and would still not make the sprint pose play a reload.

### 1.6 Verdict
**Native C++.** The sprint/reload exclusion lives in the player's native action/trigger system (`ActionType`, `PlayerTriggerState`, `can_enter_to_sprint`, `interrupt_reload`), not in cfg. The only asset-side lever (an `InputTriggerActionBlocker` on `IA_Sprint`) is unverified and, even if present, only covers "reload blocks sprint", not "sprint cancels reload". Confidence: Medium-High (native classes have no graph; exact C++ checks unreadable).

**Feasible pak-only substitute: a "combat sprint".** Keep the vanilla run pose while reloading, but move at sprint speed and drain stamina like a sprint:
1. While a reload is in progress, block the native sprint so it can't interrupt the reload: apply an effect of `EEffectType::BlockAnimationActionType` / `EActionType::Sprint` (vanilla `BlockSprintEffect`, proven on the player via exoskeletons), or a timed copy like `ConcussionBlockSprint` (Duration 3).
2. While the sprint key is held during the reload, raise speed with `Obj.set_speed_multiplier(820/370 ≈ 2.2)` or `PC.velocity_multimplier`, and drain stamina with `force_set_sp` at the `StaminaPerAction.Sprint` rate. Restore both when the reload ends.
3. Reload detection from a mod actor: bind `IA_Reload` (Started) as an EnhancedInputAction event (IHUD / ImmersiveDialogue pattern), then poll `get_current_reload_state()` or `PC.is_action_active(<Reload>)` every ~0.1-0.25 s until it ends; `IA_Sprint` bound the same way for "held".

Host: ModWorldSubsystem NewContent actor (overrides nothing). Effect application from BP: section 2.3.

---

## 2. Reload speed

### 2.1 Per-weapon cfg multipliers (High)
`GD\WeaponData\WeaponGeneralSetupPrototypes.cfg:136-155` (TemplateWeapon):
```
//Reload
   PerBulletReloadingAmmoCount = 0 // Zero means full mag reload
   AdditionalBulletsAfterReloadingCount = 0
   WeaponReloadTimePerAttachment : struct.begin
      [0] : struct.begin
         AttachPrototypeSID = empty
         TacticalReloadTimeMultiplier = 1.0
         FullReloadTimeMultiplier = 1.0
         SingleBulletReloadTimeMultiplier = 1.0
         TwinReloadTimeMultiplier = 1.0
         TwinTacticalReloadTimeMultiplier = 1.0
         TwinAuxReloadTimeMultiplier = 1.0
         TwinTacticalAuxReloadTimeMultiplier = 1.0
         UnloadTime = 0.0
      struct.end
   struct.end
   ReloadTypes : struct.begin
      [0] = EAnimationReloadTypes::Full{bskipref}
   struct.end
```
Every weapon (92 prototypes, `GunPM_HG` at `:1147` ... `Default` at `:18037`) re-declares its own array keyed by magazine attachment, all 1.0, e.g. `GunAK74_ST` (`:4058`, reload block from `:4113`):
```
   WeaponReloadTimePerAttachment : struct.begin
      [0] : struct.begin
         AttachPrototypeSID = GunAK74_MagDefault
         TacticalReloadTimeMultiplier = 1.0
         FullReloadTimeMultiplier = 1.0
         UnloadTime = 0.0
      struct.end
      [1] : struct.begin
         AttachPrototypeSID = GunAK74_MagIncreased
      ...
      [2] : struct.begin
         AttachPrototypeSID = GunAK_MagPaired
         TwinReloadTimeMultiplier = 1.0 ...
```
Counts in the file: 92× Tactical/Full, 14× Twin, 12× TwinTactical/TwinAux/TwinTacticalAux, 5× SingleBullet. The only non-default values are `UnloadTime = 1.0` on shotguns (`:8304, :8563, :9152`). Because each weapon overrides the whole array, patching `TemplateWeapon` alone won't reach them; a preset has to patch every weapon's entries.
Nexus mods confirm lower = faster: "Fast Reloading and Aiming" (mod 1746) sets `TacticalReloadTimeMultiplier = 0.35` / `FullReloadTimeMultiplier = 0.35`; "Sota - Faster Reload ADS Equip (bpatch)" (mod 1919) uses 0.8; "Fastest weapon reload" (1566) 0.7 (search-result summaries; pages 403). Confidence: Medium-High.
`WeaponGeneralSetupPrototypes` is probably shared by NPC and player weapons (NPC/Player split lives in `CharacterWeaponSettingsPrototypes/*`, which has no reload fields), so this route likely speeds NPC reloads too. Confidence: Medium.

### 2.2 Global/player effect: `EEffectType::ReloadingTime` (High)
`GD\EffectPrototypes.cfg:5556-5574`:
```
ReloadingTimeDecBy25 : struct.begin {refkey=[0]}
   SID = ReloadingTimeDecBy25
   EffectLevel = EEffectLevel::VeryLow
   Text = Descrease ReloadingTime
   Type = EEffectType::ReloadingTime
   ValueMin = -25%
   ValueMax = -25%
   bIsPermanent = true
   Positive = EBeneficial::Positive
struct.end
ReloadingTimeDecBy25_Temp : ... Type = EEffectType::ReloadingTime  ValueMin = -25%  ValueMax = -25%  Duration = 5.0
```
Live on the player through the sleepiness mechanic: `GD\EffectPrototypes.cfg:7016-7033` (`MediumSleepinessReloadingDecrease` +15%, `StrongSleepinessReloadingDecrease` +18.75%, both `Type = EEffectType::ReloadingTime`, `Positive = EBeneficial::Negative`), applied from the SleepMechanics `ApplicableEffects` lists (`:3882`, `:3919`). So negative % = faster, and a player-level ReloadingTime effect is honoured by native code. OXA (installed) also defines `ReloadTime_Minus4..8` (`ext\OXA_StandardA_P\EffectPrototypes\OXAEffects_WeaponHandling.cfg:668-690`) and attaches them to magazines via `EffectPrototypeSIDs` (`ItemPrototypes\OXAAttach_Magazine.cfg`), which confirms the effect also works at attachment level.
No vanilla upgrade, perk or ability uses ReloadingTime (grep of `UpgradePrototypes.cfg`, `AbilityPrototypes.cfg`: 0 hits). No global reload variable in `CoreVariables.cfg` (only a Gauss VFX path at `:868`).

### 2.3 Runtime (MCM slider) options
There is no BP-callable "set reload speed". Candidates, in order of preference:

| # | Mechanism | Evidence | Confidence |
|---|---|---|---|
| A | **`ApplyEffectComponent`** on our mod actor: `apply_effects(target_object)` / `remove_effects(target_object)` (`bp_api_dump2.txt:1581-1583`). One component per preset effect (e.g. ReloadingTime -10/-20/-30/-40/-50%, defined in our own cfg); the slider picks which one is applied to the player. Clean add/remove. | Used natively by `SimpleVolumeForEffects.effect_component` (`:6518`) and `EMPArea.apply_effects_on_enter/exit` (`:2784-2785`). Its effect-list property is **not** BlueprintVisible (the dump shows only the two functions), so whether it can be set in the Details panel of a BP component is **unknown**: must be opened in the editor. | Medium if editable |
| B | **Quest node** `EQuestNodeType::SetCharacterEffect` started from BP with `CppMediator.start_quest_node(sid)` (`bp_api_dump2.txt:2233`, static BlueprintFunctionLibrary). Vanilla node applies to the player with an empty target: `GD\QuestNodePrototypes\Benchmark_combat.cfg:273-290` `Benchmark_combat_SetCharacterEffect_HealPlayer ... NodeType = EQuestNodeType::SetCharacterEffect ... TargetQuestGuid = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA  EffectPrototypeSID = MedkitHealing`. No "remove effect" node type exists (full node-type census done), so use timed effects (`Duration = N`, like `ReloadingTimeDecBy25_Temp`) re-applied on each reload start or on a ~1 s timer. MCM itself ships a mod quest the same way (`ext\...MCM...OverrideContent\QuestNodePrototypes\MCM_Restart.cfg`, `QuestPrototypes\MCM_Restart.cfg`). | Low-Medium: whether `start_quest_node` works for a node whose quest was never started is untested. |
| C | `ExecuteConsoleCommand("XApplyEffectOnPlayer <SID>")`. The command exists (`GD\QuestNodePrototypes\Arch_Bossfight_Faust.cfg:2209` `ConsoleCommand = XApplyEffectOnPlayer FaustPsyResist`) but it's a cheat-manager command that may be compiled out of Shipping. | Low |
| D | Anim play rate: `AnimationUtilitiesBPFunctionLibrary.get_character_anim_instance(obj)` (`bp_api_dump2.txt:1510`) + engine `AnimInstance.Montage_SetPlayRate` on the active reload montage. The weapon mesh's own AnimBP (`AnimBP_AK74_fp` etc.) plays a separate synced montage and native reload timing may not follow, so desync is likely. | Low, last resort |

Cfg-only fallback (no MCM): optional pak variants, e.g. `-25%` / `-50%`, as a permanent player effect, or `WeaponReloadTimePerAttachment` multipliers generated by script for all 92 weapons. The effect route is one small patch and doesn't touch NPC weapons, but a cfg-only way to attach a permanent effect to the player still has to be found: `Player.ApplicableMechanicsEffects` (`GD\ObjPrototypes.cfg`, Player block) takes *mechanics* effects. A mechanics effect with an always-true condition applying our ReloadingTime effect is plausible but unverified. The per-weapon multiplier route is proven by several Nexus mods.

---

## 3. MCM (Mod Configuration Menu 2.0, Nexus 2225, by KynesPeace/ZunaSW) — installed

Installed at `~mods\AAK-Mod Configuration Menu 2.0 2225 2.0 ...\ModConfigurationMenuStalker2-Windows-{NewContent,OverrideContent}.{pak,ucas,utoc}` (High). Plugin descriptor (extracted `ModConfigurationMenu.uplugin`): `"FriendlyName": "ModConfigurationMenu", "CreatedBy": "KynesPeace", "Category": "Game Features", "ExplicitlyLoaded": true, "Mod": true`.

NewContent packages (mount `/ModConfigurationMenu/`), read with `zen_names.py`:
- `/ModConfigurationMenu/BPI_MCM_API`: functions `AddUniqueModID`, `RegisterModSetting`, `RegisterDefaultModSetting`, `GetModSetting`, `SetModSetting`, `TriggerButtonAction`; parameter names `ModID, SettingID, Category, Type, AuthorName, SettingHoverText, BoolValue, FloatValue, IntValue/IntegerValue, StringValue, VectorValue, RotatorValue, TransformValue, Keybind, ComboBoxOptions, ComboBoxSelectedOption, ButtonText, FloatControllerStepValue, ModHeaderTexture, CategoryHeaderTexture`.
- `/ModConfigurationMenu/BPI_MCM_SettingsProvider`: `RegisterMCMSettings`, `RegisterMCMDefaultSettings`, `ApplyMCMSettings`, `OnMCMButtonPressed` (params `BPI MCM API` interface ref, `ModID`, `SettingID`).
- `/ModConfigurationMenu/Enums/E_MCM_SettingType` (NewEnumerator0..9: bool/float/int/string/vector/rotator/transform/key/combobox/button, matching the widget set `WBP_MCM_Setting{Bool,Float,Int,String,Vector,Rotator,Transform,Key,ComboBox,Button}`).
- Structs `ST_MCM_ModSetting`, `ST_MCM_Save`, `ST_MCM_Categories`; `BP_MCM_Manager` (actor), `MWS_MCM` (ModWorldSubsystem: `OnWorldBeginPlay`, `OnSaveDataRequested`/`OnSaveDataLoaded`, `SetDataForSave`: settings are stored with the save game), `SG_MCM_Settings` (SaveGame: `AllSettings`, `DefaultSettings`).
- Discovery: `BP_MCM_Manager` calls `GetAllActorsWithInterface` and then `RegisterMCMSettings` / `RegisterMCMDefaultSettings` / `ApplyMCMSettings` through the interface (its call nodes include `CallFunc_GetAllActorsWithInterface_OutActors`, `CallFunc_RegisterMCMSettings_BPI_MCM_API_CastInput`, `CallFunc_ApplyMCMSettings_BPI_MCM_API_CastInput`, and `GetModSetting_*`). **So a provider is any spawned actor implementing `BPI_MCM_SettingsProvider`.** High.
- Open key: the manager's names include `F9`; web says the default is B and it can be rebound in the menu. Unimportant.
- OverrideContent holds only `Autogenerated_*_WorldSubsystemData`, `Autogenerated_*_ActorPatchesData`, and loose cfg `QuestNodePrototypes/MCM_Restart.cfg`, `QuestPrototypes/MCM_Restart.cfg` (a "Restart your game" fade-screen quest).
- Author guide: Notion page linked from the Steam Workshop item (https://steamcommunity.com/workshop/filedetails/?id=3725137523) and Nexus (https://www.nexusmods.com/stalker2heartofchornobyl/mods/2225), plus an "MCM Example Mod". Not fetched (403/429); worth downloading the example mod for its Blueprint.

**Soft dependency.** Any Blueprint that implements or calls these interfaces hard-imports `/ModConfigurationMenu/...`; without MCM installed it fails to load (Immersive HUD's actor does exactly this, per `stalker2-immersive-dialogue\zonekit\README.md:365` and `:485`). Plan (same as ImmersiveDialogue's unimplemented 2.1 plan, `README.md:485-493`):
- The main `ImmersiveReloading` plugin keeps every setting as a variable on its own actor (defaults = the no-MCM behaviour) and imports nothing from MCM.
- An optional `ImmersiveReloadingMCM` plugin (separate NewContent + its own ModWorldSubsystem) spawns a provider actor implementing `BPI_MCM_SettingsProvider`. It registers settings (e.g. a float slider "Reload speed" 0.5-1.0 plus bools "Combat sprint while reloading", "Drain stamina"), and in `ApplyMCMSettings` it finds our main actor (`GetActorOfClass`) and writes the values.
- **Unverified:** whether a Zone Kit NewContent Blueprint in plugin B can reference a class in plugin A's NewContent and cook cleanly (the known rule covers NewContent→same-mod and override→mod-only; cross-mod is untested). Fallback that avoids a cross-plugin reference: ship MCM as a *variant* of the whole mod (an `ImmersiveReloading_MCM` plugin that contains the logic plus the provider), and have the user install one or the other.
- Editor prerequisite: to compile the add-on, MCM's interface assets must be in the kit. That means either MCM's source plugin in `<kit>\Stalker2\Mods\` (the Example Mod / guide may provide it) or recreated stub interfaces with identical paths and signatures (risky).

---

## 4. Cfg patch mechanism (High, from installed mods + official Zone Kit doc summary)

Official doc (ZoneKit support "Config patches", https://zonekit-support.stalker2.com/hc/en-us/articles/39357395461265-Config-patches; 403, search snippet): the loader scans `Content\GameLite\GameData`; for each `.cfg` / `.cfg.bin` it looks for `.cfg_patch_*` / `.cfg.bin_patch_*`. `{bpatch}` on a struct adds or replaces child values; `removenode` deletes a node and works only inside `{bpatch}`; patches can reference base nodes without `refurl`.

Placement conventions seen in Noah's `~mods` (all verified by listing/extracting):
1. **Sibling file** `<Name>.cfg_patch_<Mod>` next to the base cfg:
   - `Stalker2/Content/GameLite/GameData/CoreVariables.cfg_patch_PIR` (PIR Main OC), `..._patch_ZoneWatch` (ZST NewContent, path `Content/GameLite/GameData/`), `..._patch_Bloodsplats` (AVCR).
   - Contents (`ext\ProjectItemizationReborn_Main_OC\CoreVariables.cfg_patch_PIR`): `DefaultConfig : struct.begin {bpatch}` / `UIFloatPrecision = 2` / `struct.end`.
2. **Folder named after the cfg** with `<Name>_patch_<Mod>.cfg` (or `<Name>.cfg_patch_<Mod>.cfg`) inside:
   - `GameData/ObjPrototypes/ObjPrototypes.cfg_patch_BS.cfg` (Better Stamina): `Player : struct.begin {bpatch}` / `StaminaPerAction : struct.begin {bpatch}` / `Sprint = 2.6` ... / `struct.end` / `struct.end`.
   - `GameData/ObjPrototypes/ObjPrototypes_patch_PIR_NoRegen.cfg`: `Player : struct.begin {bpatch}` / `VitalParams : struct.begin {bpatch}` / `DegenRadiation = 0.0`.
   - `GameData/WeaponData/WeaponGeneralSetupPrototypes/WeaponGeneralSetupPrototypes_patch_RecoilOverhaul.cfg` (grEdit Ballistics; vanilla has no such folder, the file alone is enough), with nested `{bpatch}` on every struct level, e.g. `GunPM_HG : struct.begin {bpatch}` / `RecoilParams : struct.begin {bpatch}` / `RecoilRadius = 209.1` ...
   - DLC copies are patched separately: `GameLite/DLCGameData/{DLC1,Deluxe,PreOrder,Ultimate}/WeaponData/WeaponGeneralSetupPrototypes/..._patch_RecoilOverhaul.cfg`, so DLC weapon SIDs need their own patch files.
3. **Array elements by index**: `[3] : struct.begin {bpatch}` (grEdit `ImpactPhysicalMaterialPrototypes_patch_ArmorBalance.cfg:22`); remove with `[0] : removenode` (OXA `ItemPrototypes\WeaponPrototypes_OXA.cfg:105-106`, inside `PreinstalledUpgrades : struct.begin {bpatch}`).
4. **New prototypes** in plain files in the same folder, with `{refurl=../EffectPrototypes.cfg; refkey=[0]}` inheritance (OXA `EffectPrototypes/OXAEffects_WeaponHandling.cfg:668`), or in a Zone Kit NewContent plugin under `Content/GameLite/ModGameData/<Mod>/<Type>/...cfg` (Sleeping Bag, PIR NC, ZST; picked up via the GameFeatureData's `AddConfigsPathGameFeatureAction`, visible in MCM's GameFeatureData names).
5. Delivery: pak-only mods ship these as loose files in a `.pak` (mount `../../../Stalker2/Content/GameLite/GameData/`). Zone Kit mods put them in the OverrideContent pak (PIR `_OC`) or in ModGameData (NewContent).

For Immersive Reloading:
- New effects (e.g. `ImmReload_ReloadTime_Minus25`, a timed sprint block) go in `ModGameData/ImmersiveReloading/EffectPrototypes/*.cfg`.
- Per-weapon multiplier presets would be `GameData/WeaponData/WeaponGeneralSetupPrototypes/WeaponGeneralSetupPrototypes_patch_ImmersiveReloading.cfg` (plus the DLC paths), generated by script.
- To patch one field of one array element: `GunAK74_ST : struct.begin {bpatch}` / `WeaponReloadTimePerAttachment : struct.begin {bpatch}` / `[0] : struct.begin {bpatch}` / `TacticalReloadTimeMultiplier = 0.75` / `struct.end` x3.

---

## 5. Conflict candidates in the installed `~mods`

Scanned: every `.pak` listed (`paklists\`), every `.utoc` listed (`utoclists\`), and a byte grep of all `.utoc` files for player anim/input asset names.

| Mod | What it touches | Relevance |
|---|---|---|
| **OXA - Standard 3.0.8** (`AAW`) | `OXA_StandardA_P`: `WeaponData/WeaponGeneralSetupPrototypes/WeaponGeneralSetupPrototypes_OXA.cfg` with `{bpatch}` on `WeaponReloadTimePerAttachment` for many weapons (e.g. `Gun_Lummox_AR_GS`, lines 49-72, all 1.0), plus 27 `..._def_<Weapon>.cfg` new weapons with their own reload arrays; `ReloadTime_Minus*` effects on magazines; **overrides 38 vanilla `AnimCollection_fp_*`** (player reload montage collections). `OXA_StandardC_P` has DLC `WeaponGeneralSetupPrototypes_{DLC1,Deluxe,PreOrder,Ultimate}.cfg`. | Direct overlap with a per-weapon multiplier preset: last loaded wins per field, and OXA's new weapons would be missed. The player-effect route avoids this. Also the reason never to override `AnimCollection_fp_*`. |
| **grEdit Ballistics Redux 5.0.5** (`ABC`) | `WeaponGeneralSetupPrototypes_patch_RecoilOverhaul.cfg` (recoil/dispersion only), `CharacterWeaponSettingsPrototypes_patch_*` | Same file, different fields: no conflict with bpatch. |
| **ZST Immersive HUD variant 1.0.1** (`ZZZZ-ZST`) | `ZoneWatchStalker2-Windows-OverrideContent_30_P`: **`AnimBP_Player`**, `W_Inventory` + inventory widgets, `IA_ZoneWatch*`; NewContent `CoreVariables.cfg_patch_ZoneWatch`. No `IMC_Exploration` in this variant. | Any `AnimBP_Player` override would fight it (already a project rule). |
| **ImmersiveDialogue 2.0.0** (`AAP`) | `zzz_ImmersiveDialogue_20_P`: `BP_Stalker2Character` override | Never override the pawn (rule). Note: the installed copy is 2.0.0, not 2.1.x. |
| **Better Vaulting / Better Stamina / Fall Damage** | `ObjPrototypes/ObjPrototypes.cfg_patch_BS.cfg` (Player `StaminaPerAction.Sprint = 2.6`, Jump...), `..._patch_BV.cfg` | Changes sprint stamina cost; our fake-sprint drain should read or copy the vanilla rate, and only matters if we patch the same fields. |
| **PIR Complete** | `ObjPrototypes_patch_PIR*.cfg` (Player VitalParams), `EffectPrototypes_patch_PIR_SleepMechanics.cfg` (may change sleepiness effects incl. the ReloadingTime ones), `ObjWeightParamsPrototypes_patch_*` | Stacks with our ReloadingTime effect; no direct conflict. |
| **FMAO Aiming/Gamepad** | `EffectPrototypes/~FMAO/EffectPatches_*.cfg`, `ObjPrototypes/~FMAO/zFMAO_aiming.cfg`, `Config/UserInput.ini` | Aiming only. |
| **Weapon Reposition Project** (`AAI`) | `Animations/Player/Curves/Swing/*` and weapon hip curves | Visual only; no conflict. |
| **Weapon Roughness Fix** (`AAO`), **Ballistics SFX**, **AVCR** (CoreVariables patch, projectiles), **BloodFX**, **DWO**, **NVG**, **Sleeping Bag** (own `MG_fp_BedOnGround_SBM` montage in its NewContent), **Sleep Timer**, **Instant Intel**, **PoNR**, **Clear Crouch** (PostProcess cfg), **UltraPlus** (ini) | nothing sprint/reload/weapon-anim/input related | none |
| **MCM 2.0** | its own paths only | dependency for the add-on |

Nobody in the load order overrides `IMC_Exploration`, `IA_Sprint`/`IA_Reload`, `AnimBP_PlayerWeaponLayer`, or `AnimBP_player_bh`.

---

## 6. Open questions: what to ask Noah / open in the Zone Kit

**In game (vanilla behaviour, no new build needed):**
1. Reload, then press sprint mid-reload: does the reload cancel, and does the magazine count revert? Sprint, then press reload: does sprint drop to run with the reload playing, or is the reload refused? Try both hold and toggle sprint. (Defines what the mod must change.)
2. While reloading at a normal run, is movement speed normal run speed, or slowed?

**In the Zone Kit editor (one asset each):**
3. Open `IA_Sprint` and `IA_Reload` (`/Game/_Stalker_2/data/input/InputActions/Delayable/`): list the Triggers and Modifiers, and check whether either has an `InputTriggerActionBlocker` (and its Condition class / `Trigger State`). Can also be done headless with `zonekit/tools/dump_imc.py`-style Python.
4. Add an `ApplyEffectComponent` to a scratch Blueprint actor: does the Details panel show an editable effects list (effect SIDs)? This decides route A for runtime reload speed and for the runtime sprint block.
5. In any Blueprint, place `Is Action Active` (PC) and read the full `ActionType` enum dropdown (does it contain Reload / Unload / Unjam?). Place `Get Current Reload State` and read the `MagazineReloadState` values. These give a clean "reload in progress" test.
6. Open `AnimCollection_fp_AK74` (PlayerFirearmAnimCollection) to see where reload montages and any sprint/interrupt settings live (per-reload-type montage list, play-rate fields).
7. Open one reload montage (e.g. `Player/AnimSequences/ar/ak74/reload/MG_*`) and list its notifies (`AnimNotify_PlayerAction` / `AnimNotify_PlayerActionTrigger` / `AnimNotifyState_ForceReloadingEnd`), to see whether montage notifies (`SPRINT_TRIGGER` DEACTIVATE?) are part of the gate.

**Test builds (each one change):**
8. Runtime effect application: `CppMediator.Start Quest Node` on a mod `SetCharacterEffect` node (route B) vs ApplyEffectComponent (route A). Visible side effect for the canary: an obvious effect such as `ReloadingTimeDecBy25`, reload noticeably faster.
9. `Set Speed Multiplier` vs `velocity_multimplier` while reloading: which one actually changes speed and survives the game's own `update_movement_speed` calls.
10. Cross-plugin reference for the MCM add-on (plugin B's actor referencing plugin A's class) survives the cook.
