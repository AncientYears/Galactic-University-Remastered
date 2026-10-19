# PROJECT NOTES — Galactic University Remastered

Project-specific notes for *this* mod. Generic Stellaris modding knowledge lives in `AGENTS.md`; do not duplicate it here. `AGENTS.md` is intentionally project-neutral.

## What this mod is
Adds a "custom leader maker": a diplomatic-style GUI (`event galuni.2`) where the player picks a leader class and any traits, paying energy per trait.

## How to test
In-game console (~): `event galuni.2`. Then pick class → traits → Hire.
After any run, read `Documents\Paradox Interactive\Stellaris\logs\error.log`.

## Architecture / conventions
- Two-phase UI driven by global flags: `galUni_hiring_phase_one`, `galUni_hiring_phase_two`.
- Class choice → country variable `galUni_leader_type` (0 = scientist, 1 = commander, 2 = official).
- Per-trait selection state, two parallel stores that MUST stay in sync:
  - Global flag `galUni_trait_<TYPE>_<NAME>_selected` → drives the UI button state.
  - Country variable `galUni_<trait_id>` → consumed at creation (e.g. `galUni_basic_adaptable`).
- Running cost accumulator: country variable `galUni_hiringCost`.
- Trait types: `BASIC` and `SPECIALISED`.

## Key files
- `common/button_effects/galUni_traitMenuEffects.txt`
  - `galUni_leader_create_button_effect` — creates the leader, applies traits, then `galUni_reset_hiring_menu`.
  - Per-trait triples `..._button_effect`, `..._selected_`, `..._disabled_`.
- `common/inline_scripts/galUni_trait_inline.txt` — shared toggle logic for every trait button (takes `$flag$ $type$ $trait_name$ $trait_id$ $trait_tool$ $cost$`).
- `common/scripted_effects/galUni_effects.txt` — `galUni_reset_hiring_menu`, `galUni_trait_enable_disable_effect`, `galUni_move_to_phase_two`.
- `common/scripted_triggers/galUni_triggers.txt`, `common/scripted_loc/galUni_locStuff.txt`, `common/scripted_variables/galUniVars.txt` (`@galUni_base_*_price`).
- `events/galUni.txt` — `galuni.1/.2/.3`.
- `interface/something.gui` (window `someContainerDiplo`), `interface/something.gfx`.
- `localisation/galTestLoc_l_english.yml`.

## Tooltip mechanics for leader traits (hard-won)
- `custom_tooltip = "['trait:leader_trait_x']"` (single quotes inside are REQUIRED) shows the trait name as an underlined link — BUT hovering it opens a SECOND nested tooltip (name + `_desc` only, no effects). Awkward: requires hovering twice. **Do not use this.**
- `custom_tooltip = "leader_trait_x"` (the bare loc key, no `['trait:…']` wrapper) shows the trait NAME inline as plain text, no nested hover. Loc version number (`:0`, `:3`, …) does not matter when referenced. ← **current approach**
- Effects text is produced by `tooltip = { add_trait = leader_trait_x }` → "gains the X trait, granting the following effects: …".
- Pattern used per trait button (final):
  1. `every_owned_leader = { limit = { has_leader_flag = galUni_preview } tooltip = { add_trait = $trait_tool$ } }` (inline preview in leader scope → rich vanilla formatting: name + effects with `\n`, colours, bullets)
  2. `custom_tooltip = "$trait_tool$_desc"` (flavour)
