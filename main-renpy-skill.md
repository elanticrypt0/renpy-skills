---
name: renpy
description: >
  Use this skill whenever the user wants to create, scaffold, architect, or improve a Ren'Py visual novel project.
  Triggers include: starting a new Ren'Py game, structuring a Ren'Py project, writing Ren'Py script (.rpy files),
  asking about best practices, dialogue systems, character definitions, screen language, transforms, transitions,
  image/audio management, save/load, flags and variables, routing/branching, menus, minigames, maps, inventory,
  RPG stats, persistent data, survival mechanics, time/day cycles, quest systems, sandbox free-roam,
  relationship trees, scene unlocking, gallery systems, NPC scheduling, adult content gating,
  or anything related to Ren'Py development.
  Use this even when the user doesn't say "Ren'Py skill" — if they mention labels, jumps, calls, `define`, `default`,
  `screen`, `.rpy` files, or visual novel mechanics, this skill is relevant.
  Primary focus: Visual Novels with optional subsystems (RPG stats, maps, inventory, minigames, survival, quests, sandbox/adult VN).
---

# Ren'Py Visual Novel Project Skill

This skill guides the creation and architecture of Ren'Py projects with professional standards:
clean, readable script, reusable components, clear separation of concerns, and maintainability at scale.

**Default scope**: Visual Novel with dialogue, branching, characters, and saves.  
**Extended scope**: See reference files when the project includes subsystems:
- RPG/stats systems → `references/renpy-rpg.md`
- Maps and world navigation → `references/renpy-maps.md`
- Minigames → `references/renpy-minigames.md`
- UI / Screen Language deep-dive → `references/renpy-screens.md`
- Survival stats & time/day cycle → `references/renpy-survival.md`
- Quest / mission system → `references/renpy-quests.md`
- Inventory system (5 item types, equipment, gifts, quest items) → `references/renpy-inventory.md`
- Sandbox / adult VN (relationship trees, scenes, gallery, NPC scheduling) → `references/renpy-adult-vn.md`

---

## 1. Project Structure

```
game/
├── script.rpy               # Entry point: splash, init, main_menu jump
├── options.rpy              # Core Ren'Py config (generated, keep clean)
├── gui.rpy                  # GUI style overrides (generated)
│
├── characters/
│   └── characters.rpy       # All Character() definitions
│
├── story/
│   ├── chapter_01.rpy       # One file per chapter / act
│   ├── chapter_02.rpy
│   └── endings.rpy
│
├── screens/
│   ├── hud.rpy              # Custom HUD overlays
│   ├── inventory.rpy        # Inventory screen (if used)
│   ├── map.rpy              # World map screen (if used)
│   └── stats.rpy            # Stats/status screen (if used)
│
├── systems/
│   ├── flags.rpy            # All default/define for game state
│   ├── inventory_system.rpy # Inventory logic (if used)
│   ├── stats_system.rpy     # RPG stats logic (if used)
│   ├── debug_system.rpy     # Optional: debug toggles, panel (if used)
│   └── audio_manager.rpy    # Music/sfx helpers (if used)
│
├── minigames/               # Optional: one file per minigame
│   ├── blackjack.rpy
│   └── memory.rpy
│
├── gallery/                 # Optional: scene registry + gallery screen
│   ├── scenes_registry.rpy
│   └── gallery_screen.rpy
│
├── tl/                      # Optional: translations (i18n)
│   ├── english/
│   └── portuguese/
│
├── images/
│   └── (Ren'Py auto-detects images here)
│
├── audio/
│   ├── music/
│   └── sfx/
│
└── fonts/
```

**Rules:**
- `script.rpy` only sets up init and jumps to the first scene — no story content there.
- One chapter / arc per `.rpy` file. Never put all story in one file.
- All variable declarations (`default`, `define`) live in `systems/flags.rpy` unless tightly scoped to one system.
- Screen definitions belong in `screens/`, never inline in story files.

---

## 2. Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Labels | `snake_case` | `label chapter_01_park_scene:` |
| Variables | `snake_case` | `default player_name = "Alex"` |
| Flags (bool) | `snake_case`, descriptive past | `default met_elara = False` |
| Characters | `UPPER` short name | `define e = Character("Elara")` |
| Images | `tag attribute...` | `show elara happy` |
| Screens | `snake_case` | `screen inventory_screen():` |
| Transforms | `snake_case` | `transform shake:` |
| Styles | `snake_case` | `style dialogue_box:` |

