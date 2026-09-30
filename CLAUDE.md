@harness/CLAUDE.md

# Working in the harness repo itself

This repo is the upstream of `harness/` for every Stalker 2 mod repo (see `harness/docs/harness-workflow.md`). There is
no mod here and no `mod.json`; tools that need a mod take `-Mod`.

- Commit and push harness changes directly here, then run `harness\tools\sync_harness.ps1` in each active mod repo
  (sibling folders `..\stalker2-immersive-*`).
- Keep everything generic: no mod-specific history (that goes in the mod's `zonekit/README.md`), no machine-specific
  paths outside `harness/config.json` defaults.
- The repo is public: nothing private (credentials, emails, personal files) in docs, tools or commit messages.
- Root `README.md` / `CLAUDE.md` / `.gitignore` belong to this repo only; mods get theirs from `harness/templates/`.
