# Stalker 2 modding harness

A shared "agentic harness" for building **S.T.A.L.K.E.R. 2: Heart of Chornobyl** mods with the official Zone Kit
and an AI coding agent (written for Claude Code): the agent instructions, everything learned about the game and
the kit while shipping real mods, and the tools that cook, install and release them. Every mod repo is created
from this one and keeps an identical copy of `harness/`, so knowledge never splits between repos.

Mods built with it: [Immersive Dialogue](https://github.com/noahsemus/stalker2-immersive-dialogue),
[Immersive Campfires](https://github.com/noahsemus/stalker2-immersive-campfires),
[Immersive Reloading](https://github.com/noahsemus/stalker2-immersive-reloading).

## What's inside
- [harness/CLAUDE.md](harness/CLAUDE.md): the agent's standing instructions, imported by every mod's `CLAUDE.md`.
- [harness/docs/](harness/docs/): how to work with a human tester, the Zone Kit as a research tool, the
  cook/install/release pipeline and its hidden rules, Blueprint paste-as-text, input and rebinds, running logic
  without overriding game assets, animation facts and crash families, cfg data, UE4SS probes, compatibility with
  popular mods, verified game behaviour.
- [harness/tools/](harness/README.md): PowerShell and kit-Python tools (cook + install, editor remote Python,
  headless editor scripts, Blueprint T3D generation, cooked-package readers, installed-mod conflict scan, release
  zips, new mod / sync / promote).
- [harness/templates/](harness/templates/): the starting files of a new mod repo.

## The model
- A human **tester** plays the game and does the clicks in the Zone Kit editor; the **agent** does the research,
  scripting, Blueprint paste text, cooking, installing, log reading, docs and releases.
- Pak-only mods. Prefer new content over overriding game assets; cfg changes as patch files only.

## Use it
Requirements: Windows, the S.T.A.L.K.E.R. 2 Zone Kit (Epic Games Store, ~600 GB), the game, Git, the GitHub CLI
(`gh`), Claude Code (or another agent that reads `CLAUDE.md`).

1. Fork this repo (or use it as is) and clone it next to where your mod repos will live.
2. If your paths differ, create `harness/local.json` with any of the keys in
   [harness/config.json](harness/config.json) (kit, game, GitHub owner, harness remote).
3. Create a mod:
   ```
   powershell -File harness\tools\new_mod.ps1 -Name MyMod -Short MyM -RepoName stalker2-my-mod -Description "What it does."
   ```
   This clones the harness into `..\stalker2-my-mod`, fills in the templates, creates the Zone Kit plugin and a
   public GitHub repo. Open the new folder in Claude Code and start with its `PLAN.md`.
4. Keep mods current with `harness\tools\sync_harness.ps1`; send what a mod teaches you back with
   `harness\tools\push_harness.ps1 -Message "..."`. Details: [harness/docs/harness-workflow.md](harness/docs/harness-workflow.md).

## License
MIT. Game names and assets belong to GSC Game World; this repo contains no game assets.
