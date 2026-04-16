# Ren'Py Maps & World Navigation Reference

Use this when the project has a world map, location selection screen, or area-based navigation.

---

## Table of Contents
1. [Map Architecture](#1-map-architecture)
2. [Location Data](#2-location-data)
3. [Map Screen](#3-map-screen)
4. [Travel Logic](#4-travel-logic)
5. [Mini-Map / Area HUD](#5-mini-map--area-hud)

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