---

## 3. Characters

Define all characters in `characters/characters.rpy`. Never define inline in story files.

```renpy
# characters/characters.rpy

# ── Player ────────────────────────────────────────────────────────────
define mc = Character(
    "[player_name]",
    color="#e8d5b0",
    who_suffix=" (you)",
)

# ── Main Cast ─────────────────────────────────────────────────────────
define e = Character(
    "Elara",
    color="#a8d8ea",
    image="elara",                   # ties to image tag for auto-expression
    voice_tag="elara",               # for voice acting support
    what_prefix="\"",
    what_suffix="\"",
)

define n = Character(
    "Narrator",
    color="#d4c5a9",
    kind=nvl,                        # NVL style narration
)

# ── Special / System Characters ───────────────────────────────────────
define sys = Character(
    None,
    color="#888888",
    what_style="system_text",        # for system messages, tutorials
)
```

**Tips:**
- Use `image=` tag so Ren'Py automatically links `show elara happy` to that Character.
- Create a `kind=nvl` narrator only if you use NVL mode; never mix ADV and NVL in the same scene without intent.
- Keep voice tags consistent — use the same identifier across all audio files.

---

## 4. Script / Story Standards

### Label Hierarchy

```renpy
# chapter_01.rpy

label chapter_01:
    jump chapter_01_intro

label chapter_01_intro:
    scene park_day with dissolve
    play music "audio/music/peaceful_afternoon.ogg" fadein 1.0

    show elara neutral at left with moveinleft
    e "You made it! I was starting to worry."

    jump chapter_01_choice_01

label chapter_01_choice_01:
    menu:
        "I got a little lost.":
            $ met_elara_warmly = True
            jump chapter_01_path_warm
        "I'm always on time.":
            jump chapter_01_path_cool

label chapter_01_path_warm:
    e happy "That's adorable, actually."
    jump chapter_01_park_walk

label chapter_01_path_cool:
    e surprised "Right. Of course you are."
    jump chapter_01_park_walk

label chapter_01_park_walk:
    # ... continues
    return
```

### Rules:
- **One label per narrative beat** — don't chain hundreds of lines under one label.
- **`jump` for non-returning flow**, `call` when you need to return (cutscenes, subroutines).
- **`return` at the end of every `call`-able label** — never fall through.
- **Inline Python blocks** (`$` or `python:`) for logic only — never for display.
- **Comments** before every scene change, every major branch, every flag set.

### Python Blocks

```renpy
# ✅ Good — logic only, clearly commented
$ # Player chose the empathy route
$ met_elara_warmly = True
$ relationship_elara += 5

# ✅ Multi-line when needed
python:
    # Award bonus if all side quests done
    if side_quest_a_done and side_quest_b_done:
        bonus_ending_unlocked = True
        renpy.notify("All side quests complete!")

# ❌ Avoid — display logic in python blocks
python:
    renpy.show("elara", ["happy"])   # use show statement instead
```

---

## 5. Variables & Flags (`systems/flags.rpy`)

All game state lives here. Separate by system with clear comments.

```renpy
# systems/flags.rpy

# ── Player Info ────────────────────────────────────────────────────────
default player_name = "Alex"
default player_pronouns = "they/them"

# ── Story Flags ────────────────────────────────────────────────────────
default met_elara = False
default met_elara_warmly = False
default chapter_01_complete = False
default true_ending_unlocked = False

# ── Relationship Values (0–100) ────────────────────────────────────────
default relationship_elara = 50
default relationship_marcus = 50

# ── Inventory / Items (use inventory_system.rpy for logic) ────────────
default inventory = []
default gold = 0

# ── Persistent Flags (survive New Game) ───────────────────────────────
# Use 'define' + persistent for cross-playthrough state
define persistent.seen_true_ending = False
define persistent.unlocked_gallery = []
```

**Rules:**
- `default` — resets on new game. Use for all story/gameplay variables.
- `define` — set once at init, constant. Use for Character objects, transforms, config.
- `persistent.*` — survives game resets. Use for gallery unlocks, achievements, NG+ flags only.
- Never scatter `default` statements across story files — always centralize in `flags.rpy`.

