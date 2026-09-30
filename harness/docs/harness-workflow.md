# Harness lifecycle: new mods, syncing, promoting

One central harness (github.com/noahsemus/stalker2-modding-harness), one copy of `harness/` inside every mod repo,
never edited independently. Mod repos are created from the harness and keep it as a git remote named `harness`.

## Ownership
| Path | Owner | Changes by |
|---|---|---|
| `harness/**` | upstream harness | `push_harness.ps1` (promote) and `sync_harness.ps1` (pull) only |
| `harness/local.json` | this machine | never committed (gitignored) |
| `CLAUDE.md`, `mod.json`, `PLAN.md`, `BUILD.md`, `README.md`, `zonekit/**`, `.gitignore` | the mod | normal commits |
| `.harness-sync` | tools | the upstream commit `harness/` was last synced to |

The mod's `CLAUDE.md` starts with `@harness/CLAUDE.md`, so every session in every mod loads the shared rules.

## New mod
From the harness repo (or any mod repo):
```
powershell -File harness\tools\new_mod.ps1 -Name ImmersiveFoo -Short ImmFoo -RepoName stalker2-immersive-foo -Description "..."
```
Clones the harness into a sibling folder, renames the remote to `harness`, writes the templates
(`harness/templates/`), `mod.json`, empty classifier lists and `.harness-sync`, runs `CreatePlainMod.bat` and fixes
the `.uplugin`, mirrors the plugin into the repo, commits, and creates + pushes the public GitHub repo (`-Private`,
`-NoGitHub`, `-NoKit` to change that). Then the tester restarts the editor and picks the mod in the toolbar selector
once (creates the GameFeatureData); mirror again and commit.

Forkers outside this account: fork on GitHub, clone, set `harness_remote_url` / `github_owner` in
`harness/local.json` (or edit `config.json` in your fork), then use `new_mod.ps1` from your fork.

## Pull harness updates into a mod
`powershell -File harness\tools\sync_harness.ps1` at the start of a session (and whenever another mod promoted
something). It replaces `harness/` with upstream `main` (deleting files removed upstream), updates `.harness-sync`,
commits "Sync harness to <sha>". It refuses if `harness/` has changes that were never promoted.

## Promote a learning from a mod
1. Sync first.
2. Edit `harness/docs/*.md` (or a tool) in the mod repo. Write for any mod: game/kit/pipeline facts, tester workflow,
   tools. Mod-only history stays in the mod's log.
3. `powershell -File harness\tools\push_harness.ps1 -Message "docs: <what>"`: applies the diff to the local harness
   clone (`..\stalker2-modding-harness`, cloned if missing), commits, pushes, then syncs this repo.
Do it in the same session. A mod with unpromoted harness edits is exactly the fracture this setup prevents.

## Working in the harness repo itself
It has no `mod.json`; tools that need a mod take `-Mod`. Commit and push normally; then run `sync_harness.ps1` in
each active mod repo (or let the next session do it). Keep `harness/templates/` in step with how the mods are really
laid out.

## Adopting an existing mod repo
Add the remote, check out the harness, record the sync point, move the repo's generic tools/docs out (they now live
in `harness/`), and make `CLAUDE.md` import `@harness/CLAUDE.md`:
```
git remote add harness https://github.com/noahsemus/stalker2-modding-harness.git
git fetch harness main
git checkout harness/main -- harness
git rev-parse harness/main > .harness-sync
```
Then add `mod.json` (see `templates/`), delete the repo's copies of tools now in `harness/tools/`, and keep only
mod-specific generators in `zonekit/tools/`.
