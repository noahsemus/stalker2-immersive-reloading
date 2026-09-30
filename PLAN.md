# ImmersiveReloading — plan

Sprint and reload at the same time, with an optional reload-speed setting.

Full research with verbatim cfg/dump quotes: [zonekit/research/2026-09-30-sprint-reload-research.md](zonekit/research/2026-09-30-sprint-reload-research.md).

## How the game does it today

**Sprint vs reload is decided in native C++, not in data.**
- Player reports (to be confirmed by the tester, Q1): pressing sprint during a reload cancels it (it restarts from
  the beginning next time); pressing reload while sprinting drops Skif to a run and the reload plays.
- No cfg field mentions it (keyword sweep of all GameData: 0 hits). `IA_Sprint` and `IA_Reload`
  (`/Game/_Stalker_2/data/input/InputActions/Delayable/`) have **no triggers or modifiers** (read in the editor
  2026-09-30), so there is no input-asset blocker either. The rule lives in the player's native action system
  (`ActionType::RELOAD_WEAPON` / `SPRINT`, `PlayerTriggerState::RELOAD_TRIGGER` / `SPRINT_TRIGGER`, montage notifies
  `AnimNotify_PlayerAction` END/INTERRUPT).
- A true "reload animation in the sprint pose" would need native changes. Not possible pak-only.

**What the game does give us** (BP-callable, `bp_api_dump2.txt`):
- Reload in progress: `PC.Is Action Active(RELOAD_WEAPON)` (enum read 2026-09-30), `Obj.Get Current Reload State`
  (DEFAULT / EJECTED / INSERTED / NONE).
- Sprint: `Obj.Can Enter To Sprint`, `Is Sprinting`, `On Sprint Pressed`, `PC.On Sprint Released`.
- Speed: `Obj.Set Speed Multiplier` (:5081), `PC.velocity_multimplier` (Read-Write, :5380), `Update Movement Speed`.
  Player speeds (`ObjPrototypes.cfg` Player `MovementParams`): run 370, jog 625, sprint 820.
- Stamina: `Obj.Force Set SP` / `Get SP` / `Get Max SP`; sprint costs `StaminaPerAction.Sprint = 5.25`
  (Better Stamina patches it to 2.6).
- Blocking sprint by data: effect type `BlockAnimationActionType` with `EActionType::Sprint` (vanilla
  `BlockSprintEffect`, used on heavy exoskeletons; `ConcussionBlockSprint` is a 3 s timed one).
- Reload speed by data: effect type `ReloadingTime`, negative % = faster (vanilla `ReloadingTimeDecBy25` = -25%;
  sleepiness already applies +15% / +18.75% to the player). Per weapon: `WeaponReloadTimePerAttachment`
  multipliers, which OXA (installed) also patches for its own weapons and which probably hit NPCs too.
- No function "apply effect to player" exists. Two candidates, neither tested:
  **(A)** an `ApplyEffectComponent` on our actor (`Apply Effects` / `Remove Effects`); its effect list is invisible to
  Python, so the tester must check whether the Details panel lets us fill it;
  **(B)** `CppMediator.Start Quest Node` on our own `SetCharacterEffect` quest node (applies to the player with an
  empty target; no "remove" node, so timed effects re-applied while needed).

## Approach

Everything runs from a **ModWorldSubsystem + actor in NewContent** (`harness/docs/runtime-host.md`), plus new cfg
prototypes in `ModGameData/ImmersiveReloading/`. **No game asset is overridden** (OXA overrides 38 weapon anim
collections, ZST `AnimBP_Player`, ImmersiveDialogue the pawn; we touch none of them).

### Feature 1: "combat sprint" while reloading
The reload plays in the normal running pose while Skif moves at sprint speed and spends stamina like a sprint.
1. **Keep the reload alive**: while `Is Action Active(RELOAD_WEAPON)`, apply a short timed sprint-block effect
   (our copy of `ConcussionBlockSprint`, ~0.5 s, refreshed while reloading), so a sprint press can't start the
   native sprint that cancels the reload.
2. **Sprint intent**: the actor listens to `IA_Sprint` (Started / Completed) and follows the game's sprint mode
   (`SettingsVariablesPC.cfg` `SprintInputType = ToggleOrHold`: hold = sprint while held, tap = toggle).
3. **Speed and stamina**: while reloading with sprint intent, forward input and stamina above the game's sprint
   threshold (16.1): speed multiplier 820/370 ≈ 2.2, stamina drained at the sprint rate. Restore the multiplier the
   moment the reload ends or intent stops.
