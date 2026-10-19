# Stellaris modding — AGENTS.md

Orientation notes for any Stellaris modding task (new mods, updates, compatibility patches, UI tweaks).
Scope: Windows + Steam install. **The game folder and other people's mods are read-only** — only write inside your own mod root.

## Environment paths
- Game install (read-only): `C:\Program Files (x86)\Steam\steamapps\common\Stellaris`
  - Vanilla UI: `<game>\interface\` (`.gui` layout, `.gfx` sprites), textures under `<game>\gfx\`
- Workshop mods (read-only, one folder per workshop id): `C:\Program Files (x86)\Steam\steamapps\workshop\content\281990\<remote_file_id>\`
- User data: `C:\Users\<user>\Documents\Paradox Interactive\Stellaris\`
  - Local mods: `...\mod\<name>\` (the mod root is usually the working directory)
  - Logs: `...\logs\error.log`, `...\logs\game.log` — first thing to read after any change
  - Generated script docs (all effects/scopes/triggers/modifiers/localisations): `...\logs\script_documentation\{effects,scopes,triggers,modifiers,localizations}.log` — grep these instead of guessing keyword names
  - Saves: `...\save games\`
- Wiki: https://stellaris.paradoxwikis.com/Modding (interface notes: `.../Interface_modding`)

## Mod folder layout
```
<mod root>/
  descriptor.mod          # metadata: name, version, tags, supported_version, dependencies, remote_file_id
  interface/              # .gui layout and .gfx sprite definitions
  gfx/                    # textures (.dds) — must mirror the vanilla/other-mod relative path to override
  common/                 # gameplay scripts (techs, buildings, defines, scripted triggers/effects)
  events/                 # event definitions
  localisation/<lang>/    # .yml localisation (needs UTF-8 with BOM)
  unchecked_defines/      # define overrides, applied only if the user has unlocked defines
```
- The launcher reads `descriptor.mod`; the sibling `<name>.mod` file (outside the folder, in `mod\`) holds `path = "mod/<name>"`. Do not put `path` in `descriptor.mod`.
- Do not add stray folders/files to the mod root — everything there is loaded by the game.
- `dependencies = { "Mod A" "Mod B" }` in `descriptor.mod` asks the launcher to load those first.

## Load order & override rules (most important)
- Every enabled mod is loaded; conflicts are resolved by **launcher load order — later wins**. Patch to override, don't edit the other mod.
- **Same relative path:** the later mod's **entire file** replaces the earlier one. Its windows/elements are matched by `name`; anything the earlier file defined and yours does not is gone.
- **Different relative path:** both files load, and elements merge by name (later wins). Windows you don't redefine keep coming from the other file.
- Community convention: keep overrides in a subfolder whose name starts with Greek **Alpha** (`Α`, U+0391) so it sorts before Latin-named folders; keep the exact folder name consistent when patching a specific mod.
- To patch another mod, prefer overriding **individual windows** from a differently-named subfolder instead of replacing their whole file, so unrelated future updates still flow through.

## Interface files (.gui / .gfx)
- Element types: `containerWindowType` (window/group; `name` must be unique), `instantTextBoxType` (`text`, `font`, `maxWidth`, `format`), `iconType` (`spriteType`, `frame`, `scale`), `buttonType` (`quadTextureSprite`/`spriteType`), `smoothListBoxType` (dynamic list filled by code), `positionType`, `gridBoxType`.
- Arithmetic on variables works in sizes/positions: `@my_var = 200` … `position = { x = @[my_var + 14] y = 5 }`. Negative `x` = offset from the right edge (`orientation = upper_right`).
- Useful keys: `orientation = upper_left|upper_right`, `format = left|center|right`, `alwaysTransparent = yes` (click-through), `fixedSize = yes` (no auto-grow), `vertical_alignment = center`.
- Resizing a window usually requires moving its children too (e.g. its list `position = { x = 0 y = N }`), or you get gaps/overlaps.
- Commented-out (`#`) blocks still show up in text searches — scope replacements to one window/section.

### Sprites & atlases (.gfx)
- `spriteType = { name = "GFX_foo" texturefile = "gfx/.../foo.dds" noOfFrames = N }`.
- Frame width = `textureWidth / noOfFrames` (engine assumes one horizontal strip). A mod that overrides the `.dds` without matching `noOfFrames` (very common in outdated mods) misaligns every frame → half/garbled/wrong icons.
- Read `.dds` header directly: width = UInt32 at byte 16, height = UInt32 at byte 12.
- Many `GFX_` sprites are driven by code (frame chosen at runtime); a `frame =` in the .gui is only a fallback.

## Other content types (quick map)
- `common/`: gameplay definitions (defines, technologies, buildings, scripted triggers/effects…). Files in `common/defines/` only take effect with defines unlocked → ship them in `unchecked_defines/`.
- `events/`: `event = { id = namespace.N ... }`; every localisation key referenced must exist.
- `localisation/<lang>/*.yml`: `l_english:` header, then ` key:0 "value"` lines. Requires UTF-8 **with BOM**; missing keys render as raw key names in game.
- `gfx/`, `sound/`, `fonts/`: assets, same relative-path override rule as everything else.

