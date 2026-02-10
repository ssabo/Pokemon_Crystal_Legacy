# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pokemon Crystal Legacy is a Game Boy Color ROM hack based on the [PRET pokecrystal disassembly](https://github.com/pret/pokecrystal). The entire codebase is RGBDS assembly (Z80-like, Game Boy CPU) with custom C/Python build tools. This is **not** a typical software project — it's a reverse-engineered Game Boy ROM with ~163K lines of assembly.

## Build Commands

**Prerequisites:** RGBDS 0.5.2, make, gcc, git

```bash
make                    # Build pokecrystal.gbc (default target)
make crystal11          # Build pokecrystal11.gbc (version 1.1)
make crystal_au         # Build Australian release
make crystal_debug      # Build with debug symbols
make crystal11_debug    # Build v1.1 with debug symbols
make clean              # Remove all build artifacts including compiled graphics
make tidy               # Remove only ROM files, object files, and tool binaries
make compare            # Build ROMs and verify SHA1 checksums against roms.sha1
make tools              # Build only the custom C tools in tools/
```

Use `make RGBDS=path/to/rgbds/` to use a local RGBDS installation instead of a global one.

Build variants are controlled by assembler defines: `_CRYSTAL11`, `_CRYSTAL_AU`, `_DEBUG`, `_CRYSTAL11_VC`.

## Verification

There is no test suite. The primary verification method is `make compare`, which builds all ROM variants and checks their SHA1 checksums against `roms.sha1`. The RGBDS assembler is run with `-L -Weverything -Wnumeric-string=2 -Wtruncation=1` so compiler warnings serve as a form of linting.

## Architecture

### Memory Model

The Game Boy has a banked memory architecture. ROM0 (the "home bank") is always accessible and contains core routines. ROMX banks (128 x 16KB) are swapped in/out as needed. The bank layout is defined in `layout.link`.

- **home.asm / home/**: ROM0 — always-resident core functions (interrupts, VBlank, joypad, text rendering, core battle/pokemon/menu routines, memory operations, far-call dispatch)
- **main.asm**: Master include file that pulls in all banked engine code and data via SECTION declarations
- **engine/**: Banked game logic organized by system (battle/, overworld/, pokemon/, items/, menus/, events/, etc.)
- **data/**: Static game data (pokemon stats, moves, items, maps, text strings, trainer parties, shop inventories)
- **maps/**: Map event scripts — one .asm file per map containing NPC interactions, warps, triggers, and signposts
- **constants/**: Named constants organized by domain (~47 files). `constants.asm` is the master include.
- **macros/**: Assembly macros. `macros.asm` is the master include. `macros/scripts/` has event scripting macros.

### Key Systems

- **Far calls**: Since most code lives in banked ROM, cross-bank calls use `farcall` / `callfar` mechanisms defined in `home/farcall.asm`
- **Event scripting**: Maps use a scripting system with commands documented in `docs/event_commands.md`. Scripts are written using macros from `macros/scripts/`
- **Battle system**: Split across `engine/battle/` with 4 turn stages (core.asm, effect_commands.asm, move_effects/) and battle animations in `engine/battle_anims/`
- **Predef system**: A dispatch table (`engine/predef.asm`) for calling numbered functions across banks, invoked via `predef` macro

### Graphics Pipeline

Graphics are PNG files in `gfx/` that get converted during build:
1. `rgbgfx` converts PNG → 1bpp/2bpp (Game Boy tile format)
2. `tools/gfx` applies post-processing (dedup, trim, interleave, etc.)
3. `tools/lzcomp` applies LZ compression where needed
4. Pokemon front sprites additionally generate animation tilemaps, frame data, and bitmasks via `tools/pokemon_animation_graphics` and `tools/pokemon_animation`

### Audio

`audio/engine.asm` is the sound engine. Music tracks are in `audio/music/` as assembly. Pokemon cries are in `audio/cries.asm`.

## Code Style (from STYLE.md)

- **Indentation**: Tabs for indentation, spaces for alignment
- **Labels**: PascalCase for ROM labels, prefixed for memory locations (`w` = WRAM, `s` = SRAM, `v` = VRAM, `h` = HRAM). Local labels use `.snake_case` for jumps, `.PascalCase` for code/data blocks.
- **Constants**: UPPER_CASE for most constants, `r` prefix for hardware registers
- **Directives**: UPPERCASE for meta directives (SECTION, INCLUDE, MACRO), lowercase for data (db, dw) and code macros (farcall, predef)
- **Comments**: Above the code they describe, not inline. Semicolon followed by a space. 80-char soft limit.
- **Commented-out code**: Semicolon before the tab indent (`;	nop` not `	; nop`)

## Documentation

- `docs/event_commands.md` — Event scripting command reference
- `docs/text_commands.md` — Text rendering commands
- `docs/map_event_scripts.md` — How map event scripts work
- `docs/music_commands.md` — Audio/music system commands
- `docs/battle_anim_commands.md` — Battle animation commands
- `docs/bugs_and_glitches.md` — Known bugs from the original game
- `docs/design_flaws.md` — Documented design flaws
- PRET wiki tutorials: https://github.com/pret/pokecrystal/wiki
