# Ren'Py Sandbox / Adult VN Reference

Use this when building a sandbox-style visual novel with free-roam exploration, multiple character relationship trees, scene unlocking, and a gallery system. This is the integration reference for games that combine survival stats, quests, and maps into a full life-sim loop.

**Cross-skill dependencies**: this skill assumes `store.player_inventory` exists and exposes `.has(item_id, qty)`, `.add(item_id, qty)`, `.remove(item_id, qty)`. Set up via `renpy-inventory.md` (full system with gift/quest/key item types) or `renpy-rpg.md` § 4 (minimal Item/Inventory). For gift-driven relationship mechanics use the full inventory skill.

---

## Table of Contents
1. [Project Structure (Sandbox VN)](#1-project-structure-sandbox-vn)
2. [Character Relationship Trees](#2-character-relationship-trees)
3. [Scene Registry & Unlock System](#3-scene-registry--unlock-system)
4. [Hub & Free-Roam Screen](#4-hub--free-roam-screen)
5. [NPC Scheduling System](#5-npc-scheduling-system)
6. [Content Gating Patterns](#6-content-gating-patterns)
7. [Gallery Screen](#7-gallery-screen)
8. [Persistent Unlock Data](#8-persistent-unlock-data)
9. [Adult Content Flags](#9-adult-content-flags)
10. [Multi-Stage Scene Chains](#10-multi-stage-scene-chains)
11. [Multi-Dimensional NPC Stats](#11-multi-dimensional-npc-stats)
12. [Opaque NPC Stats — Qualitative Indicators](#12-opaque-npc-stats--qualitative-indicators)
13. [City-Wide Reputation / Scandal Meter](#13-city-wide-reputation--scandal-meter)
14. [Mini-Game Interruptions During Scenes](#14-mini-game-interruptions-during-scenes)
15. [NPC Autonomous Events (Off-Screen Cameos)](#15-npc-autonomous-events-off-screen-cameos)
16. [Achievement System with Bonus Content](#16-achievement-system-with-bonus-content)
17. [Adult Content Distribution Policy](#17-adult-content-distribution-policy)
18. [Full Integration Checklist](#18-full-integration-checklist)

---

## 1. Project Structure (Sandbox VN)

```
game/
├── script.rpy               # Entry: init, splash, main menu → hub
├── options.rpy
├── gui.rpy
│
├── characters/
│   └── characters.rpy       # Character() definitions
│
├── story/
│   ├── hub_events.rpy       # Hub transitions, day-start logic
│   ├── quests/
│   │   ├── quest_intro.rpy
│   │   ├── quest_sarah.rpy  # One file per character quest chain
│   │   └── quest_monica.rpy
│   └── scenes/
│       ├── scenes_sarah.rpy # Scenes for each character
│       └── scenes_monica.rpy
│
├── screens/
│   ├── hub_screen.rpy       # Free-roam hub / location selection
│   ├── survival_hud.rpy     # Stats + time + actions
│   ├── quest_journal.rpy    # Mission log
│   ├── gallery.rpy          # Scene replay gallery
│   └── character_status.rpy # Relationship status per character
│
├── systems/
│   ├── flags.rpy            # All default/define
│   ├── survival_system.rpy  # Energy, mood, money, fitness
│   ├── time_system.rpy      # Day/period cycle, action economy
│   ├── quest_system.rpy     # Quest state machine
│   ├── quest_data.rpy       # Quest definitions
│   ├── character_system.rpy # Relationship values, scene unlock logic
│   ├── npc_schedule.rpy     # Where each NPC is at each time period
│   └── inventory_system.rpy # Items
│
├── images/
│   ├── bg/                  # Locations
│   ├── sprites/             # Character sprites by expression
│   ├── cg/                  # Full-image scenes (CGs)
│   └── ui/                  # Hub icons, map images
│
└── audio/
    ├── music/
    └── sfx/
```

---

## 2. Character Relationship Trees

Each character has:
- A **relationship value** (0–100) that rises through interactions
- **Unlock thresholds** that gate story scenes
- A **seen-scenes tracker** for gallery replay
- Optional **item requirements** for certain scenes

```renpy
# systems/character_system.rpy

init python:

    class CharacterRelationship:
        """
        Tracks relationship progress and scene unlocks for one NPC.
        """

        def __init__(self, char_id, name, scenes=None):
            self.char_id  = char_id
            self.name     = name
            self.value    = 0       # 0–100
            self.scenes   = scenes or []   # ordered list of SceneDefinition

        def modify(self, delta):
            self.value = max(0, min(100, self.value + delta))

        def next_unlockable_scene(self):
            """Returns the next scene that has not been seen but requirements are met."""
            for scene in self.scenes:
                if not scene.is_seen() and scene.requirements_met(self):
                    return scene
            return None

        def all_unlocked_scenes(self):
            return [s for s in self.scenes if s.is_seen()]

        def progress_percent(self):
            total = len(self.scenes)
            seen  = len(self.all_unlocked_scenes())
            return int((seen / total) * 100) if total else 0


    class SceneDefinition:
        """
        Defines a single unlockable scene for a character.
        """

        def __init__(self, scene_id, title, thumbnail,
                     label, rel_required=0,
                     items_required=None, flags_required=None,
                     is_adult=False):
            self.scene_id       = scene_id
            self.title          = title
            self.thumbnail      = thumbnail   # image path for gallery
            self.label          = label       # story label to call
            self.rel_required   = rel_required
            self.items_required = items_required or []   # list of item_id strings
            self.flags_required = flags_required or []   # list of flag variable names
            self.is_adult       = is_adult    # True = requires adult content enabled

        def is_seen(self):
            return self.scene_id in store.unlocked_scenes

        def requirements_met(self, relationship):
            # Relationship value check
            if relationship.value < self.rel_required:
                return False
            # Item requirements
            inv = store.player_inventory
            for item_id in self.items_required:
                if not inv or not inv.has(item_id):
                    return False
            # Flag requirements
            for flag in self.flags_required:
                if not getattr(store, flag, False):
                    return False
            # Adult content gate
            if self.is_adult and not store.adult_content_enabled:
                return False
            return True

        def missing_requirements(self, relationship):
            """Returns a list of human-readable missing requirement strings."""
            missing = []
            if relationship.value < self.rel_required:
                missing.append("Relationship ≥ {}".format(self.rel_required))
            inv = store.player_inventory
            for item_id in self.items_required:
                if not inv or not inv.has(item_id):
                    missing.append("Need: {}".format(item_id))
            for flag in self.flags_required:
                if not getattr(store, flag, False):
                    missing.append("Complete: {}".format(flag))
            return missing
```

**Defining characters with their scene trees:**

```renpy
# systems/character_system.rpy (continued)

init python:

    def _build_characters():
        return {

            "sarah": CharacterRelationship(
                char_id = "sarah",
                name    = "Sarah",
                scenes  = [
                    SceneDefinition(
                        scene_id     = "sarah_scene_01",
                        title        = "First Meeting",
                        thumbnail    = "images/cg/sarah_scene_01_thumb.png",
                        label        = "scene_sarah_01",
                        rel_required = 0,
                    ),
                    SceneDefinition(
                        scene_id     = "sarah_scene_02",
                        title        = "Coffee Date",
                        thumbnail    = "images/cg/sarah_scene_02_thumb.png",
                        label        = "scene_sarah_02",
                        rel_required = 20,
                        flags_required = ["quest_sarah_01_complete"],
                    ),
                    SceneDefinition(
                        scene_id     = "sarah_scene_03",
                        title        = "Late Night",
                        thumbnail    = "images/cg/sarah_scene_03_thumb.png",
                        label        = "scene_sarah_03",
                        rel_required = 50,
                        items_required = ["item_wine"],
                        is_adult     = True,
                    ),
                ],
            ),

            "monica": CharacterRelationship(
                char_id = "monica",
                name    = "Monica",
                scenes  = [
                    SceneDefinition("monica_scene_01", "Chance Encounter",
                                    "images/cg/monica_01_thumb.png",
                                    "scene_monica_01", rel_required=0),
                    SceneDefinition("monica_scene_02", "Workout Partner",
                                    "images/cg/monica_02_thumb.png",
                                    "scene_monica_02", rel_required=30,
                                    flags_required=["fitness_threshold_met"]),
                ],
            ),

        }

    _CHARACTER_DATA = _build_characters()


default characters = None
default unlocked_scenes = []

init python:
    def init_characters():
        store.characters = _build_characters()
```

---

## 3. Scene Registry & Unlock System

```renpy
# systems/character_system.rpy (continued)

init python:

    def try_unlock_scene(char_id, scene_id):
        """
        Attempt to unlock a scene. Returns True if newly unlocked.
        Call after raising relationship or completing a quest.
        """
        char = store.characters.get(char_id)
        if not char:
            return False
        for scene in char.scenes:
            if scene.scene_id == scene_id and not scene.is_seen():
                if scene.requirements_met(char):
                    store.unlocked_scenes.append(scene_id)
                    renpy.notify("New scene available: {}".format(scene.title))
                    return True
        return False

    def play_scene(scene_id):
        """
        Mark a scene as seen and call its label.
        Use this as the canonical way to enter any scene.
        """
        if scene_id not in store.unlocked_scenes:
            store.unlocked_scenes.append(scene_id)
        renpy.call(scene_id_to_label(scene_id))

    def scene_id_to_label(scene_id):
        """Map scene_id → its story label. Looks up from character data."""
        for char in store.characters.values():
            for scene in char.scenes:
                if scene.scene_id == scene_id:
                    return scene.label
        return None

    def check_new_scenes_available(char_id):
        """After relationship change, check and notify about newly available scenes."""
        char = store.characters.get(char_id)
        if not char:
            return
        for scene in char.scenes:
            if not scene.is_seen() and scene.requirements_met(char):
                renpy.notify("{}: new scene available!".format(char.name))
                break
```

**In story — after raising relationship:**

```renpy
label talk_to_sarah_compliment:
    python:
        characters["sarah"].modify(+10)
        check_new_scenes_available("sarah")
        quest_manager.check_all_stat_objectives()
    "Sarah smiles warmly. You feel closer to her."
    jump hub_main
```

---

## 4. Hub & Free-Roam Screen

The hub is the central gameplay loop: pick a location, spend an action, trigger events or dialogue.

```renpy
# screens/hub_screen.rpy

init python:

    class HubLocation:
        """A visitable location in the hub."""
        def __init__(self, loc_id, name, icon, bg,
                     available_periods=None, entry_label=None,
                     unlock_condition=None):
            self.loc_id           = loc_id
            self.name             = name
            self.icon             = icon              # small icon for the hub map
            self.bg               = bg                # background image
            self.available_periods = available_periods or ["morning","afternoon","evening","night"]
            self.entry_label      = entry_label
            self.unlock_condition = unlock_condition

        def is_available(self):
            period = store.game_time.period if store.game_time else "morning"
            if period not in self.available_periods:
                return False
            if self.unlock_condition and not self.unlock_condition():
                return False
            return True

    HUB_LOCATIONS = {
        "apartment":   HubLocation("apartment",   "Your Apartment",   "ui/icon_apt.png",
                                   "bg/bg_apartment.png",
                                   entry_label="hub_apartment"),
        "gym":         HubLocation("gym",          "Gym",              "ui/icon_gym.png",
                                   "bg/bg_gym.png",
                                   available_periods=["morning","afternoon"],
                                   entry_label="hub_gym"),
        "shop":        HubLocation("shop",         "Corner Store",     "ui/icon_shop.png",
                                   "bg/bg_shop.png",
                                   available_periods=["morning","afternoon","evening"],
                                   entry_label="hub_shop"),
        "park":        HubLocation("park",         "City Park",        "ui/icon_park.png",
                                   "bg/bg_park.png",
                                   entry_label="hub_park"),
        "work":        HubLocation("work",         "Office",           "ui/icon_work.png",
                                   "bg/bg_office.png",
                                   available_periods=["morning","afternoon"],
                                   entry_label="hub_work"),
        "bar":         HubLocation("bar",          "The Bar",          "ui/icon_bar.png",
                                   "bg/bg_bar.png",
                                   available_periods=["evening","night"],
                                   entry_label="hub_bar"),
    }


screen hub_screen():
    zorder 5

    add "images/bg/bg_hub_overview.png"

    # Location buttons arranged on a map/grid
    python:
        available_locs = [loc for loc in HUB_LOCATIONS.values() if loc.is_available()]

    vbox:
        xalign 0.5 yalign 0.5
        spacing 16

        text "Where would you like to go?" xalign 0.5 size 20 bold True

        hbox:
            spacing 20
            xalign 0.5
            for loc in HUB_LOCATIONS.values():
                vbox:
                    spacing 4
                    xalign 0.5
                    imagebutton:
                        idle  "images/[loc.icon]"
                        xsize 80 ysize 80
                        sensitive loc.is_available()
                        action Jump(loc.entry_label)
                    text loc.name size 13 xalign 0.5 color ("#ecf0f1" if loc.is_available() else "#555")

    # Quick access buttons
    hbox:
        xalign 0.5 yalign 0.97
        spacing 20
        textbutton "Journal"  action ShowMenu("quest_journal_screen")
        textbutton "Gallery"  action ShowMenu("gallery_screen")
        textbutton "Inventory" action ShowMenu("inventory_screen")
        textbutton "Sleep" if game_time.period == "night" action Jump("activity_sleep")
```

**Hub location labels:**

```renpy
# story/hub_events.rpy

label hub_gym:
    scene bg_gym with dissolve
    "What do you want to do at the gym?"
    menu:
        "Work out (costs 15 energy)":
            jump activity_exercise
        "Talk to Monica" if npc_is_here("monica", "gym"):
            jump talk_to_monica_gym
        "Leave":
            jump hub_return

label hub_return:
    call check_player_state
    jump hub_main
```

---

## 5. NPC Scheduling System

NPCs appear at different locations based on time of day. Check availability before showing interaction options.

```renpy
# systems/npc_schedule.rpy

init python:

    # Schedule: {char_id: {period: loc_id or None}}
    NPC_SCHEDULE = {
        "sarah": {
            "morning":   "apartment",   # Sarah is at her apartment in the morning
            "afternoon": "shop",        # at the shop in the afternoon
            "evening":   "park",        # in the park in the evening
            "night":     None,          # not available at night
        },
        "monica": {
            "morning":   "gym",
            "afternoon": "gym",
            "evening":   "bar",
            "night":     None,
        },
        "diane": {
            "morning":   "work",
            "afternoon": "work",
            "evening":   "apartment",
            "night":     "apartment",
        },
    }

    def npc_current_location(char_id):
        """Returns the loc_id where this NPC currently is, or None."""
        schedule = NPC_SCHEDULE.get(char_id, {})
        period   = store.game_time.period if store.game_time else "morning"
        return schedule.get(period)

    def npc_is_here(char_id, loc_id):
        """True if this NPC is at the given location right now."""
        return npc_current_location(char_id) == loc_id

    def npcs_at_location(loc_id):
        """Returns list of char_ids present at a location right now."""
        return [cid for cid in NPC_SCHEDULE if npc_current_location(cid) == loc_id]
```

**Using in story:**

```renpy
label hub_park:
    scene bg_park with dissolve
    python:
        present = npcs_at_location("park")

    "You arrive at the park."

    if "sarah" in present:
        "You spot Sarah sitting on a bench."
        menu:
            "Talk to Sarah":
                jump talk_to_sarah_park
            "Leave her be":
                pass

    if not present:
        "The park is quiet today."

    menu:
        "Take a walk (restore mood +10)":
            jump activity_park_walk
        "Leave":
            jump hub_return
```

---

## 6. Content Gating Patterns

### Relationship gate

```renpy
# Only show scene option if relationship threshold met
label talk_to_sarah:
    if characters["sarah"].value >= 50:
        menu:
            "Invite her over":
                call label_check_scene_unlock("sarah", "sarah_scene_03")
    else:
        e "It's still a bit early for that..."
```

### Item gate

```renpy
label hub_shop:
    menu:
        "Buy wine ($30)" if player_survival.money >= 30:
            python:
                player_survival.modify("money", -30)
                player_inventory.add("item_wine")
                # Check if any scenes are now unlocked
                check_new_scenes_available("sarah")
            "You pick up a bottle of wine."
            jump hub_return
```

### Scene trigger helper

```renpy
# systems/character_system.rpy

init python:
    def label_check_scene_unlock(char_id, scene_id):
        """Call this label to try to play a scene. Shows requirements if unmet."""
        char  = store.characters.get(char_id)
        scene = next((s for s in char.scenes if s.scene_id == scene_id), None) if char else None
        if not scene:
            return
        if scene.is_seen() or scene.requirements_met(char):
            play_scene(scene_id)
        else:
            missing = scene.missing_requirements(char)
            renpy.notify("Not yet. You need: " + ", ".join(missing))
```

### Flag-based gate

```renpy
label talk_to_monica:
    if quest_manager.is_complete("quest_monica_02"):
        jump monica_path_advanced
    elif quest_manager.is_active("quest_monica_01"):
        jump monica_path_quest_active
    else:
        jump monica_path_intro
```

---

## 7. Gallery Screen

```renpy
# screens/gallery.rpy

screen gallery_screen():
    modal True
    zorder 50

    frame:
        xalign 0.5 yalign 0.5
        xsize 800 ysize 550
        padding (20, 20)

        vbox:
            spacing 16
            text "Gallery" size 26 bold True xalign 0.5

            # Character tabs
            python:
                char_list = list(store.characters.values())

            hbox:
                xalign 0.5
                spacing 12
                for char in char_list:
                    textbutton char.name:
                        action SetScreenVariable("selected_char", char.char_id)

            # Scene grid for selected character
            python:
                sel_id   = screen.selected_char if hasattr(screen, "selected_char") else (char_list[0].char_id if char_list else None)
                sel_char = store.characters.get(sel_id)

            if sel_char:
                hbox:
                    spacing 10
                    xalign 0.5
                    for scene in sel_char.scenes:
                        if scene.is_seen():
                            imagebutton:
                                idle  scene.thumbnail
                                xsize 160 ysize 120
                                action Function(play_scene, scene.scene_id)
                                tooltip scene.title
                        else:
                            frame:
                                xsize 160 ysize 120
                                background "#1a1a1a"
                                vbox:
                                    xalign 0.5 yalign 0.5
                                    text "???" xalign 0.5 size 22 color "#555"
                                    if sel_char.value < scene.rel_required:
                                        text "Rel: {}/{}".format(sel_char.value, scene.rel_required) size 11 color "#888" xalign 0.5

            textbutton "Close" action Return() xalign 0.5

    default selected_char = (list(store.characters.keys())[0] if store.characters else None)
```

---

## 8. Persistent Unlock Data

Gallery scenes and achievements should survive a New Game restart (so players don't lose progress on replay).

```renpy
# systems/flags.rpy

# Persistent — survives new game
define persistent.seen_scenes       = []   # list of scene_ids, never reset
define persistent.achievements      = []   # list of achievement_ids
define persistent.adult_unlocked    = False

# Per-save — reset on new game
default unlocked_scenes             = []   # mirror of persistent for current save
default adult_content_enabled       = False
```

**Syncing persistent data on save/load:**

```renpy
# script.rpy

label after_load:
    # Merge persistent seen scenes with save data
    python:
        for s in persistent.seen_scenes:
            if s not in store.unlocked_scenes:
                store.unlocked_scenes.append(s)
    return

# When a scene is marked seen, write to persistent too
init python:
    _base_play_scene = play_scene
    def play_scene(scene_id):
        if scene_id not in persistent.seen_scenes:
            persistent.seen_scenes.append(scene_id)
            renpy.save_persistent()
        _base_play_scene(scene_id)
```

---

## 9. Adult Content Flags

Provide an opt-in toggle for adult content. Show non-explicit versions when disabled.

```renpy
# systems/flags.rpy
default adult_content_enabled = False

# screens — settings or first-run screen
screen adult_content_toggle():
    frame:
        padding (20, 15)
        vbox:
            spacing 12
            text "Content Settings" size 20 bold True
            hbox:
                spacing 10
                text "Adult Content" size 16 yalign 0.5
                textbutton ("Enabled" if adult_content_enabled else "Disabled"):
                    action ToggleVariable("adult_content_enabled")
                    text_color ("#2ecc71" if adult_content_enabled else "#e74c3c")
            text "Adult scenes require this to be enabled." size 13 color "#888"
```

**In story — serving alternate versions:**

```renpy
label scene_sarah_03:
    scene bg_sarah_bedroom with dissolve
    play music "audio/music/intimate.ogg" fadein 1.0

    if adult_content_enabled:
        # Full scene
        show cg_sarah_03_full with dissolve
        "..."
        $ unlocked_scenes.append("sarah_scene_03_full")
    else:
        # Fade-to-black alternative
        scene black with fade
        "The night passes between you."
        scene bg_sarah_bedroom_morning with dissolve

    jump hub_main
```

---

## 10. Multi-Stage Scene Chains

For games where unlocking a character isn't a single scene but a **multi-step seduction** (gift → date → intimate encounter), use a `SceneSequence` pattern. Each stage has its own requirements, partial rewards, and triggers the next stage on completion.

```renpy
# systems/character_system.rpy

init python:

    class SceneStage:
        """A single stage in a multi-stage scene chain (e.g., gift step, date step)."""

        def __init__(self, stage_id, title, label,
                     rel_required=0, items_required=None, money_cost=0,
                     partial_rewards=None):
            self.stage_id        = stage_id
            self.title           = title
            self.label           = label
            self.rel_required    = rel_required
            self.items_required  = items_required or []
            self.money_cost      = money_cost     # paid on stage trigger
            self.partial_rewards = partial_rewards or []   # applied after stage completes

        def requirements_met(self, relationship):
            if relationship.value < self.rel_required:
                return False
            inv = store.player_inventory
            for item_id in self.items_required:
                if not inv or not inv.has(item_id):
                    return False
            if self.money_cost > 0 and store.player_survival.money < self.money_cost:
                return False
            return True


    class SceneSequence:
        """A multi-stage scene chain attached to a character. Linear progression."""

        def __init__(self, sequence_id, char_id, stages, final_unlock=None):
            self.sequence_id   = sequence_id
            self.char_id       = char_id
            self.stages        = stages          # list of SceneStage
            self.final_unlock  = final_unlock    # callable() applied when last stage clears
            self.current_idx   = 0
            self.completed     = False
            self.cancelled     = False

        def current_stage(self):
            if self.cancelled or self.completed:
                return None
            if self.current_idx >= len(self.stages):
                return None
            return self.stages[self.current_idx]

        def can_advance(self):
            stage = self.current_stage()
            if not stage:
                return False
            rel = store.characters.get(self.char_id)
            return rel and stage.requirements_met(rel)

        def trigger_current_stage(self):
            """Pay costs, call stage label, apply partial rewards, advance."""
            stage = self.current_stage()
            rel = store.characters.get(self.char_id)
            if not stage or not stage.requirements_met(rel):
                renpy.notify("Requirements not met yet.")
                return False

            # Pay money cost
            if stage.money_cost > 0:
                store.player_survival.modify("money", -stage.money_cost)
            # Consume required items
            for item_id in stage.items_required:
                store.player_inventory.remove(item_id, 1)

            # Run the stage scene
            renpy.call(stage.label)

            # Apply partial rewards
            for reward in stage.partial_rewards:
                reward.apply()

            # Mark scene as seen
            if stage.stage_id not in store.unlocked_scenes:
                store.unlocked_scenes.append(stage.stage_id)

            # Advance
            self.current_idx += 1
            if self.current_idx >= len(self.stages):
                self.completed = True
                if self.final_unlock:
                    self.final_unlock()
                renpy.notify("Sequence complete: {}".format(self.sequence_id))
            return True

        def cancel(self, penalty_callback=None):
            """Player abandons this sequence (penalty in relationship)."""
            self.cancelled = True
            if penalty_callback:
                penalty_callback()


    # Define sequences declaratively
    SCENE_SEQUENCES = {
        "seq_lorenzo_courtship": SceneSequence(
            "seq_lorenzo_courtship", "lorenzo",
            stages=[
                SceneStage("lorenzo_stage_01", "First Gift",
                           "scene_lorenzo_stage_01",
                           rel_required=10, items_required=["item_premium_perfume"],
                           partial_rewards=[StatReward("influence", 2)]),
                SceneStage("lorenzo_stage_02", "Dinner Date",
                           "scene_lorenzo_stage_02",
                           rel_required=25, money_cost=5000,
                           partial_rewards=[StatReward("influence", 3)]),
                SceneStage("lorenzo_stage_03", "Private Evening",
                           "scene_lorenzo_stage_03",
                           rel_required=40, money_cost=10000,
                           partial_rewards=[StatReward("influence", 10), MoneyReward(20000)]),
            ],
            final_unlock=lambda: setattr(store, "lorenzo_unlocked_as_ally", True),
        ),
    }
```

**Triggering from the hub or a UI button:**

```renpy
label seq_lorenzo_courtship_advance:
    python:
        seq = SCENE_SEQUENCES["seq_lorenzo_courtship"]
        if seq.can_advance():
            seq.trigger_current_stage()
        else:
            renpy.notify("Not ready for the next step.")
    jump hub_main
```

**Integration with quest system**: each sequence can be wrapped as a Quest (see `renpy-quests.md`) where each `SceneStage` corresponds to a `StagedQuest` stage with its own objectives.

---

## 11. Multi-Dimensional NPC Stats

Some games need more than a single relationship value per NPC. Examples:
- **Trust** vs **desire** (a character can be lustful but distrustful, or vice versa).
- **Suspicion** (does the NPC suspect the player is manipulating them?).
- **Power tier** (a fixed measure of social/political clout independent of relationship).

```renpy
# systems/character_system.rpy

init python:

    class NPCStats:
        """
        Multi-dimensional NPC state. Extends or replaces simple value-based relationship.
        Stats are 0-100 unless 'fixed' (set once at creation, never modified).
        """

        def __init__(self, npc_id, name,
                     trust=0, desire=0, suspicion=0,
                     power_tier=1, fixed_attrs=None):
            self.npc_id      = npc_id
            self.name        = name
            self.trust       = trust          # 0-100
            self.desire      = desire         # 0-100 (e.g., desire for spouse/protagonist)
            self.suspicion   = suspicion      # 0-100 (suspects player's intentions)
            self.power_tier  = power_tier     # 1-5 (fixed, never changes)
            self.fixed_attrs = fixed_attrs or {}  # arbitrary fixed metadata

        def _clamp(self, v):
            return max(0, min(100, v))

        def modify(self, stat, delta):
            """Modify a mutable stat. Power tier is read-only."""
            if stat == "power_tier":
                return  # fixed
            cur = getattr(self, stat, None)
            if cur is None:
                return
            setattr(self, stat, self._clamp(cur + delta))

        def is_exposed_risk(self, threshold=80):
            """High suspicion = NPC may expose / cut ties."""
            return self.suspicion >= threshold


    # Registry — define all NPCs with multi-dim stats
    NPC_STATS = {
        "lorenzo": NPCStats("lorenzo", "Lorenzo", trust=20, desire=30,
                            suspicion=0, power_tier=3),
        "romi":    NPCStats("romi",    "Romi",   trust=10, desire=50,
                            suspicion=0, power_tier=4),
    }


    def npc_modify(npc_id, stat, delta):
        npc = NPC_STATS.get(npc_id)
        if npc:
            npc.modify(stat, delta)
            check_npc_thresholds(npc_id)


    def check_npc_thresholds(npc_id):
        """Trigger threshold events (suspicion peak, trust collapse, etc.)."""
        npc = NPC_STATS.get(npc_id)
        if not npc:
            return
        if npc.suspicion >= 100:
            renpy.call_in_new_context("event_{}_exposes_player".format(npc_id))
        elif npc.trust <= 0 and not getattr(store, "npc_lost_{}_seen".format(npc_id), False):
            setattr(store, "npc_lost_{}_seen".format(npc_id), True)
            renpy.notify("{} has lost trust in you.".format(npc.name))
```

This pattern is **compatible** with `CharacterRelationship` from Section 2 — use one or the other depending on game complexity. For Politicuck-style games, use `NPCStats`. For straightforward dating sims, `CharacterRelationship` is enough.

---

## 12. Opaque NPC Stats — Qualitative Indicators

In some games, exposing raw numerical relationship values to the player breaks immersion. Show **qualitative labels** ("Likes you", "Suspicious of you") instead of "Trust: 42/100".

```renpy
# systems/character_system.rpy

init python:

    TRUST_LABELS = [
        (90, "Inseparable"),
        (75, "Close friend"),
        (55, "Trusts you"),
        (35, "Comfortable around you"),
        (20, "Acquaintance"),
        (5,  "Indifferent"),
        (0,  "Distant"),
    ]

    SUSPICION_LABELS = [
        (90, "Watching you closely — they know"),
        (70, "Visibly suspicious of you"),
        (45, "Asking pointed questions"),
        (20, "Slightly curious about your motives"),
        (0,  "Trusts your story"),
    ]

    DESIRE_LABELS = [
        (90, "Openly enamored"),
        (70, "Watches them hungrily"),
        (50, "Attracted, but tries to hide it"),
        (25, "Mild interest"),
        (0,  "No spark"),
    ]


    def qualitative_label(value, table):
        """Return the first label whose threshold is <= value."""
        for threshold, label in table:
            if value >= threshold:
                return label
        return table[-1][1]


    def npc_status_summary(npc_id):
        """Return a dict of qualitative labels for UI display."""
        npc = NPC_STATS.get(npc_id)
        if not npc:
            return {}
        return {
            "trust":     qualitative_label(npc.trust, TRUST_LABELS),
            "suspicion": qualitative_label(npc.suspicion, SUSPICION_LABELS),
            "desire":    qualitative_label(npc.desire, DESIRE_LABELS),
            "power":     ["", "Minor", "Local", "Mid-tier", "Major", "Elite"][npc.power_tier],
        }
```

**Usage in screens** — never show numbers, only labels:

```renpy
screen npc_contact_card(npc_id):
    python:
        summary = npc_status_summary(npc_id)
        npc = NPC_STATS[npc_id]
    frame:
        padding (12, 10)
        vbox:
            spacing 4
            text npc.name size 18 bold True
            text "[summary['trust']]"     size 14 color "#aaaaaa"
            text "[summary['suspicion']]" size 14 color "#e74c3c"
            text "[summary['desire']]"    size 14 color "#9b59b6"
            text "Influence: [summary['power']]" size 13 color "#888"
```

**Debug mode override**: when a debug toggle is set, display the raw numbers too (see your project's debug skill).

---

## 13. City-Wide Reputation / Scandal Meter

Some games need a **global reputation / rumor stat** that tracks how much the city/society "knows" about the player's secret behavior. Distinct from per-NPC suspicion: this is public, affects all NPCs, and the press.

```renpy
# systems/reputation_system.rpy

init python:

    class CityRumorMeter:
        """
        Tracks city-wide gossip about the player.
        0  = clean, no one talks.
        100 = front-page scandal, irreversible damage.
        """

        TIERS = [
            (90, "explosive",  "Imminent front-page scandal"),
            (60, "loud",       "Mid-tier media talking"),
            (30, "rumors",     "Bar gossip and forum chatter"),
            (0,  "clean",      "No one's talking"),
        ]

        def __init__(self, value=0, weekly_decay=0, weekly_growth=0):
            self.value          = value
            self.weekly_decay   = weekly_decay   # baseline decay (e.g., from influence/contacts)
            self.weekly_growth  = weekly_growth  # baseline growth (e.g., already-leaked rumors)

        def _clamp(self, v):
            return max(0, min(100, v))

        def modify(self, delta):
            self.value = self._clamp(self.value + delta)

        def tick_weekly(self):
            """Call once per in-game week. Applies decay/growth."""
            net = self.weekly_growth - self.weekly_decay
            self.modify(net)

        def current_tier(self):
            for threshold, tier_id, desc in self.TIERS:
                if self.value >= threshold:
                    return (tier_id, desc)
            return self.TIERS[-1][1:]

        def public_image_modifier(self):
            """Return a per-slot or per-day Image stat modifier based on current tier."""
            tier_id, _ = self.current_tier()
            return {
                "explosive": -3,
                "loud":      -1,
                "rumors":    -0.5,
                "clean":     0,
            }.get(tier_id, 0)


default city_rumor = None

init python:
    def init_rumor_meter():
        store.city_rumor = CityRumorMeter()
```

**Hooks to raise/lower rumor**:

```renpy
init python:

    def gossip_raise(amount, reason=""):
        """Public action: a witness, a paparazzi, etc."""
        store.city_rumor.modify(amount)
        if reason:
            renpy.notify("Gossip rising: {}".format(reason))

    def gossip_bribe(cost, reduction):
        """Pay to suppress rumors. Returns True if affordable."""
        if store.player_survival.money < cost:
            return False
        store.player_survival.modify("money", -cost)
        store.city_rumor.modify(-reduction)
        renpy.notify("Paid ${} to suppress {} gossip points.".format(cost, reduction))
        return True

    def gossip_apply_weekly():
        """Call from on_new_day when weekday rolls over."""
        if store.game_time.day_name == "Monday":   # weekly tick on Monday
            store.city_rumor.tick_weekly()
```

**Integration with public stats** (e.g., Image / fame):

```renpy
def end_of_slot_hook():
    """Apply per-slot image decay based on current rumor tier."""
    mod = store.city_rumor.public_image_modifier()
    if mod != 0:
        store.player_survival.modify("image", mod)
```

---

## 14. Mini-Game Interruptions During Scenes

For dramatic moments mid-scene: a paparazzi click QTE, a snap reaction prompt, a quick-decision dialog. The player's response affects the scene outcome.

```renpy
# systems/scene_interrupts.rpy

init python:

    class ClickInterrupt:
        """A click-the-target QTE that can interrupt a scene."""

        def __init__(self, target_image, time_limit=2.0, label_on_hit=None,
                     label_on_miss=None):
            self.target_image = target_image
            self.time_limit   = time_limit
            self.label_on_hit = label_on_hit
            self.label_on_miss= label_on_miss
            self.over         = False
            self.hit          = False
            self.time_left    = time_limit

        def click(self):
            if self.over:
                return
            self.over = True
            self.hit  = True

        def tick(self, dt):
            if self.over:
                return
            self.time_left -= dt
            if self.time_left <= 0:
                self.over = True


screen click_interrupt(interrupt):
    modal True
    zorder 70

    imagebutton:
        idle interrupt.target_image
        action [Function(interrupt.click), Return(True)]
        xalign 0.85 yalign 0.4

    bar value AnimatedValue(interrupt.time_left, interrupt.time_limit, 0.05):
        xsize 240 ysize 8 xalign 0.5 yalign 0.92

    timer 0.05 repeat True action [
        Function(interrupt.tick, 0.05),
        If(interrupt.over, Return(interrupt.hit), renpy.restart_interaction),
    ]
```

**Inside a scene label:**

```renpy
label scene_lorenzo_stage_03:
    scene bg_lorenzo_quinta with dissolve
    "You and Vicky enter the secluded quinta..."

    # Mid-scene paparazzi check
    if renpy.random.random() < 0.25:    # 25% chance based on current rumor tier
        python:
            store._paparazzi = ClickInterrupt(
                target_image="images/ui/paparazzi.png",
                time_limit=2.0,
            )
        call screen click_interrupt(_paparazzi)
        $ hit = _return
        if hit:
            "You spot a flash in the bushes — you wave them off just in time."
        else:
            "A flash. Someone got the shot."
            $ gossip_raise(20, "paparazzi photo")
            $ npc_modify("lorenzo", "suspicion", +5)

    # Continue scene...
    "..."
    return
```

This pattern is **reusable** for any kind of click-the-thing interrupt: dodge a thrown object, catch a falling glass, snap a quick photo, etc.

---

## 15. NPC Autonomous Events (Off-Screen Cameos)

Sometimes the spouse (or any NPC) has a story arc happening **off-screen** — without the player present. This plants narrative seeds for later reveals. These events don't change stats; they only set silent flags consumed by future scenes.

```renpy
# systems/npc_autonomous.rpy

init python:

    # Silent flags accumulated per NPC for future story payoff
    default vicky_hints = []
    default spouse_off_screen_flags = {}

    AUTONOMOUS_HINTS = {
        # hint_id: (probability_per_eligible_day, label_to_play, prerequisites_callable)
        "vicky_hint_perfume": (0.15, "vicky_hint_perfume_event",
                                lambda: store.game_time.day > 7),
        "vicky_hint_phonecall": (0.12, "vicky_hint_phonecall_event",
                                  lambda: store.game_time.day > 15),
        "vicky_hint_chargeback": (0.10, "vicky_hint_chargeback_event",
                                   lambda: store.game_time.day > 30),
    }


    def try_trigger_autonomous_hint():
        """Call on hub entry or new day. May trigger one off-screen hint event."""
        eligible = []
        for hint_id, (prob, label, prereq) in AUTONOMOUS_HINTS.items():
            if hint_id in store.vicky_hints:
                continue
            if not prereq():
                continue
            if renpy.random.random() < prob:
                eligible.append((hint_id, label))

        if eligible:
            hint_id, label = renpy.random.choice(eligible)
            store.vicky_hints.append(hint_id)
            renpy.call_in_new_context(label)
```

**Defining a hint event label:**

```renpy
label vicky_hint_perfume_event:
    scene bg_apartment_evening with dissolve
    "Vicky takes a long moment to answer the door when you arrive."
    "When she opens, her hair is slightly disheveled."
    "There's a faint scent of cologne — not yours — in the air."
    "She smiles brightly. \"Hi amor! I was just in the shower.\""
    # No stat change. No choice. Just a silent flag now in vicky_hints.
    return
```

**Wiring into the day loop:**

```renpy
label hub_main:
    call try_trigger_autonomous_hint  # may play a hint event before hub
    # ... rest of hub
```

Hints can be queried later for narrative branches (e.g., a final-act reveal that lists every hint the player saw).

---

## 16. Achievement System with Bonus Content

Cross-save persistent achievements that unlock bonus content (extra CGs in the gallery, end-credits notes, etc.).

```renpy
# systems/achievement_system.rpy

init python:

    class Achievement:
        def __init__(self, ach_id, name, description, bonus_cg=None, hidden=False):
            self.ach_id      = ach_id
            self.name        = name
            self.description = description
            self.bonus_cg    = bonus_cg     # image path, optional
            self.hidden      = hidden       # True = secret achievement


    ACHIEVEMENTS = {
        "first_meet":    Achievement("first_meet",    "First Contact",
                                     "Meet the first major NPC.",
                                     bonus_cg="images/cg/achievement_first_meet.png"),
        "sequence_clear":Achievement("sequence_clear","Master Negotiator",
                                     "Complete a full SceneSequence.",
                                     bonus_cg="images/cg/achievement_sequence.png"),
        "rumor_survivor":Achievement("rumor_survivor","Damage Control",
                                     "Bring city rumor from 90+ back below 50.",
                                     bonus_cg="images/cg/achievement_survivor.png"),
        "secret_path":   Achievement("secret_path",   "?????",
                                     "Hidden achievement.",
                                     hidden=True,
                                     bonus_cg="images/cg/achievement_secret.png"),
    }


    def grant_achievement(ach_id):
        """Award an achievement once. Persists across saves."""
        if not hasattr(persistent, "achievements") or persistent.achievements is None:
            persistent.achievements = {}
        if ach_id in persistent.achievements:
            return False
        ach = ACHIEVEMENTS.get(ach_id)
        if not ach:
            return False
        persistent.achievements[ach_id] = renpy.get_game_runtime()
        renpy.save_persistent()
        renpy.notify("Achievement unlocked: {}".format(ach.name))
        return True


    def has_achievement(ach_id):
        if not hasattr(persistent, "achievements") or persistent.achievements is None:
            return False
        return ach_id in persistent.achievements


    def all_achievements_status():
        """Returns list of (Achievement, unlocked_bool) for the gallery / menu."""
        out = []
        for ach in ACHIEVEMENTS.values():
            out.append((ach, has_achievement(ach.ach_id)))
        return out
```

**Achievements screen**:

```renpy
screen achievements_screen():
    modal True
    zorder 50
    frame:
        xalign 0.5 yalign 0.5
        xsize 700 ysize 500
        padding (20, 16)
        vbox:
            spacing 12
            text "Achievements" size 24 bold True xalign 0.5

            viewport:
                scrollbars "vertical"
                mousewheel True
                ysize 380
                vbox:
                    spacing 8
                    for ach, unlocked in all_achievements_status():
                        hbox:
                            spacing 12
                            xfill True
                            if unlocked and ach.bonus_cg:
                                imagebutton:
                                    idle ach.bonus_cg
                                    xsize 80 ysize 60
                                    action ShowMenu("cg_viewer", ach.bonus_cg)
                            else:
                                frame:
                                    xsize 80 ysize 60
                                    background "#1a1a1a"
                                    text "?" xalign 0.5 yalign 0.5 size 24 color "#444"
                            vbox:
                                if ach.hidden and not unlocked:
                                    text "???" size 16 color "#666"
                                    text "Hidden achievement" size 13 color "#444"
                                else:
                                    text ach.name size 16 bold True
                                    text ach.description size 13 color "#bbb"

            textbutton "Close" action Return() xalign 0.5
```

**Trigger achievements from anywhere**:

```renpy
label scene_lorenzo_stage_01:
    # ... scene content ...
    $ grant_achievement("first_meet")
    return
```

---

## 17. Adult Content Distribution Policy

When shipping adult content, decisions about toggles, prohibited content, and disclaimers are not optional — they're requirements for safe distribution.

### Toggle vs no-toggle

Two valid approaches:

**A) With toggle** (default in Section 9): provides a `adult_content_enabled` flag and serves fade-to-black versions when disabled. Good for: dual SFW/NSFW distribution, broader platform reach (some adult platforms gate by toggle), accessibility for sensitive players.

**B) Without toggle** (all-explicit): the game is adult-only, period. Simpler asset pipeline (no duplicate scenes), clearer marketing. Good for: itch.io adult-only distribution, smaller studios who don't want SFW maintenance burden. Choose this when your target audience and platforms are 100% adult.

**Decision matrix**:

| Scenario | Recommendation |
|----------|----------------|
| Targeting Steam (variable adult policy) | A) with toggle |
| Targeting itch.io adult-only | B) without toggle is fine |
| Targeting Patreon for early access | A) with toggle for tier flexibility |
| Mod-friendly project (community SFW patch?) | A) with toggle (lower friction) |
| Solo dev, small scope | B) without toggle (less maintenance) |

### Prohibited content (TOS-safe)

Across **all platforms** (itch.io, Steam, Patreon, F95Zone, SubscribeStar, app stores), the following are universally prohibited and will result in delisting, bans, and possible legal action:

- **Minors in sexual content** — anyone under 18, even fictional, even drawn. Zero tolerance.
- **Sexual violence framed as positive** (rape glamorized as desirable). Non-con as horror/drama is sometimes OK with appropriate framing; non-con as titillation is not.
- **Bestiality** with real animals or photorealistic animals.
- **Real-person sexual content** without explicit permission.
- **Incest involving minors**. Adult step-relationships are generally tolerated on adult platforms but check each platform's TOS.

Document a list of `prohibited_content_check.md` in your repo that auditors can reference.

### Mandatory +18 splash disclaimer

Show on first launch and require explicit confirmation. Persistent flag so they only see it once.

```renpy
# screens/splash_disclaimer.rpy

default persistent.adult_confirmed = False

screen adult_disclaimer():
    modal True
    zorder 999

    add "#000000"

    frame:
        xalign 0.5 yalign 0.5
        xsize 700
        padding (30, 28)
        background "#1a1a1a"

        vbox:
            spacing 18
            xalign 0.5

            text "ADULT CONTENT WARNING" xalign 0.5 size 28 bold True color "#e74c3c"
            text "This game contains sexually explicit content, strong language, and themes intended for adults only.":
                xalign 0.5 size 18 color "#dddddd" text_align 0.5
            text "All characters are 18+ and fictional. Any resemblance to real persons is for parody/satire only.":
                xalign 0.5 size 15 color "#aaaaaa" text_align 0.5
            text "By continuing, you confirm you are at least 18 years old.":
                xalign 0.5 size 16 color "#ffffff" text_align 0.5

            hbox:
                xalign 0.5 spacing 30
                textbutton "I am 18+, continue":
                    action [SetField(persistent, "adult_confirmed", True), Return()]
                    text_color "#2ecc71"
                textbutton "Exit":
                    action Quit(confirm=False)
                    text_color "#e74c3c"


# In script.rpy splash flow:
label splash:
    if not persistent.adult_confirmed:
        call screen adult_disclaimer
    return
```

---

## 18. Full Integration Checklist

Before releasing a sandbox VN built on this framework:

**Systems init (called at game start):**
- [ ] `init_survival_stats()` called before hub
- [ ] `init_time()` called at game start
- [ ] `init_characters()` called at game start
- [ ] `init_inventory()` called at game start
- [ ] `init_quest_manager()` called at game start

**Per-action loop:**
- [ ] `spend_action()` called after every activity
- [ ] `check_player_state` called at hub entry
- [ ] `quest_manager.tick_unlock_conditions()` called on hub entry
- [ ] `quest_manager.check_all_stat_objectives()` called after any stat change

**Content gating:**
- [ ] All scenes have explicit `rel_required`, `items_required`, `flags_required`
- [ ] Adult scenes have `is_adult=True` and a fade-to-black alternative
- [ ] `adult_content_enabled` toggle accessible from settings
- [ ] `unlocked_scenes` mirrored to `persistent.seen_scenes`

**NPC scheduling:**
- [ ] Every named NPC has an entry in `NPC_SCHEDULE`
- [ ] Hub location menus use `npc_is_here()` before showing talk options
- [ ] "NPC not available" message shown when they're absent

**Quest chains:**
- [ ] Every character has a complete quest chain (`quest_<char>_01` → ... → `quest_<char>_final`)
- [ ] Final quest in each chain unlocks the character's last scene
- [ ] `apply_quest_rewards()` wired into `Quest.complete()`

**Gallery:**
- [ ] All scenes have a thumbnail image
- [ ] Locked scenes show "???" placeholder and hint (relationship requirement)
- [ ] Gallery accessible from hub quick menu without consuming an action