---

## 6. Images & Sprites

Ren'Py auto-detects images by filename. Follow the `tag attribute attribute` naming convention:

```
images/
├── bg/
│   ├── bg_park_day.png           → show bg park day
│   ├── bg_park_night.png         → show bg park night
│   └── bg_classroom.png          → show bg classroom
├── elara/
│   ├── elara_neutral.png         → show elara neutral
│   ├── elara_happy.png           → show elara happy
│   ├── elara_sad.png             → show elara sad
│   └── elara_surprised.png       → show elara surprised
└── cg/
    └── cg_elara_letter.png       → show cg elara letter
```

For layered sprites (outfit changes, etc.), use `LayeredImage`:

```renpy
# characters/characters.rpy or a dedicated images.rpy

layeredimage elara:
    always "elara/elara_base.png"

    group expression:
        attribute neutral default "elara/elara_neutral.png"
        attribute happy "elara/elara_happy.png"
        attribute sad "elara/elara_sad.png"

    group outfit:
        attribute school default "elara/elara_outfit_school.png"
        attribute casual "elara/elara_outfit_casual.png"
```

---

## 7. Transitions & Transforms

Define all reusable transitions and transforms in `script.rpy` or a dedicated `transforms.rpy`.

```renpy
# Common transitions — use these consistently, don't invent ad-hoc
define quick_fade = Fade(0.2, 0.0, 0.2)
define slow_fade  = Fade(0.5, 0.0, 0.5)
define scene_change = dissolve  # alias for readability

# Reusable transforms
transform at_left:
    xalign 0.2 yalign 1.0

transform at_right:
    xalign 0.8 yalign 1.0

transform at_center:
    xalign 0.5 yalign 1.0

transform shake:
    parallel:
        ease 0.05 xoffset -15
        ease 0.05 xoffset 15
        ease 0.05 xoffset -10
        ease 0.05 xoffset 0
    on hide:
        alpha 0

transform fade_in:
    alpha 0.0
    linear 0.4 alpha 1.0
```

**Rules:**
- Never use raw numbers in story files — always named transforms.
- `with dissolve` for scene changes; `with move` for character repositioning.
- Define `at_left`, `at_center`, `at_right` once; use everywhere.

---

## 8. Audio

```renpy
# In story files — always named, always with fade
play music "audio/music/peaceful_afternoon.ogg" fadein 1.0
stop music fadeout 1.5
play sound "audio/sfx/door_open.ogg"

# For loops and layers use channel management
play music "audio/music/tension.ogg" loop fadein 0.5
queue music "audio/music/resolution.ogg" fadein 1.0   # plays after current track

# Audio system helper (optional, in systems/audio_manager.rpy)
init python:
    def play_music(track, fadein=1.0, loop=True):
        renpy.music.play("audio/music/" + track + ".ogg", fadein=fadein, loop=loop)

    def stop_music(fadeout=1.5):
        renpy.music.stop(fadeout=fadeout)
```

**Rules:**
- Never hardcode raw file paths in story content — wrap in helpers or use defines.
- Always `fadein`/`fadeout` for music. SFX can be instant.
- Use `queue music` for seamless track transitions.

---

## 9. Screens (Overview)

Define all screens in `screens/`. For deep screen language guidance, read `references/renpy-screens.md`.

```renpy
# screens/hud.rpy — minimal example

screen hud():
    zorder 10
    style_prefix "hud"

    # Relationship meter example
    if show_relationship_hud:
        frame:
            xalign 1.0 yalign 0.0
            xoffset -20 yoffset 20
            hbox:
                spacing 8
                text "Elara" size 18
                bar value relationship_elara range 100 xsize 120

screen quick_menu():
    zorder 100
    hbox:
        xalign 0.5 yalign 1.0 yoffset -20
        spacing 12
        textbutton "Save"    action ShowMenu("save")
        textbutton "Load"    action ShowMenu("load")
        textbutton "Log"     action ShowMenu("history")
        textbutton "Skip"    action Skip()
        textbutton "Auto"    action Preference("auto-forward", "toggle")
        textbutton "Menu"    action MainMenu()
```

---

## 10. Save System

Ren'Py handles saves automatically. Extend it cleanly:

