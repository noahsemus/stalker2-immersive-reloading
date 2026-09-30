# Verified game behaviour

Facts established in game (probe or tester) or from the kit, with the mod that found them. Add to this whenever a
mod learns something about the game itself. Line numbers refer to `<kit>\bp_api_dump2.txt` unless noted.

## Player (`PC`, `/Script/Stalker2.PC`; runtime class `BP_Stalker2Character_C`)
- Class members from line ~5141. Useful BP surface: `can_use_inventory`, `contextual_action` / `bInContextualAction`
  (BP-writable, but writing it changes nothing by itself), `is_interaction_in_progress`, `is_using_pda`,
  `is_inspecting_artifact`, `start_use_pda(initial_page_type)`, `start_use_backpack()`, `consume_planned_item`,
  `SetMoveVector` / `MovementInputVector` (the game's own move feed), `IsInStaticDialog`, `IsInCinematic` (on `Obj`),
  `ResetInteractionTarget` / `GetInteractionTarget` / `SetInteractionTarget`, `EnableInputAfterInteraction`,
  `DisableInteractions` / `EnableInteractions`, `ToggleFOVAndForegroundRender`, `ChangeMainHandWeapon`,
  `EquipLastHeldItem`, `HasItemInMainHand`, `IsLeftHandBusy`, `IsUsingBackpack`.
- The native move handler drops `IA_LocomotionForward` while `IsInStaticDialog()`, but not look (Dialogue).
- `IsInStaticDialog()` is also true in story cutscenes that run through the dialogue system; guard with
  `NOT (IsInCinematic OR controller.IsLookInputIgnored)` (Dialogue 2.0.3).
- `IsInStaticDialog` reads a non-reflected pointer at `[PC+0x650]` that is null during the sleep transition (a native
  hook calling it then crashes; Blueprint callers are fine).
- The dialogue ends by distance (`DialogDistance = 5.0` in CoreVariables) possibly while a key is held, so an input
  action's `Completed` can be lost; zero anything the mod fed after a short stale window (Dialogue 2.1.1).
- `CoreVariables.cfg` `MaxInteractionDistance = 200` applies to every interaction; NPC dialogue distance is
  `MinDialogInteractDistance` / `MaxDialogInteractDistance` on the object prototypes (base `[0]`: 75 / 130; ~75 named
  NPCs override it).

## Interactions and contextual actions (Campfires)
- A campfire sit is a `PlayerContextualAction` actor (`BP_PlayerContextualAction*` next to `BP_Stalker2Bonfire*`):
  sets the contextual-action flag, clamps camera yaw/pitch to the actor's limits, pushes `IMC_PlayerCA` at priority
  Exclusive (`InputMappingContextPrototypes.cfg:294-298`).
- The game treats the sit as an **interaction in progress** for its whole length; native PDA / backpack open paths
  refuse while it is (keys and even direct `StartUsePDA` / `StartUseBackpack` calls do nothing). Clearing the
  interaction target ends the sit itself. Hence "own seated mode": take over after the vanilla sit-in, then
  `ResetInteractionTarget`, clear the flag, `EnableInputAfterInteraction`, `ToggleFOVAndForegroundRender(true)`,
  `DisableInteractions` (hides the seat prompt), and play the pose ourselves.
- The vanilla sit's `SaveStatesBeforeInteraction` is only undone by its own exit (weapon half-state otherwise).
- Binding `IA_PlayerCAExit` in a mod actor crashed (the game's delayable handler treats the bound object as the PC;
  AV in the `PC.IsVaulting` thunk).
- `CppMediator.lerp_player_to_location_and_rotation`, `CppMediator.start_quest_node(sid)` exist (seat placement,
  time-skip leads).

## Input
- Gamepad via GameInput.dll, not XInput. DualSense without Steam Input is not an XInput device.
- Quick slots have no BP-callable entry on `PC` (only `consume_planned_item`, which needs a natively planned item).

## Camera
- `CameraModifier_LookAt` centres the camera on the NPC in dialogue; `FindCameraModifierByClass` →
  `DisableModifier(true)` → `RemoveCameraModifier` per tick removes it.
- Actor yaw drags `ControlRotation`; `bOrientRotationToMovement` rotates the actor on strafe in dialogue. Camera
  decouple = camera `Set Absolute` (rotation) + `Set World Rotation(Get Control Rotation)`, restoring the saved relative
  rotation after.
- Camera-manager view pitch/yaw limits can be set per tick for a seated look clamp.

## Items and UI
- Artifact inspection has no input action of its own; it launches from the backpack UI.
- The dialogue skip hint `W_SkipHintView` has native show logic (`SkipHintView` base); the BP is layout + fade only.
- Trading opens inside the dialogue (`IsInStaticDialog` stays true) and takes UI-only input.
- No BP-exposed API opens the PDA / inventory views directly (nothing on UIManagerEx / ViewBase / CppMediator).
