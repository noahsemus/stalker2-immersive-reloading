@harness/CLAUDE.md

# {{MOD}} — mod context for the agent

Purpose: {{DESCRIPTION}}

Repo `{{OWNER}}/{{REPO}}`, Zone Kit plugin `{{MOD}}` (`/{{MOD}}/`), asset prefix `{{SHORT}}`. Created from the
harness on {{DATE}}. The shared rules, tester workflow, pipeline and game knowledge are in `harness/` (imported
above); this file holds only what is specific to this mod.

## Current state (one line per checkpoint / release: date, what changed, what the tester confirmed in game)
- {{DATE}}: repo created. Next: kit research for PLAN.md, then the tester picks `{{MOD}}` in the editor's toolbar
  mod selector once (GameFeatureData).

## Key facts (verified; cite file:line in the kit)
- _(fill in as research lands)_

## Assets this mod may override
- _(none yet; see harness/docs/compatibility.md before adding any)_

## Files
- `PLAN.md` plan and test matrix · `BUILD.md` every edit asset by asset · `zonekit/README.md` engineering log
  (what each test showed, dead ends) · `zonekit/{{MOD}}/` plugin mirror · `zonekit/tools/` mod-specific generators
  and classifier lists · `zonekit/builds/` checkpoint and release paks · `mod.json` names and pak suffixes.

## Release (only when the tester says "cut a release")
Follow `harness/docs/pipeline.md` § Release. Nexus page: _(record the mod id here once it exists)_.