```renpy
# In options.rpy — set save slot count
config.has_autosave = True
config.autosave_slots = 3
config.save_directory = "my-game-saves"

# Trigger saves at chapter boundaries
label chapter_01_end:
    $ chapter_01_complete = True
    $ renpy.notify("Chapter 1 complete — game saved.")
    $ renpy.save_persistent()    # flush persistent data
    # Ren'Py autosave will handle the slot save
    jump chapter_02
```

For save metadata (chapter name, scene thumbnail):

```renpy
# Wrap scene entry points to update save context
label chapter_02_library:
    $ save_name = "Chapter 2 — The Library"   # shows in save menu
    scene library_afternoon with dissolve
    # ...
```

---

## 11. Branching & Routing Patterns

### Simple Flag Gate
```renpy
if met_elara_warmly:
    e happy "I knew you'd come back!"
else:
    e neutral "Oh. You're here."
```

### Multi-Condition Routing
```renpy
label ending_check:
    if true_ending_unlocked and relationship_elara >= 80:
        jump ending_true
    elif relationship_elara >= 60:
        jump ending_good
    else:
        jump ending_neutral
```

### Avoid Deep Nesting
```renpy
# ❌ Avoid — hard to trace, hard to edit
label scene_a:
    menu:
        "Option A":
            if flag_x:
                if flag_y:
                    "..."
                    menu:
                        # now 4 levels deep...

# ✅ Prefer — flat labels, easy to navigate
label scene_a:
    menu:
        "Option A":
            jump scene_a_option_a
        "Option B":
            jump scene_a_option_b

label scene_a_option_a:
    # ...
    jump scene_a_rejoin

label scene_a_option_b:
    # ...
    jump scene_a_rejoin

label scene_a_rejoin:
    # ...
```

---

## 12. Accessibility & Quality Checklist

Before shipping any label block:
- [ ] All characters defined in `characters/characters.rpy`
- [ ] All variables declared in `systems/flags.rpy` with `default`
- [ ] No magic numbers in script — use named transforms and defines
- [ ] Scene changes use named transitions, not raw `dissolve` everywhere
- [ ] Every `call` label ends with `return`
- [ ] Music changes use `fadein`/`fadeout`
- [ ] Screen definitions are in `screens/`, not inline
- [ ] Chapter boundary sets `save_name`
- [ ] `persistent.*` used only for cross-playthrough data
- [ ] No story content in `script.rpy` (entry point only)
- [ ] Deep nesting flattened into separate labels
- [ ] i18n: `config.has_autosave_locale_*` enabled if shipping with translations
- [ ] If mobile target: variant-aware screens for `small` / `touch`
- [ ] If adult content: +18 disclaimer splash present and gated by `persistent.adult_confirmed`
- [ ] Debug mode (if used) disabled in release build (`config.developer = False`)

---

## 12.5. Mobile / Multi-Platform

If targeting Android (or iOS where allowed), Ren'Py provides `renpy.variant` for per-platform UI:

```renpy
# In screen definitions
screen game_hud():
    if renpy.variant("small"):
        use game_hud_mobile()
    else:
        use game_hud_pc()
```

**Common variants**:
- `"pc"` — desktop default
- `"small"` — typically Android phones
- `"touch"` — any touchscreen device
- `"tablet"` — tablet form factor

**Mobile-specific considerations**:
- **Touch targets ≥ 44×44 px** (Apple HIG / Material Design guideline). Increase hitboxes on `imagebutton`.
- **No hover state** on mobile — replace hover tooltips with long-press or alternative cues.
- **Landscape only** for VN content (force `config.gl_resize` and orientation in `options.rpy`).
- **Smaller text legibility**: bump dialogue text size by 1.1× on `small` variant.
- **Save slots layout**: vertical scrolling vs grid on PC.

To test mobile layouts on desktop, set `config.variants = ["small", "touch"]` temporarily.

See `references/renpy-screens.md` § Responsive Layouts for full HUD examples.

---

## 12.6. Debug Mode (Brief Reference)

Two-tier debug system pattern:

- **Debug Full** — enabled when `config.developer = True` (development build). Adds Ren'Py's built-in console (Shift+O), dev menu (Shift+D), Shift+R reload, plus a custom panel with stat editors, time skips, location unlocks, etc.
- **Debug Lite** — accessible in release builds via a hidden sequence (e.g., 7 clicks on logo within 5s, or `Ctrl+Shift+M`). Reduced toggle set, no Python console. Persistent in `persistent.debug_lite_enabled`.

