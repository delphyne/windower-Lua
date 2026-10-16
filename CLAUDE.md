# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Official Lua addons and scripts for **Windower v4** (the Final Fantasy XI client mod). Code runs inside the Windower host process on Windows, embedded **Lua 5.1** (`.luarc.json` sets `runtime.version` accordingly — no `goto`, integer division, `utf8`, etc.). Developer docs: https://dev.windower.net/

There is no build, lint, or test tooling in this repo. Code can only be executed in-game:
- Addons: `//lua load <name>`, `//lua reload <name>`, `//lua unload <name>`
- Scripts in `scripts/`: `//lua exec <name>` (run once, then unloaded)

For static checking, the Lua Language Server picks up `.luarc.json`, which adds `definitions/` (EmmyLua `---@meta` stubs for the `windower.*` API and lib globals) to the workspace library. Keep those stubs in mind as the reference for the host API, since the runtime is not available locally.

## Layout

- `addons/<Name>/<Name>.lua` — each addon is a directory whose entry file matches the directory name. Windower sets `windower.addon_path` to that directory; addon-local modules are loaded with `require` relative to it.
- `addons/libs/` — shared libraries any addon can `require` (e.g. `tables`, `lists`, `sets`, `strings`, `functions`, `logger`, `config`, `texts`, `images`, `packets`, `resources`, `extdata`, `files`).
- `addons/addons.xml` — registry of addons (name, author, description, bugtracker, support). New addons must be added here.
- `scripts/` — one-shot Lua scripts.
- `definitions/` — LuaLS type stubs only; not loaded at runtime.
- `addons/enemybar2/`, `addons/Balloon/`, `addons/pop/` — git submodules (personal forks in this checkout), not part of upstream.

## Addon conventions

- Top of the entry file declares metadata: `_addon.name`, `_addon.author`, `_addon.version`, and `_addon.command` or `_addon.commands` (chat command aliases).
- Behavior is wired through `windower.register_event('load' | 'unload' | 'addon command' | 'incoming chunk' | 'outgoing chunk' | 'prerender' | ...)`.
- Settings: `config = require('config')`, define a `defaults` table, then `settings = config.load(defaults)`; this persists to `data/settings.xml` inside the addon directory. On-screen text uses `texts.new(...)` with a settings sub-table.
- Game data (spells, items, buffs, zones, ...) comes from `res = require('resources')`, which reads files from Windower's `res/` directory at runtime — those files are not in this repo.
- Packet work uses `packets = require('packets')` with field definitions in `addons/libs/packets/fields.lua`.

## Shared library mechanics

The libs extend the standard library and Lua's primitive types rather than returning isolated modules:
- Each lib registers itself in the global `_libs` table (e.g. `_libs.tables = table`) and cross-references other libs through it; check `_libs.<x>` before assuming an optional dependency is loaded.
- `tables`, `lists`, `sets`, `strings`, `functions` add methods and constructors used pervasively in addons: `T{}` (table), `L{}` (list), `S{}` (set), plus chained methods like `:map`, `:filter`, `:reduce`, `:keyset`, `:it()`, and string methods such as `:split`, `:color`.
- `logger` replaces global functions like `error`/`print` with chat-log versions (originals preserved in `_raw`; GearSwap restores `error = _raw.error` for this reason).
- Changes to `addons/libs/` affect every addon — treat them as a public API.

## GearSwap

`addons/GearSwap` is the largest and most active addon. `gearswap.lua` is the entry point; it wraps `windower.text`/`windower.prim` creation, then delegates to modules for packet parsing (`packet_parsing.lua`), action flow and precast/midcast/aftercast sequencing (`flow.lua`), target resolution (`targets.lua`), equipment validation/processing, and the API exposed to user job files (`user_functions.lua`, `helper_functions.lua`). Users' job files live in `GearSwap/data/` and are sandboxed against that exposed API.

## Commits

Commit subjects are prefixed with the addon name in brackets, e.g. `[GearSwap] Consolidate r_line assignment`, `[Battlemod] Fix for message id 38`. The main branch is `dev`.