## Colours & inline text markup
- Colour codes are written as a section sign `§` followed by a single letter, e.g. `§G` green, `§R` red. `§!` returns to the colour in use before the last change. Docs: https://stellaris.paradoxwikis.com/Localisation_modding#Color_Codes
- Common codes: `W` white, `T` light grey (default body text), `g` dark grey (disabled/inactive), `L` brown/kaki (lore), `P` light dirty pink (aggressive text), `R` red (negative modifiers), `S` dark orange (subtle highlight), `H`/`K` mango/orange (highlight), `Y`/`I` yellow (sub-optimal/neutral), `G` green (positive modifiers), `V` dark green (event text), `E` teal (large text blocks), `C` cyan (concept text → itself spawns a tooltip), `B` cyan-blue (pop effects), `M` purple (rare techs), `_` magenta (placeholders), `c` blue-green (leader trait), `v` faded green (veteran leader trait), `d` tan (destiny trait), `r` light purple (renowned), `l` light green (legendary).
- Newline in loc values is `\n` (literal backslash-n inside the quoted string). Other escapes: `\t` tab, `\"`.
- Icon/flag/variable insertion inside text uses `[ ]` data functions, e.g. `[This.GetName]`, `[leader_trait_adaptable]`, `[trait:leader_trait_x]`. `'…'` single quotes create a nested-tooltip concept link (hover opens a secondary tooltip — avoid if you don't want the double hover).
- When scripting edits, remember the file is UTF-8 **with BOM** for localisation: preserve the BOM or the whole file may fail to load.

## Scopes (very important)
- Most effects/triggers only work in specific scopes and produce errors (`cannot find`, wrong target, silently no-op) otherwise. Always know the **current scope** and the **Output Scope** of each link. Docs: https://stellaris.paradoxwikis.com/Scopes
- Scope rules that bite most often:
  - `create_leader`, `add_resource`, `set_variable` (country vars) → **country** scope. `add_trait`, `change_leader_class`, `kill_leader`, leader flags → **leader** scope.
  - `last_created_leader` scopes to the last leader created anywhere — only reliable *immediately* after a `create_leader`.
  - `add_trait` prints `ADD_TRAIT: "$LEADER$ gains the $TRAIT$ trait, granting the following effects:\n$EFFECTS$\n"`. The `$EFFECTS$` block renders **richly (line breaks, colours, bullets) only in leader scope**; in country scope it collapses onto one line. To show a formatted trait tooltip use `tooltip = { add_trait = … }` while scoped to a leader.
  - In an `effectButtonType`, `THIS` = selected object or player country; `FROM` = player country (see vanilla `common/button_effects/example.txt`).
- **Event targets are ephemeral**: `save_event_target_as` survives only within one unbroken event chain. Separate button clicks (separate events) do NOT share event targets — persist a handle via a **country variable + leader/country flag** and re-find it with `every_owned_leader = { limit = { has_leader_flag = … } }` / `every_country = { … }`.
- Useful scope links: `owner`, `from`/`fromfrom`, `capital_scope`, `last_created_leader`, `every_owned_leader`/`any_owned_leader`, `random_owned_leader`, `ruler`, `overlord`, `planet`/`sector` for governors, `ship`/`fleet` for admirals.
- `event_leader = yes` in `create_leader` makes a hidden "event leader" (used for special named leaders). It may not appear in normal owned-leader scopes/pool — account for that when looking leaders up.
- When something "does nothing", the cause is almost always a scope mismatch — check `logs\error.log` (e.g. `Unable to add trait for [reason] leader_class_does_not_match`).

## Editing workflow (what works well here)
- Shell is Windows PowerShell 5.1 with no `grep`/`sed`/`tail`; use the Read/Grep/Glob tools for inspection.
- Files are CRLF, usually without BOM — preserve line endings (multi-line matches must include `\r\n`).
- For bulk/structured edits write a small Node script into `%LOCALAPPDATA%\Temp\opencode\` and run it: locate a section by its header string, use `indexOf` of the next header as the end boundary, assert occurrence counts, then replace. Never blind-replace a string that can occur in several windows.
- Verify after every edit: (1) `{` count == `}` count, (2) edited element names still unique per window, (3) content stays inside the intended visible bounds, (4) re-read the changed lines.
- Two very common traps: dropping elements a script expects by name (shows as `Could not find icon/window "x"`) and changing shared strings across unrelated windows.

## Testing & debugging
- A full game restart is the only guaranteed reload; the in-game console (`~`) accepts `reload <file.gui>` and `reload texture all` for UI-only iteration.
- `guibounds` draws every element's box + name — the primary tool to find where an element actually lands and whether a code-driven window exists at all.
- `debugtooltip` (wireless launch option `-debug_mode`) for ids/tooltips.
- Always check `...\logs\error.log` after a run for `Malformed token`, `Could not find ... in window ...`, unmatched braces.
- Diagnosing visually: diff the same window between vanilla and the mod you're patching, and check whether that mod also overrides textures/`.gfx`.

## Pre-flight checklist
1. Open the vanilla (and relevant mod) copy of every file you touch and diff before editing.
2. Confirm *why* your version wins: same path? unique window name? load order?
3. If icons are involved, check atlas dimensions against `noOfFrames`.
4. Prefer minimal, scoped overrides over whole-file replacements.
5. Re-verify braces/uniqueness/bounds and the log, then have a human confirm in-game (icon/layout bugs are visual).