- **Why the preview leader**: `tooltip = { add_trait = X }` renders the `ADD_TRAIT` loc string (`"$LEADER$ gains the $TRAIT$ trait, granting the following effects:\n$EFFECTS$\n"`). In **country** scope `$EFFECTS$` flattens (no `\n`/colours); in **leader** scope it renders the vanilla-formatted effect list. So we keep a transient preview leader around during phase two.
- **Scopes caveat**: `save_event_target_as` targets only survive within one unbroken event chain, NOT across separate button_effect clicks. So the preview leader is found via `every_owned_leader = { limit = { has_leader_flag = galUni_preview } }` (leader flag), not an event target. Country variable `galUni_preview_leader_exists` guards duplicate creation.
- Preview leader is created on the class buttons (`galUni_create_preview_leader_effect`), destroyed by `galUni_reset_hiring_menu` (`galUni_destroy_preview_leader_effect`).
- **Create button tooltip leaks**: any raw effect put directly in the button's `effect` shows as tooltip text (e.g. `X's Skill level set to 0`, `Subtracts 1.00 from the … variable`, `… dies`). Wrap all game-affecting plumbing in `hidden_effect = { … }`; keep only `custom_tooltip` + `tooltip = { add_trait = … }` lines visible. The create button shows `custom_tooltip = "Hire this leader"`, `custom_tooltip = "totalCost"`, then per-selected-trait `tooltip = { add_trait = … }` previews (scoped to the preview leader for rich formatting), followed by a `hidden_effect` that does the real create/trait-apply/reset.
- `totalCost` loc uses `[This.galUni_hiringCost]` (data function); the old `$@galUni_hiringCost$` form does not resolve; energy value shown in `§Y` (yellow).
- **Scope trap that broke trait application**: inside `last_created_leader = { … }` (leader scope), `check_variable = { which = galUni_basic_adaptable … }` reads a variable *on the leader*, not the country → never matches → no traits applied. Wrap with `owner = { check_variable = … }`. Same for the class check (`galUni_leader_type`).
- `kill_leader` requires block form: `kill_leader = { }` (NOT `kill_leader = yes`).
- `create_leader = { name = random … }` is valid vanilla syntax.
- Per-empire menu state uses **country flags** (migrated from global flags). All menu flags (`galUni_hiring_phase_one`/`_two`, `galUni_hiring_menu_opened`, `galUni_trait_<TYPE>_<NAME>_selected`, `galUni_PHASE_TWO_<class>`) are set via `set_country_flag`/`remove_country_flag` and checked via `has_country_flag`.
- Scope convention after migration: in button-effect `potential`/`allow` (THIS may be a non-country selected object) and inline-script trigger checks, reference flags as `from = { has_country_flag = X }` (FROM = player country). In `effect` top-level setters, wrap as `from = { set_country_flag/remove_country_flag = X }`. Inside `galUni_reset_hiring_menu` and inside `root = { }` blocks the scope is already the country → bare `set_country_flag`/`remove_country_flag`.
- Country-flag migration verified in-game (class select, trait toggle, create, reopen; plus a second empire — no leakage).
- **Storage country** (`galUni_trait_storage` in `common/country_types/galUniCountry.txt`): created on game start by `galuni.100` (hooked via `common/on_actions/galUni_on_actions.txt` → `on_game_start`), saved as `save_global_event_target_as = galUni_storage`. Purpose: host the preview leader so trait tooltips render richly *without* adding a leader to the player's roster or firing hired/died notifications. For existing saves, run `event galuni.100` manually; `galUni_ensure_storage_effect` also lazily creates it (guarded by global flag `galUni_storage_created`) so it self-heals.
- **CRITICAL scope-link name**: saved global event targets are read back with **`event_target:NAME`**, NOT `global_event_target:NAME`. `save_event_target_as` and `save_global_event_target_as` are both accessed via `event_target:`. Using the wrong prefix makes Stellaris treat it as a scripted-effect name → `Script Error: Invalid scripted effect` and the whole effect misbehaves (previously killed the player's leader). The storage country is `event_target:galUni_storage`.
- Every storage-target use is guarded with `has_global_flag = galUni_storage_created` so it can't misfire when the country doesn't exist.
- Pattern cross-checked against Gigastructural Engineering (`common/country_types/giga_utility_country_types.txt` → `giga_global_situation`, created in `events/giga_002_menu.txt:54`, accessed as `event_target:giga_global_situation_country`). Our country_type mirrors giga's: `government = no`, `default_ships = no`, `share_communications = no`, `ai = { enabled = no }`, faction `auto_delete = no`, modules `standard_event_module` + `exclusive_diplomacy_module { contact_rule = on_action_only }`.
- Storage-country approach verified working in-game (new game).
- `ADD_TRAIT` loc (`triggers_effects_l_english.yml:1412`) always prefixes `$LEADER$`; preview leader named "Student" so hover reads "Student: Student gains the … trait". Accepted for now.
- When using `String.replace`/generators, note `$'`, `$&`, `$$` are special in the replacement string — in JS use a function replacer; elsewhere escape appropriately.