Watermark "DEBUG MODE" in the corner whenever any debug is active. Achievements unlocked in a debug session should not count toward cross-save persistent flags.

(Full pattern lives in a project-specific `systems/debug_system.rpy` — consider extracting a reusable skill if you ship multiple Ren'Py games.)

---

## 13. Extended Subsystems

When the project grows beyond core VN mechanics, read the relevant reference:

| Feature | Reference File |
|---|---|
| RPG stats, leveling, skills, combat | `references/renpy-rpg.md` |
| World maps, location navigation | `references/renpy-maps.md` |
| Minigames (puzzles, rhythm, QTE, cards) | `references/renpy-minigames.md` |
| Complex UI / Screen Language | `references/renpy-screens.md` |
| Survival stats, time/day cycle, action economy | `references/renpy-survival.md` |
| Quest / mission system, objectives, rewards | `references/renpy-quests.md` |
| Inventory: 5 item types, equipment, gifts, quest items, shops | `references/renpy-inventory.md` |
| Sandbox / adult VN: relationship trees, scene unlock, gallery, NPC scheduling | `references/renpy-adult-vn.md` |

Each reference file is self-contained. Read only what the project needs.

**Cross-skill dependencies — heads-up**: the inventory pattern is touched by several skills (`renpy-quests` via `ItemObjective`, `renpy-adult-vn` via gifts, `renpy-screens` via the inventory screen, `renpy-rpg` via potions/equipment). `renpy-inventory.md` is the **canonical source**. The other skills define what they need locally and cross-ref back. Pick `renpy-inventory.md` early if your project will have any meaningful item economy beyond healing potions.

### Subsystem combination guide

For **sandbox life-sim / adult VN** projects (multiple characters, free-roam hub, survival loop):
1. Start with `renpy-survival.md` — establish time cycle and stat management
2. Add `renpy-inventory.md` — items, consumables, gifts for NPCs
3. Add `renpy-quests.md` — define character quest chains (uses `ItemObjective` from inventory)
4. Add `renpy-adult-vn.md` — wire relationship trees, scene gating, gallery, NPC schedules, gifting
5. Add `renpy-maps.md` only if the world is large enough to need a visual map
6. Add `renpy-rpg.md` only if there are combat mechanics (separate from survival stats)

For **sandbox political / corruption sim** (life-sim with multi-dimensional reputation, NTR-style asymmetric progression, satirical tone):
1. Start with `renpy-survival.md` — time cycle, slot economy, scheduled bills, debt ramp, day-of-week gating
2. Add `renpy-inventory.md` — full 5-type system: consumables (energy/coca), equipment (suits), gifts (per-NPC affinity), quest items, key items, shops
3. Add `renpy-adult-vn.md` — multi-stage scene chains, multi-dimensional NPC stats, opaque qualitative indicators, city-wide rumor meter, achievement system
4. Add `renpy-quests.md` — staged quests, procedural quests, soft deadlines, category filtering
5. Add `renpy-minigames.md` — universal continuous difficulty, gambling session economy (buy-in + caps), stat modifiers, narrative interrupts
6. Add `renpy-maps.md` — flat sandbox variant, locked hotspots with reason tooltips, sub-zones, hotspot indicators
7. Add `renpy-screens.md` — HUD top/bottom bars, themed phone overlay, stat-change notification queue, auto-hide HUD, responsive layouts
8. **Skip** `renpy-rpg.md` — political sims rarely need combat or leveling

For **dating sim with combat** (visual novel with relationship trees + RPG combat segments):
1. `renpy-rpg.md` — stats, leveling, combat
2. `renpy-inventory.md` — potions / equipment / gift items unified
3. `renpy-adult-vn.md` — relationship trees, scene gating
4. `renpy-quests.md` — quest chains tying combat to story

For **small / minimal projects** (linear VN with a few collectibles):
1. `main-renpy-skill.md` only — use the minimal Inventory in `renpy-rpg.md` § 4 if needed
2. Skip `renpy-inventory.md` — overkill for a handful of pickups

## 14. Official docs

For specifics search here.

[Ren's py Docs main page](https://www.renpy.org/doc/html/)