4. **Hand-back**: when the reload ends with sprint still held, the native sprint resumes (call `On Sprint Pressed`
   if it doesn't on its own).

### Feature 2: reload speed
- Our own `ReloadingTime` effects (e.g. -10 % … -50 % in 10 % steps, permanent) in
  `ModGameData/ImmersiveReloading/EffectPrototypes/`. Player-only, all weapons including other mods' weapons, no clash
  with OXA's per-weapon arrays, stacks with sleepiness.
- The main mod applies the one matching its `ReloadSpeed` variable (default: vanilla, i.e. none) through route A or B.
- Without MCM: optional pak variants with a different default (e.g. "Faster Reload -25 %", "-50 %"), chosen in
  Vortex's file chooser.
- With MCM: optional `ImmersiveReloadingMCM` plugin, a `BPI_MCM_SettingsProvider` actor that registers a
  "Reload speed" slider and "Sprint while reloading" / "Reloading costs stamina" toggles and writes them into our
  actor. The main mod never imports MCM (`harness/docs/compatibility.md` § MCM).

## Decisions for the tester (before build 1)
1. Is the combat sprint (run pose at sprint speed while reloading) what you want, given a true sprint-pose reload
   isn't possible?
2. Should sprinting while reloading cost stamina like a normal sprint? (Proposal: yes.)
3. Default reload speed: vanilla, with the setting only in MCM / optional variants? (Proposal: yes.)

## Open questions and how each gets answered
| # | Question | Answered by |
|---|---|---|
| Q1 | Vanilla: reload → press sprint = reload cancelled, ammo reverted? sprint → press reload = drops to run? (hold and toggle sprint) Slower while reloading at a run? | tester, in game, no build |
| Q2 | Can an `ApplyEffectComponent` on a Blueprint actor be given effects in the Details panel? | tester opens a scratch Blueprint, adds the component, screenshots Details |
| Q3 | Does route A (or B) actually apply an effect to the player at runtime? | build 1 |
| Q4 | Which of `Set Speed Multiplier` / `velocity_multimplier` changes speed and survives the game's own `Update Movement Speed`? | build 3 |
| Q5 | Does a blocked sprint press still reach our `IA_Sprint` event? | build 2 |
| Q6 | Can plugin B's Blueprint reference plugin A's actor class and survive the cook? | build 5 (fallback: MCM variant of the whole mod) |
| Q7 | MCM's interface assets in the kit (example mod / source plugin) | download "MCM Example Mod" from MCM's Nexus page |

## Builds (one change each, tested before the next)
| Build | Change | Success looks like |
|---|---|---|
| 0 | tester picks `ImmersiveReloading` in the toolbar mod selector (GameFeatureData); Q1, Q2 | |
| 1 | subsystem + actor; at load it applies vanilla `ReloadingTimeDecBy25` to the player (route A, else B) | every reload visibly faster (proves the host runs and the effect route works) |
| 2 | sprint block while reloading | reload, then press sprint: the reload finishes, Skif runs |
| 3 | speed multiplier + stamina drain while reloading with sprint intent | reload while sprinting: sprint speed, stamina bar drains, reload finishes |
| 4 | own ReloadingTime effects + `ReloadSpeed` variable (default vanilla); build-1 test effect removed | vanilla speed by default; changing the variable changes the speed |
| 5 | `ImmersiveReloadingMCM` add-on | MCM shows the settings; changes apply without reloading the save |
| 6 | optional preset variants | each variant's speed is right |

## Test matrix (all must pass before a release)
| # | Setup | Steps | Expected |
|---|---|---|---|
| 1 | keyboard, hold sprint | sprint, press reload | sprint speed during the whole reload, reload completes, sprint continues after |
| 2 | keyboard, hold sprint | reload, then hold sprint mid-reload | reload not cancelled, speeds up to sprint |
| 3 | toggle sprint (tap) | both orders | same as 1-2 |
| 4 | gamepad (stick click) | both orders | same |
| 5 | low stamina / overweight | sprint + reload | no speed boost when the game wouldn't let you sprint |
| 6 | weapon types | pistol, rifle, shotgun (shell by shell), twin mag, unload, unjam | each completes; no stuck speed afterwards |
| 7 | interruptions | fire, aim, swap weapon, jump, crouch, open backpack mid-reload | same as vanilla; speed always restored |
| 8 | reload speed | each setting / variant | reload time matches; NPC reloads unchanged |
| 9 | with our other mods + OXA + MCM installed / MCM not installed | play 10 min | no errors; main mod works without MCM |
| 10 | save / load mid-reload, death, cutscene, dialogue | | no stuck speed or blocked sprint |
