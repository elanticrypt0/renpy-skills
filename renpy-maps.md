# Ren'Py Maps & World Navigation Reference

Use this when the project has a world map, location selection screen, or area-based navigation.

---

## Table of Contents
1. [Map Architecture](#1-map-architecture)
2. [Location Data](#2-location-data)
3. [Map Screen](#3-map-screen)
4. [Travel Logic](#4-travel-logic)
5. [Mini-Map / Area HUD](#5-mini-map--area-hud)
6. [Flat Sandbox Variant (No Connections)](#6-flat-sandbox-variant-no-connections)
7. [Animated Hotspot Indicators](#7-animated-hotspot-indicators)
8. [Sub-Zones Within a Location](#8-sub-zones-within-a-location)
9. [Locked Hotspot with Reason Tooltip](#9-locked-hotspot-with-reason-tooltip)
10. [Travel Cost (Optional)](#10-travel-cost-optional)

---

## 1. Map Architecture

```
systems/
└── map_system.rpy       # Location data, travel logic, unlock tracking

screens/
└── map.rpy              # World map screen

images/
└── maps/
    ├── world_map.png    # Base map image
    ├── icon_town.png    # Location marker icons
    ├── icon_dungeon.png
    └── icon_locked.png
```

**Design principle:** Locations are data, not labels. Separate the data (what places exist, what they require) from the display (the map screen) and from the story (labels that get called when you arrive).

---

## 2. Location Data

```renpy
# systems/map_system.rpy

init python:

    class Location:
        """Represents a single navigable location on the world map."""

        def __init__(self, loc_id, name, description, map_pos,
                     icon="icon_town", entry_label=None,
                     unlock_condition=None, connections=None):
            self.loc_id            = loc_id
            self.name              = name
            self.description       = description
            self.map_pos           = map_pos          # (x, y) as fractions of map image (0.0–1.0)
            self.icon              = icon
            self.entry_label       = entry_label      # label to call on arrival
            self.unlock_condition  = unlock_condition  # callable() → bool, or None (always unlocked)
            self.connections       = connections or [] # list of loc_ids reachable from here

        def is_unlocked(self):
            if self.unlock_condition is None:
                return True
            return self.unlock_condition()

        def is_visited(self):
            return store.visited_locations.get(self.loc_id, False)


    # ── Location Registry ────────────────────────────────────────────
    LOCATIONS = {
        "town_start": Location(
            "town_start", "Ashford Village",
            "A quiet starting village.",
            map_pos=(0.18, 0.65),
            icon="icon_town",
            entry_label="arrive_town_start",
            connections=["forest_path", "old_road"],
        ),
        "forest_path": Location(
            "forest_path", "Whispering Forest",
            "Dense trees hide ancient secrets.",
            map_pos=(0.35, 0.45),
            icon="icon_dungeon",
            entry_label="arrive_forest_path",
            unlock_condition=lambda: store.met_elara,
            connections=["town_start", "forest_ruins"],
        ),
        "forest_ruins": Location(
            "forest_ruins", "Ancient Ruins",
            "Collapsed towers from a forgotten age.",
            map_pos=(0.50, 0.30),
            icon="icon_dungeon",
            entry_label="arrive_forest_ruins",
            unlock_condition=lambda: store.chapter_01_complete,
            connections=["forest_path"],
        ),
        "old_road": Location(
            "old_road", "Old Kingdom Road",
            "A long road leading to the capital.",
            map_pos=(0.25, 0.80),
            icon="icon_town",
            entry_label="arrive_old_road",
            connections=["town_start"],
        ),
    }

    def get_available_locations(from_loc_id=None):
        """Return unlocked locations reachable from current position."""
        if from_loc_id is None:
            return [loc for loc in LOCATIONS.values() if loc.is_unlocked()]
        current = LOCATIONS.get(from_loc_id)
        if not current:
            return []
        return [LOCATIONS[cid] for cid in current.connections
                if cid in LOCATIONS and LOCATIONS[cid].is_unlocked()]


# ── State ─────────────────────────────────────────────────────────────
default current_location = "town_start"
default visited_locations = {}    # {loc_id: True}
```

---

## 3. Map Screen

```renpy
# screens/map.rpy

screen world_map_screen():
    modal True
    zorder 50

    add "images/maps/world_map.png" xalign 0.5 yalign 0.5

    # Location markers
    python:
        available = get_available_locations(store.current_location)
        available_ids = {loc.loc_id for loc in available}

    for loc_id, loc in LOCATIONS.items():
        if loc.is_unlocked():
            $ xpos = int(loc.map_pos[0] * 1280)   # adjust to your map image width
            $ ypos = int(loc.map_pos[1] * 720)

            imagebutton:
                xpos xpos ypos ypos anchor (0.5, 0.5)
                idle  "images/maps/[loc.icon].png"
                hover "images/maps/[loc.icon]_hover.png" if loc_id in available_ids else "images/maps/[loc.icon].png"
                insensitive "images/maps/[loc.icon]_dim.png"
                sensitive (loc_id in available_ids and loc_id != store.current_location)
                action [
                    SetVariable("_map_travel_target", loc_id),
                    Return("travel"),
                ]
                tooltip loc.name

            # Location name label
            text loc.name:
                xpos xpos ypos (ypos + 28) anchor (0.5, 0.0)
                size 14 color "#ffffff" outlines [(1, "#000000", 0, 0)]

    # Current location indicator
    python:
        cur = LOCATIONS.get(store.current_location)
    if cur:
        $ cx = int(cur.map_pos[0] * 1280)
        $ cy = int(cur.map_pos[1] * 720)
        image "images/maps/icon_current.png" xpos cx ypos cy anchor (0.5, 0.5)

    # Close button
    textbutton "Close Map":
        action Return("close")
        xalign 0.98 yalign 0.02

    # Tooltip display
    if GetTooltip():
        frame:
            xalign 0.5 yalign 0.95
            text GetTooltip() size 18
```

---

## 4. Travel Logic

```renpy
# systems/map_system.rpy — travel handler

label open_world_map:
    call screen world_map_screen
    $ map_result = _return

    if map_result == "travel" and _map_travel_target:
        jump travel_to_location

    return  # "close" or no selection

label travel_to_location:
    python:
        target_id = store._map_travel_target
        target_loc = LOCATIONS.get(target_id)
        store._map_travel_target = None

    # Mark as visited
    $ visited_locations[target_id] = True
    $ current_location = target_id

    # Optional: travel narration
    "[target_loc.name]..."

    # Jump to entry label if defined
    if target_loc.entry_label:
        call expression target_loc.entry_label
    
    return
```

**In story files, open the map:**

```renpy
label town_square_hub:
    "Where would you like to go?"
    menu:
        "Open World Map":
            call open_world_map
        "Stay in Town":
            jump town_square_events
```

Or show as a screen button in HUD:

```renpy
screen town_hud():
    imagebutton:
        idle "images/ui/map_icon.png"
        action Jump("open_world_map")
        xalign 0.98 yalign 0.98
```

---

## 5. Mini-Map / Area HUD

For games with room-based exploration within a location:

```renpy
# systems/map_system.rpy

init python:

    class Room:
        """A single navigable room within a location."""

        def __init__(self, room_id, name, exits=None):
            self.room_id = room_id
            self.name    = name
            self.exits   = exits or {}  # {"north": "room_id", "east": "room_id"}

    DUNGEON_ROOMS = {
        "entrance":  Room("entrance",  "Dungeon Entrance",  {"north": "hall_a"}),
        "hall_a":    Room("hall_a",    "Dark Corridor",     {"south": "entrance", "east": "chamber_b", "north": "boss_room"}),
        "chamber_b": Room("chamber_b", "Side Chamber",      {"west": "hall_a"}),
        "boss_room": Room("boss_room", "Boss Chamber",      {"south": "hall_a"}),
    }


default current_room = "entrance"

screen area_navigation():
    zorder 10
    frame:
        xalign 0.5 yalign 0.98
        hbox:
            spacing 12
            python:
                exits = DUNGEON_ROOMS.get(store.current_room, Room("", "")).exits
            for direction, target_id in exits.items():
                textbutton direction.capitalize():
                    action [
                        SetVariable("current_room", target_id),
                        Jump("enter_room"),
                    ]
```

---

## 6. Flat Sandbox Variant (No Connections)

For sandbox / life-sim games where every unlocked location should be **reachable from anywhere** (no travel routing). Skip the `connections` field entirely and replace `get_available_locations()`:

```renpy
init python:

    def get_available_locations_flat():
        """Sandbox variant: all unlocked locations are reachable, regardless of position."""
        return [loc for loc in LOCATIONS.values() if loc.is_unlocked()]
```

When defining locations, omit `connections=[...]` or set to `None`. The map screen then shows every unlocked hotspot as clickable. Use the flat variant when:
- The game is set in a city / single area where travel is implicit.
- You want lower friction in slot-based time systems (don't waste a turn on travel between two rooms).
- The narrative doesn't depend on adjacency or path-finding.

---

## 7. Animated Hotspot Indicators

Make hotspots visually communicate "something is happening here" — pending NPC, active quest, special event.

```renpy
# screens/map.rpy — enhanced markers

init python:

    def hotspot_indicator(loc_id):
        """
        Return an indicator string for visual state of a hotspot.
        - "event"  : a quest objective points here, or NPC waiting
        - "npc"    : a known NPC is currently at this location
        - "new"    : location just unlocked, not yet visited
        - "normal" : nothing special
        """
        # Quest-driven indicator
        for q in store.quest_manager.active_quests():
            for obj in getattr(q, "objectives", []):
                if isinstance(obj, LocationObjective) and obj.location_id == loc_id and not obj.is_complete():
                    return "event"
        # NPC presence (cross-skill with renpy-adult-vn npc_schedule)
        try:
            if npcs_at_location(loc_id):
                return "npc"
        except NameError:
            pass
        # New unlock
        if not store.visited_locations.get(loc_id, False):
            return "new"
        return "normal"
```

```renpy
transform hotspot_glow:
    ease 0.6 alpha 0.6
    ease 0.6 alpha 1.0
    repeat


# Inside world_map_screen, augment each marker:
for loc_id, loc in LOCATIONS.items():
    if loc.is_unlocked():
        $ xpos = int(loc.map_pos[0] * 1280)
        $ ypos = int(loc.map_pos[1] * 720)
        $ ind = hotspot_indicator(loc_id)

        imagebutton:
            xpos xpos ypos ypos anchor (0.5, 0.5)
            idle  "images/maps/[loc.icon].png"
            hover "images/maps/[loc.icon]_hover.png"
            action [SetVariable("_map_travel_target", loc_id), Return("travel")]

            # Overlay glow for events / NPCs
            if ind in ("event", "npc"):
                at hotspot_glow

            # Badge for "new"
            if ind == "new":
                add "images/maps/badge_new.png" xpos 16 ypos -10
```

---

## 8. Sub-Zones Within a Location

Some locations are large enough that they need an internal sub-menu — for example a casino with separate "blackjack room", "bar", "bathroom", "private salon". Different from the dungeon Room pattern (Section 5): sub-zones are usually a flat menu, not a navigation grid.

```renpy
init python:

    class SubZone:
        """A sub-area inside a location. Selectable from a sub-menu."""

        def __init__(self, zone_id, name, entry_label,
                     unlock_condition=None, npc_present=None):
            self.zone_id          = zone_id
            self.name             = name
            self.entry_label      = entry_label
            self.unlock_condition = unlock_condition
            self.npc_present      = npc_present   # callable() → npc_id or None


    # Attach sub-zones to a Location by extending the data
    Location.sub_zones = None   # default: no sub-zones


    LOCATIONS["casino_premium"].sub_zones = [
        SubZone("vip_blackjack",  "VIP Blackjack",
                "casino_premium_vip_blackjack",
                unlock_condition=lambda: store.influence >= 50),
        SubZone("private_salon",  "Private Salon",
                "casino_premium_private_salon"),
        SubZone("bar",            "Bar",
                "casino_premium_bar"),
        SubZone("bathroom",       "Bathroom",
                "casino_premium_bathroom"),
    ]
```

**Sub-zone selector screen** — shown when entering a location with `sub_zones`:

```renpy
screen sub_zone_picker(location):
    modal True
    zorder 30
    frame:
        xalign 0.5 yalign 0.5
        padding (24, 20)
        vbox:
            spacing 12
            text "[location.name]" size 22 bold True xalign 0.5
            text "Where do you want to go?" size 14 color "#bbb" xalign 0.5
            for zone in location.sub_zones:
                $ unlocked = zone.unlock_condition is None or zone.unlock_condition()
                hbox:
                    spacing 8
                    textbutton "[zone.name]":
                        action [Return("zone"), SetVariable("_sub_zone_target", zone.entry_label)]
                        sensitive unlocked
                    if not unlocked:
                        text "(locked)" size 12 color "#888"
            textbutton "Leave" action Return("leave") xalign 0.5
```

**In travel_to_location**, hand off to sub-zone picker when applicable:

```renpy
label travel_to_location:
    python:
        target_id  = store._map_travel_target
        target_loc = LOCATIONS.get(target_id)
        store._map_travel_target = None
    $ visited_locations[target_id] = True
    $ current_location = target_id

    if target_loc.sub_zones:
        call screen sub_zone_picker(target_loc)
        if _return == "zone":
            call expression store._sub_zone_target
        return
    elif target_loc.entry_label:
        call expression target_loc.entry_label
    return
```

---

## 9. Locked Hotspot with Reason Tooltip

When a hotspot is locked, show *why* on hover instead of just an opaque dim icon.

```renpy
init python:

    def location_lock_reason(loc):
        """Return a human-readable string explaining why this location is locked."""
        if loc.unlock_condition is None or loc.unlock_condition():
            return None
        # Custom reason factory per location
        return getattr(loc, "lock_reason", lambda: "Requirements not met.")()


    # When defining locations, attach a `lock_reason` callable
    LOCATIONS["congress"].lock_reason = lambda: "Requires Influence ≥ 25"
    LOCATIONS["quinta_nordelta"].lock_reason = lambda: "Requires an invitation."
    LOCATIONS["high_court"].lock_reason = lambda: "Requires Influence ≥ 60 and judicial connections."
```

```renpy
# In world_map_screen, surface the reason as tooltip on locked markers
for loc_id, loc in LOCATIONS.items():
    $ unlocked = loc.is_unlocked()
    $ reason   = None if unlocked else location_lock_reason(loc)
    imagebutton:
        idle "images/maps/[loc.icon].png" if unlocked else "images/maps/[loc.icon]_dim.png"
        sensitive unlocked
        tooltip (loc.name if unlocked else "{} — {}".format(loc.name, reason))
        action [SetVariable("_map_travel_target", loc_id), Return("travel")]
```

---

## 10. Travel Cost (Optional)

Some games charge slots / energy / money for traveling. Disabled by default — opt-in by configuring the cost handler.

```renpy
init python:

    # Default: free travel
    default travel_cost_config = {
        "slot_cost":   0,    # how many time slots a travel consumes
        "money_cost":  0,    # cash cost
        "energy_cost": 0,    # energy cost
    }


    def apply_travel_cost():
        """Charge the configured travel cost. Returns True if affordable."""
        cfg = store.travel_cost_config
        if store.player_survival.money < cfg["money_cost"]:
            renpy.notify("Not enough money to travel.")
            return False
        if store.player_survival.energy < cfg["energy_cost"]:
            renpy.notify("Too tired to travel.")
            return False
        store.player_survival.modify("money",  -cfg["money_cost"])
        store.player_survival.modify("energy", -cfg["energy_cost"])
        for _ in range(cfg["slot_cost"]):
            spend_action(1)
        return True
```

```renpy
# Hook into travel_to_location:
label travel_to_location:
    if not apply_travel_cost():
        return   # cancelled, can't afford
    # ... existing travel logic
```

For Politicuck-style sandbox: keep all costs at 0 by default; only meaningful **actions** consume slots. Document this clearly in your game's wiki / tutorial so players understand the time economy.

**Cross-references**:
- NPC presence by period/day: `renpy-adult-vn.md` § NPC Scheduling.
- Day-of-week gating of locations: `renpy-survival.md` § Day-of-Week Gating.