## Bugs found & fixed (this session)
- **ROOT CAUSE — every trait button dead (black/disabled, not clickable, no highlight, leader created with no traits)**: `galUni_trait_inline.txt`'s `potential` contained `if = { limit = { $type$ = SPECIALISED } OR = { $leader_type$ } }`. `if`/`$type$ = X`/`$leader_type$` are not valid triggers, so the button_effect failed to load. The single working-looking button (architectural interest) was the only inline call that passed `leader_type`, so only it expanded cleanly. Every other call left a literal `$leader_type$` token → whole button effect invalid → button rendered dark/disabled and did nothing → no `galUni_<id>` variables set → leader created with no traits.
  - Fix: rewrote the inline to a valid shape — `potential = { has_global_flag = galUni_hiring_phase_two }`, `allow = { always = yes }`, then `effect` with tooltips + `hidden_effect` toggle/cost/variable. No `$leader_type$` use.
  - Diagnosed from `logs/error.log`: `Error in scripted trigger, cannot find: effect / hidden_effect / set_variable / $leader_type$ / galUni_trait_enable_disable_effect` reported against `galUni_trait_inline.txt` lines — i.e. effect tokens being parsed as triggers because the button_effect never parsed.
- **Leader created with no traits (secondary)**: `galUni_leader_create_button_effect` had an extra `}` (~old line 744) that closed the country `owner = {` block early, so trait `if` checks ran in `last_created_leader` scope. Fixed by restructuring to run fully in country scope; each trait is `if = { check_variable = {which=galUni_<id> value >= 1} } last_created_leader = { add_trait = … } subtract_variable = {which=galUni_<id> value = 1}`.
- **Leader still traitless + energy deducted**: restructured the whole create body under an explicit `from = { … }` scope. In a button_effect THIS = selected object *or player country*, `from` = player country (see vanilla `common/button_effects/example.txt`). `create_leader`, the class `if`/`else_if`, and all trait `if`s now run inside `from = { … }`, guaranteeing country scope. `last_created_leader` scopes to the last created leader from anywhere (vanilla `scopes.log`). (Pending in-game confirm.)
- **Architectural Interest tooltip differed from others**: it was the only inline call passing `leader_type`, and its specialisation `selected`/`disabled` potential duplicated class checks. Removed `leader_type` param; gave the button's own `potential` the class guard `OR = { check_variable leader_type=0  check_variable leader_type=2 }` (official/scientist), matching the trait's `leader_class = { official scientist }`; `disabled` now negates that.
- **Deselect didn't deselect**: inline toggle-off branch set `$trait_id$ = 1` (copied from select branch). Now sets `0`.
- **Hidden-effect noise**: state changes wrapped in `hidden_effect = { … }` so raw "Changes the … variable" lines stop leaking into tooltips.
- **Skill/age**: renamed random vars to `galUni_new_leader_skill` / `galUni_new_leader_age` (collided with `leader_skill`/`leader_age`).

## Known remaining issues
- **Opposite-trait exclusivity was dropped** in the inline rewrite: the old guards `if = { limit = { $trait_name$ = EAGER } … }` were also invalid (`EAGER = EAGER`). Re-add correctly by putting the clear-others block in each affected *button_effect* (guarded by that trait's own `..._selected` global flag, after the inline call) rather than in the shared inline.
- `custom_tooltip = "['trait:$trait_tool$']"` logs `Missing localization key [...]` but still renders the trait link in-game.
- Several `custom_tooltip = "Hire …"`/`"Close …"` strings in button effects have no loc keys (cosmetic).

## Conventions / gotchas
- Files are CRLF, usually no BOM; tabs for indentation. Preserve when scripting edits.
- Validate after edits: `{` count == `}` count (strip `#` comments first), element names unique per window.
- 3 duplicate entries per trait (button_effect + selected + disabled) — bulk trait addition is the remaining grind (~16 of 100+ wired).

## TODO
- Harvest vanilla `common/traits/*.txt` and auto-generate the 3-part button-effect + inline entries for all traits.
- Populate missing localisation keys (`galUni_trait_tooltip_*`, `_desc`).
- `localisation/galTestLoc_l_english.yml:17` `totalCost` uses non-resolving `$@galUni_hiringCost$` — fix.
