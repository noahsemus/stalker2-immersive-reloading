# ImmersiveReloading — engineering log

What each kit check and each in-game test showed, newest at the bottom. Record dead ends here so they are never
retried. Plan: [../PLAN.md](../PLAN.md); reproducible edits: [../BUILD.md](../BUILD.md). Facts that hold for any
mod go to `harness/docs/` instead (promote them).

## 2026-09-30 — repo created from the harness

## 2026-09-30 — research
- Research agent report: `research/2026-09-30-sprint-reload-research.md` (cfg sweep, dumps, installed mods, MCM
  containers, patch syntax). Verdict: the sprint/reload exclusion is native; plan = combat sprint + ReloadingTime
  effects from a NewContent host (PLAN.md).
- Editor Python (open editor, `ue_exec.py`): `IA_Sprint` and `IA_Reload` have no triggers and no modifiers (no
  asset-side blocker). `ActionType` includes `RELOAD_WEAPON`, `UNLOAD_WEAPON`, `UNJAM_WEAPON`, `SPRINT`, `RUN`;
  `MagazineReloadState` = DEFAULT / EJECTED / INSERTED / NONE. `ApplyEffectComponent`'s effect list is not visible to
  Python (guessed names `effects`, `effect_prototype_sids`, ... all fail): needs the tester's Details-panel check.
