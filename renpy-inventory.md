# Ren'Py Inventory System Reference

Use this when the project has an inventory of items the player can collect, use, equip, gift, or carry as quest/key items. This is the canonical reference for the Item / Inventory pattern; other skills (RPG, adult VN, quests, shops, screens) reference back here for the full picture and define only what they need locally.

---

## Table of Contents
1. [Architecture](#1-architecture)
2. [Item Base Class](#2-item-base-class)
3. [Item Subclasses (5 Standard Types)](#3-item-subclasses-5-standard-types)
4. [ITEM_CATALOGUE Pattern](#4-item_catalogue-pattern)
5. [Inventory Class](#5-inventory-class)
6. [Persistence & Save Integration](#6-persistence--save-integration)
7. [Equipment Slots & Equipping](#7-equipment-slots--equipping)
8. [Gifting Pattern (Give to NPC)](#8-gifting-pattern-give-to-npc)
9. [Quest Items Integration](#9-quest-items-integration)
10. [Inventory Screen with Type Filters](#10-inventory-screen-with-type-filters)
11. [Shop Integration Helpers](#11-shop-integration-helpers)
12. [Extension Hook: Custom Item Types](#12-extension-hook-custom-item-types)
13. [Full Integration Checklist](#13-full-integration-checklist)

---

## 1. Architecture

```
systems/
├── inventory_system.rpy   # Item base + subclasses + Inventory + ITEM_CATALOGUE
└── flags.rpy              # default player_inventory + equipped slots

screens/
└── inventory.rpy          # Inventory screen with type filter tabs

images/
└── items/                 # Icons (one per item, fallback to default)
    ├── item_coffee.png
    ├── item_perfume.png
    └── item_key_quinta.png
```

**Design principle**: items are **data** (their definition lives once in `ITEM_CATALOGUE`). The `Inventory` only tracks **what the player has** (item_id → quantity). All behavior — what an item does when used, equipped, or gifted — lives in its catalogue entry. This keeps `Inventory` serializable and behavior centralized.

---

## 2. Item Base Class

```renpy
# systems/inventory_system.rpy

init python:

    # Standard item types — extension hook in Section 12 explains how to add new ones
    ITEM_TYPE_CONSUMABLE = "consumable"
    ITEM_TYPE_EQUIPMENT  = "equipment"
    ITEM_TYPE_GIFT       = "gift"
    ITEM_TYPE_QUEST      = "quest"
    ITEM_TYPE_KEY        = "key"

    ALL_ITEM_TYPES = [
        ITEM_TYPE_CONSUMABLE,
        ITEM_TYPE_EQUIPMENT,
        ITEM_TYPE_GIFT,
        ITEM_TYPE_QUEST,
        ITEM_TYPE_KEY,
    ]


    class Item:
        """
        Base item class. Don't instantiate directly — use a subclass.
        Override item_type in subclasses.
        """

        item_type = None    # set by subclass

        def __init__(self, item_id, name, description,
                     icon=None, value=0,
                     stackable=True, max_stack=None,
                     unique=False, sellable=True, discardable=True):
            self.item_id      = item_id
            self.name         = name
            self.description  = description
            self.icon         = icon or "images/items/_default.png"
            self.value        = value          # base shop value
            self.stackable    = stackable      # False = each pickup is a separate slot
            self.max_stack    = max_stack      # None = unlimited per slot
            self.unique       = unique         # True = inventory holds at most 1
            self.sellable     = sellable       # False = shop refuses
            self.discardable  = discardable    # False = player UI hides "Discard"

        def can_carry_more(self, current_qty):
            """Return True if the player can pick up at least one more of this."""
            if self.unique and current_qty >= 1:
                return False
            if self.max_stack is not None and current_qty >= self.max_stack:
                return False
            return True

        def __repr__(self):
            return "<Item {}:{}>".format(self.item_type, self.item_id)
```

---

## 3. Item Subclasses (5 Standard Types)

### 3.1 Consumable

Used once, applies effects to player stats (or arbitrary side effects), then disappears.

```renpy
init python:

    class Consumable(Item):
        """
        Single-use item. Effects are data-driven by default (a dict of stat → delta)
        with an optional callable for complex logic.
        """

        item_type = ITEM_TYPE_CONSUMABLE

        def __init__(self, item_id, name, description,
                     effects=None, effect_callable=None,
                     icon=None, value=0, stackable=True, max_stack=None,
                     side_effects=None):
            super().__init__(item_id, name, description,
                             icon=icon, value=value,
                             stackable=stackable, max_stack=max_stack,
                             discardable=True, sellable=True)
            self.effects         = effects or {}            # {"energy": +30, "image": -1}
            self.effect_callable = effect_callable          # callable(target) → None
            self.side_effects    = side_effects or []       # list of strings (narrative flags / notify msgs)

        def use(self, target):
            """
            Apply effects to target (typically store.player_survival).
            Returns True if the item was consumed.
            """
            for stat, delta in self.effects.items():
                if hasattr(target, "modify"):
                    target.modify(stat, delta)
                else:
                    cur = getattr(target, stat, None)
                    if cur is not None:
                        setattr(target, stat, cur + delta)
            if self.effect_callable:
                self.effect_callable(target)
            for msg in self.side_effects:
                renpy.notify(msg)
            return True
```

**Examples**:

```renpy
Consumable("item_coffee", "Coffee", "A cup of strong coffee. +20 energy.",
           effects={"energy": +20}, value=200),

Consumable("item_energy_drink", "Energy Drink", "+30 energy.",
           effects={"energy": +30}, value=500),

Consumable("item_cocaine", "White Powder", "+50 energy. Risky.",
           effects={"energy": +50},
           effect_callable=lambda t: npc_modify_global_suspicion(+1),
           side_effects=["You feel sharp. Maybe too sharp."],
           value=5000),
```

### 3.2 Equipment

Persistent gear that occupies a slot. Equipping applies passive modifiers; unequipping reverses them.

```renpy
init python:

    class Equipment(Item):
        """
        Wearable / equippable item. Persists until removed. Occupies a slot.
        Modifiers apply while equipped.
        """

        item_type = ITEM_TYPE_EQUIPMENT

        def __init__(self, item_id, name, description, slot,
                     modifiers=None,
                     icon=None, value=0):
            super().__init__(item_id, name, description,
                             icon=icon, value=value,
                             stackable=False, unique=False,
                             sellable=True, discardable=True)
            self.slot      = slot              # "outfit", "accessory", "weapon", etc.
            self.modifiers = modifiers or {}   # {"image": +5, "charisma": +3} while equipped

        def on_equip(self, target):
            for stat, delta in self.modifiers.items():
                if hasattr(target, "apply_modifier"):
                    target.apply_modifier(self.item_id, stat, delta)
                else:
                    cur = getattr(target, stat, None)
                    if cur is not None:
                        setattr(target, stat, cur + delta)

        def on_unequip(self, target):
            for stat, delta in self.modifiers.items():
                if hasattr(target, "remove_modifier"):
                    target.remove_modifier(self.item_id, stat)
                else:
                    cur = getattr(target, stat, None)
                    if cur is not None:
                        setattr(target, stat, cur - delta)
```

**Examples**:

```renpy
Equipment("item_political_suit", "Political Suit",
          "A formal suit. Required for Congress appearances.",
          slot="outfit",
          modifiers={"image": +10},
          value=50000),

Equipment("item_gold_watch", "Gold Watch", "Subtle status signal.",
          slot="accessory",
          modifiers={"image": +3},
          value=15000),
```

### 3.3 Gift

Consumed by **giving to an NPC** (not "using on self"). NPCs may have affinity bonuses for specific gifts.

```renpy
init python:

    class Gift(Item):
        """
        Gift item. Used by being given to an NPC. Some NPCs have affinity for specific gifts.
        """

        item_type = ITEM_TYPE_GIFT

        def __init__(self, item_id, name, description,
                     base_trust_delta=+5,
                     affinity=None,
                     icon=None, value=0, stackable=True):
            super().__init__(item_id, name, description,
                             icon=icon, value=value,
                             stackable=stackable,
                             discardable=True, sellable=True)
            self.base_trust_delta = base_trust_delta   # default trust gain when gifted
            self.affinity         = affinity or {}     # {npc_id: bonus_trust_delta}

        def trust_delta_for(self, npc_id):
            """Total trust delta when this gift is given to npc_id."""
            return self.base_trust_delta + self.affinity.get(npc_id, 0)
```

**Examples**:

```renpy
Gift("item_premium_perfume", "Carolina Herrera Perfume",
     "Imported perfume. Most women appreciate it.",
     base_trust_delta=+8,
     affinity={"vicky": +5},
     value=20000),

Gift("item_whisky_18", "Whisky 18 años",
     "A respected gift among older men of means.",
     base_trust_delta=+5,
     affinity={"lorenzo": +10},   # Lorenzo loves this
     value=15000),

Gift("item_protein_powder", "Premium Protein Mix",
     "Gym-bro tier gift.",
     base_trust_delta=+2,
     affinity={"romi": +8},
     value=5000),
```

### 3.4 QuestItem

Items tied to active quests. Cannot be sold, cannot be discarded while their quest is active. Visually grouped under "Quest" in the inventory screen.

```renpy
init python:

    class QuestItem(Item):
        """
        Quest-bound item. Auto-removed when the quest completes (optional).
        Cannot be sold or discarded while quest is active.
        """

        item_type = ITEM_TYPE_QUEST

        def __init__(self, item_id, name, description,
                     quest_id, auto_remove_on_complete=True,
                     icon=None):
            super().__init__(item_id, name, description,
                             icon=icon, value=0,
                             stackable=False, unique=True,
                             sellable=False, discardable=False)
            self.quest_id                = quest_id
            self.auto_remove_on_complete = auto_remove_on_complete
```

**Examples**:

```renpy
QuestItem("item_lorenzo_envelope", "Brown Envelope",
          "Subsecretary's documents. Lorenzo asked for these.",
          quest_id="quest_lorenzo_envelope"),

QuestItem("item_compromising_photo", "Compromising Photo",
          "A photograph you weren't supposed to take.",
          quest_id="quest_blackmail_judge",
          auto_remove_on_complete=False),   # keep as evidence
```

### 3.5 KeyItem

Unique progression items. Never stackable, never sellable, never discardable. Often invisible to the player (used by gameplay logic to gate scenes).

```renpy
init python:

    class KeyItem(Item):
        """
        Unique progression / narrative item. Permanent until story removes it explicitly.
        Hidden from inventory if `hidden=True`.
        """

        item_type = ITEM_TYPE_KEY

        def __init__(self, item_id, name, description,
                     icon=None, hidden=False):
            super().__init__(item_id, name, description,
                             icon=icon, value=0,
                             stackable=False, unique=True,
                             sellable=False, discardable=False)
            self.hidden = hidden
```

**Examples**:

```renpy
KeyItem("item_quinta_key", "Quinta Key",
        "Spare key to the quinta. Lorenzo gave it to you.",
        icon="images/items/key_quinta.png"),

KeyItem("item_press_credential", "Press Credential",
        "Lets you into TV studios and press conferences.",
        icon="images/items/press_pass.png"),

KeyItem("flag_knows_about_kuchner_scandal",
        "Hidden Knowledge",
        "Internal flag, not shown.",
        hidden=True),
```

---

## 4. ITEM_CATALOGUE Pattern

All items live in a single registry, defined declaratively at init time. The registry is the **source of truth** for item definitions; the Inventory only references items by their `item_id`.

```renpy
# systems/inventory_system.rpy (continued)

init python:

    def _build_item_catalogue():
        return {
            # ── Consumables ──────────────────────────────────────────────
            "item_coffee":        Consumable("item_coffee", "Coffee",
                                             "Strong cup of coffee.",
                                             effects={"energy": +20}, value=200),
            "item_energy_drink":  Consumable("item_energy_drink", "Energy Drink",
                                             "Sugar bomb.",
                                             effects={"energy": +30}, value=500),

            # ── Equipment ────────────────────────────────────────────────
            "item_political_suit": Equipment("item_political_suit",
                                              "Political Suit",
                                              "Required for Congress.",
                                              slot="outfit",
                                              modifiers={"image": +10},
                                              value=50000),

            # ── Gifts ────────────────────────────────────────────────────
            "item_premium_perfume": Gift("item_premium_perfume",
                                          "Carolina Herrera Perfume",
                                          "Imported perfume.",
                                          base_trust_delta=+8,
                                          affinity={"vicky": +5},
                                          value=20000),

            # ── Quest items ──────────────────────────────────────────────
            "item_lorenzo_envelope": QuestItem("item_lorenzo_envelope",
                                                "Brown Envelope",
                                                "Subsecretary's docs.",
                                                quest_id="quest_lorenzo_envelope"),

            # ── Key items ────────────────────────────────────────────────
            "item_quinta_key": KeyItem("item_quinta_key",
                                       "Quinta Key",
                                       "Spare key to the quinta."),
        }


    ITEM_CATALOGUE = _build_item_catalogue()


    def get_item(item_id):
        """Return the catalogue entry for an item_id, or None."""
        return ITEM_CATALOGUE.get(item_id)
```

**Convention**: prefix all item_ids with `item_`. Use `snake_case`. Group related items in the catalogue with section comments.

---

## 5. Inventory Class

The Inventory tracks **what the player owns**: a dict from `item_id` to quantity. Behavior delegates to `ITEM_CATALOGUE`.

```renpy
# systems/inventory_system.rpy (continued)

init python:

    class Inventory:
        """
        Tracks the player's owned items. All behavior delegates to ITEM_CATALOGUE.
        Items are referenced by item_id string; quantity is an integer.

        Supports:
        - stackable / non-stackable items (per item's `stackable` flag)
        - unique items (max 1 in inventory)
        - max_stack per item
        - optional overall max_capacity (None = unlimited)
        """

        def __init__(self, max_capacity=None):
            self._items       = {}              # {item_id: quantity}
            self.max_capacity = max_capacity    # None = unlimited

        # ── Capacity ───────────────────────────────────────────────────
        def total_count(self):
            return sum(self._items.values())

        def is_full(self):
            if self.max_capacity is None:
                return False
            return self.total_count() >= self.max_capacity

        def can_add(self, item_id, qty=1):
            item = get_item(item_id)
            if not item:
                return False
            current = self._items.get(item_id, 0)
            if not item.can_carry_more(current):
                return False
            if self.max_capacity is not None and self.total_count() + qty > self.max_capacity:
                return False
            return True

        # ── Add / Remove ───────────────────────────────────────────────
        def add(self, item_id, qty=1):
            """Add qty of item_id. Returns the qty actually added (may be less if capacity)."""
            item = get_item(item_id)
            if not item or qty <= 0:
                return 0
            current = self._items.get(item_id, 0)
            # Compute max addable given unique / max_stack / capacity
            addable = qty
            if item.unique:
                addable = min(addable, max(0, 1 - current))
            if item.max_stack is not None:
                addable = min(addable, max(0, item.max_stack - current))
            if self.max_capacity is not None:
                free = self.max_capacity - self.total_count()
                addable = min(addable, max(0, free))
            if addable <= 0:
                return 0
            self._items[item_id] = current + addable
            return addable

        def remove(self, item_id, qty=1):
            """Remove qty. Returns True if removed, False if not enough."""
            if self._items.get(item_id, 0) < qty:
                return False
            self._items[item_id] -= qty
            if self._items[item_id] <= 0:
                del self._items[item_id]
            return True

        # ── Query ──────────────────────────────────────────────────────
        def has(self, item_id, qty=1):
            return self._items.get(item_id, 0) >= qty

        def count(self, item_id):
            return self._items.get(item_id, 0)

        def all_items(self):
            """Returns list of (Item, quantity) for items currently held."""
            return [(get_item(k), v) for k, v in self._items.items() if get_item(k)]

        def all_by_type(self, item_type):
            """Filter held items by type."""
            return [(item, qty) for (item, qty) in self.all_items()
                    if item.item_type == item_type]

        def filter(self, predicate):
            """Filter by arbitrary callable(item, qty) → bool."""
            return [(item, qty) for (item, qty) in self.all_items() if predicate(item, qty)]

        def visible_items(self):
            """All items except hidden key items (used by player-facing UI)."""
            return [(item, qty) for (item, qty) in self.all_items()
                    if not (item.item_type == ITEM_TYPE_KEY and getattr(item, "hidden", False))]


    # ── Default global instance ────────────────────────────────────────
    default player_inventory = None

    def init_inventory(max_capacity=None, starting_items=None):
        store.player_inventory = Inventory(max_capacity=max_capacity)
        for item_id, qty in (starting_items or {}).items():
            store.player_inventory.add(item_id, qty)
```

**Usage**:

```renpy
# At game start
$ init_inventory(starting_items={"item_coffee": 2})

# Add
$ player_inventory.add("item_premium_perfume", 1)

# Query
if player_inventory.has("item_lorenzo_envelope"):
    "You hand over the envelope."
    $ player_inventory.remove("item_lorenzo_envelope")

# Use a consumable
python:
    item = get_item("item_coffee")
    if player_inventory.has("item_coffee") and isinstance(item, Consumable):
        item.use(player_survival)
        player_inventory.remove("item_coffee")
```

---

## 6. Persistence & Save Integration

The `Inventory` only stores plain dicts (`{item_id: qty}`) and is fully serializable by Ren'Py's pickle-based save system out of the box. No extra work needed for per-save state.

For **cross-save** items (e.g., "trophy items" that persist across new games for completionists), use `persistent.*`:

```renpy
# systems/flags.rpy
define persistent.unlocked_trophy_items = []   # list of item_ids ever obtained

# When picking up a tagged item, also flag in persistent
init python:
    TROPHY_ITEMS = {"item_quinta_key", "item_political_suit"}

    _base_add = Inventory.add

    def inventory_add_with_trophy(self, item_id, qty=1):
        result = _base_add(self, item_id, qty)
        if result > 0 and item_id in TROPHY_ITEMS:
            if item_id not in persistent.unlocked_trophy_items:
                persistent.unlocked_trophy_items.append(item_id)
                renpy.save_persistent()
        return result

    Inventory.add = inventory_add_with_trophy
```

**Catalogue versioning**: if you add new fields to item classes in updates, write a migration in `after_load`:

```renpy
label after_load:
    python:
        # Drop any item_ids that no longer exist in the current catalogue
        if store.player_inventory:
            stale = [k for k in store.player_inventory._items
                     if k not in ITEM_CATALOGUE]
            for k in stale:
                del store.player_inventory._items[k]
    return
```

---

## 7. Equipment Slots & Equipping

Equipment lives in the inventory (so it can be unequipped and re-equipped) plus tracks **which item is currently equipped per slot**.

```renpy
# systems/inventory_system.rpy (continued)

init python:

    class EquipmentSlots:
        """Tracks the currently equipped item per slot (item_id or None)."""

        def __init__(self, slots):
            # slots: list of slot names (["outfit", "accessory", "weapon"])
            self.slot_names = list(slots)
            self.equipped   = {name: None for name in slots}

        def equip(self, item_id, target):
            """
            Equip item_id (must be in inventory). Auto-unequips current slot occupant.
            Returns True if equipped.
            """
            item = get_item(item_id)
            if not item or item.item_type != ITEM_TYPE_EQUIPMENT:
                return False
            if not store.player_inventory.has(item_id):
                return False
            slot = item.slot
            if slot not in self.equipped:
                return False
            # Unequip current occupant
            if self.equipped[slot]:
                self.unequip(slot, target)
            # Equip new
            self.equipped[slot] = item_id
            item.on_equip(target)
            return True

        def unequip(self, slot, target):
            item_id = self.equipped.get(slot)
            if not item_id:
                return False
            item = get_item(item_id)
            if item:
                item.on_unequip(target)
            self.equipped[slot] = None
            return True

        def equipped_in(self, slot):
            return self.equipped.get(slot)

        def has_equipped(self, item_id):
            return item_id in self.equipped.values()


    default equipment_slots = None

    def init_equipment_slots(slots=("outfit", "accessory")):
        store.equipment_slots = EquipmentSlots(slots)
```

**Usage**:

```renpy
$ init_equipment_slots(slots=("outfit", "accessory"))

$ player_inventory.add("item_political_suit", 1)
$ equipment_slots.equip("item_political_suit", player_survival)
# → player_survival now has image +10 applied via on_equip
```

**Gating in story**: check whether an item is equipped before allowing certain actions.

```renpy
label enter_congress:
    if not equipment_slots.has_equipped("item_political_suit"):
        "You can't enter Congress dressed like that."
        jump hub_main
    "The guard waves you through."
    jump congress_hall
```

---

## 8. Gifting Pattern (Give to NPC)

Gifts are consumed by giving them to an NPC; the NPC's trust (or other relationship stat) increases by the gift's `trust_delta_for(npc_id)`.

```renpy
# systems/gifting_system.rpy

init python:

    def give_gift(item_id, npc_id, npc_stats=None):
        """
        Give a Gift item to an NPC. Consumes from inventory, modifies NPC trust.
        Returns True if the gift was given.

        npc_stats: object supporting .modify(stat, delta). If None, falls back to NPC_STATS dict.
        """
        item = get_item(item_id)
        if not item or item.item_type != ITEM_TYPE_GIFT:
            return False
        if not store.player_inventory.has(item_id):
            renpy.notify("You don't have that.")
            return False

        delta = item.trust_delta_for(npc_id)
        target = npc_stats
        if target is None and hasattr(store, "NPC_STATS"):
            target = store.NPC_STATS.get(npc_id)
        if target and hasattr(target, "modify"):
            target.modify("trust", delta)
        store.player_inventory.remove(item_id, 1)
        renpy.notify("Gave {} to {} (+{} trust).".format(item.name, npc_id, delta))
        return True


    def available_gifts_for(npc_id):
        """Return list of (gift_item, qty, trust_delta) the player can give to npc_id."""
        out = []
        for (item, qty) in store.player_inventory.all_by_type(ITEM_TYPE_GIFT):
            out.append((item, qty, item.trust_delta_for(npc_id)))
        # Sort by best trust delta first
        out.sort(key=lambda t: -t[2])
        return out
```

**Story usage**:

```renpy
label talk_to_lorenzo:
    "Lorenzo greets you with his usual back-slap."
    menu:
        "Give him a gift":
            python:
                gifts = available_gifts_for("lorenzo")
            if not gifts:
                "You don't have anything to offer right now."
            else:
                # Show a small picker
                $ chosen = renpy.display_menu([(g[0].name + " (+{})".format(g[2]), g[0].item_id)
                                                for g in gifts] + [("Cancel", None)])
                if chosen:
                    $ give_gift(chosen, "lorenzo")
        "Talk":
            jump lorenzo_dialogue
        "Leave":
            jump hub_main
```

---

## 9. Quest Items Integration

`QuestItem` ties to a quest via `quest_id`. Cross-skill integration with `renpy-quests.md`:

- **Cannot be sold**: `sellable = False` (shops refuse).
- **Cannot be discarded** while the linked quest is active.
- **Auto-removed** on quest completion if `auto_remove_on_complete=True`.

```renpy
init python:

    def cleanup_quest_items_on_complete(quest_id):
        """Call after a quest completes. Removes its quest items if auto-cleanup is on."""
        for (item, qty) in list(store.player_inventory.all_by_type(ITEM_TYPE_QUEST)):
            if (item.quest_id == quest_id
                and getattr(item, "auto_remove_on_complete", False)):
                store.player_inventory.remove(item.item_id, qty)


    def can_discard(item):
        """Used by inventory UI to decide whether the Discard button appears."""
        if not item.discardable:
            return False
        if item.item_type == ITEM_TYPE_QUEST:
            quest = store.quest_manager.get(item.quest_id) if store.quest_manager else None
            if quest and quest.state == QUEST_ACTIVE:
                return False
        return True
```

Hook into quest completion (see `renpy-quests.md`):

```renpy
# In Quest.complete()
def complete(self):
    if self.state == QUEST_ACTIVE:
        self.state = QUEST_COMPLETE
        apply_quest_rewards(self)
        cleanup_quest_items_on_complete(self.quest_id)   # ← added
        if self.on_complete:
            self.on_complete()
        return True
    return False
```

For `ItemObjective` (also in `renpy-quests.md`), the existing pattern works unchanged — `player_inventory.has(item_id, qty)` is the canonical check.

---

## 10. Inventory Screen with Type Filters

Tabbed inventory screen with type filters. Each tab shows a viewport of items, with use/equip/gift/discard actions where applicable.

```renpy
# screens/inventory.rpy

default inventory_tab = ITEM_TYPE_CONSUMABLE

screen inventory_screen():
    modal True
    zorder 50
    tag menu

    frame:
        xalign 0.5 yalign 0.5
        xsize 820 ysize 580
        padding (20, 16)

        vbox:
            spacing 10

            text "Inventory" size 24 bold True xalign 0.5

            # Tab bar
            hbox:
                xalign 0.5 spacing 14
                for tab_id, label in [
                    (ITEM_TYPE_CONSUMABLE, "Consumables"),
                    (ITEM_TYPE_EQUIPMENT,  "Equipment"),
                    (ITEM_TYPE_GIFT,       "Gifts"),
                    (ITEM_TYPE_QUEST,      "Quest"),
                    (ITEM_TYPE_KEY,        "Key Items"),
                ]:
                    textbutton label:
                        action SetScreenVariable("inventory_tab", tab_id)
                        text_color ("#f1c40f" if inventory_tab == tab_id else "#888")

            # Item list for current tab
            python:
                tab_items = [(item, qty) for (item, qty) in store.player_inventory.all_by_type(inventory_tab)
                             if not (item.item_type == ITEM_TYPE_KEY and getattr(item, "hidden", False))]

            viewport:
                scrollbars "vertical"
                mousewheel True
                ysize 400
                vbox:
                    spacing 8
                    if not tab_items:
                        text "Nothing in this category." size 14 color "#888" xalign 0.5
                    for (item, qty) in tab_items:
                        frame:
                            padding (10, 8)
                            background "#1a1a1aaa"
                            hbox:
                                spacing 12
                                xfill True
                                add item.icon xsize 48 ysize 48
                                vbox:
                                    spacing 2
                                    text "[item.name]  ×[qty]" size 16 bold True
                                    text item.description size 13 color "#bbb"
                                null xfill True
                                # Per-type actions
                                if item.item_type == ITEM_TYPE_CONSUMABLE:
                                    textbutton "Use":
                                        action [
                                            Function(item.use, store.player_survival),
                                            Function(store.player_inventory.remove, item.item_id),
                                            renpy.restart_interaction,
                                        ]
                                if item.item_type == ITEM_TYPE_EQUIPMENT:
                                    if equipment_slots and equipment_slots.has_equipped(item.item_id):
                                        textbutton "Unequip":
                                            action [
                                                Function(equipment_slots.unequip, item.slot, store.player_survival),
                                                renpy.restart_interaction,
                                            ]
                                    else:
                                        textbutton "Equip":
                                            action [
                                                Function(equipment_slots.equip, item.item_id, store.player_survival),
                                                renpy.restart_interaction,
                                            ]
                                if can_discard(item):
                                    textbutton "Discard":
                                        action [
                                            Function(store.player_inventory.remove, item.item_id),
                                            renpy.restart_interaction,
                                        ]

            textbutton "Close" action Return() xalign 0.5
```

This screen replaces the simpler inventory_screen in `renpy-screens.md` § 4 (which now cross-refs here). It supports all 5 item types with appropriate buttons per type.

---

## 11. Shop Integration Helpers

Shops sell items from the catalogue. Standard helpers:

```renpy
# systems/shop_system.rpy

init python:

    class Shop:
        """A shop with a list of buyable item_ids and a markup."""

        def __init__(self, shop_id, name, item_ids,
                     markup=1.0, buyback_ratio=0.5):
            self.shop_id       = shop_id
            self.name          = name
            self.item_ids      = item_ids       # list of item_ids the shop sells
            self.markup        = markup         # multiplier on item.value for buying
            self.buyback_ratio = buyback_ratio  # ratio applied when shop buys from player

        def buy_price(self, item_id):
            item = get_item(item_id)
            return int(item.value * self.markup) if item else 0

        def sell_price(self, item_id):
            item = get_item(item_id)
            if not item or not item.sellable:
                return 0
            return int(item.value * self.buyback_ratio)

        def can_buy(self, item_id):
            """Can the player buy this item from this shop right now?"""
            if item_id not in self.item_ids:
                return False
            price = self.buy_price(item_id)
            if store.player_survival.money < price:
                return False
            if not store.player_inventory.can_add(item_id, 1):
                return False
            return True

        def buy(self, item_id):
            if not self.can_buy(item_id):
                return False
            price = self.buy_price(item_id)
            store.player_survival.modify("money", -price)
            store.player_inventory.add(item_id, 1)
            renpy.notify("Bought {} for ${}.".format(get_item(item_id).name, price))
            return True

        def sell(self, item_id):
            if not store.player_inventory.has(item_id):
                return False
            item = get_item(item_id)
            if not item or not item.sellable:
                renpy.notify("This can't be sold.")
                return False
            price = self.sell_price(item_id)
            store.player_inventory.remove(item_id, 1)
            store.player_survival.modify("money", +price)
            renpy.notify("Sold {} for ${}.".format(item.name, price))
            return True


    SHOPS = {
        "supermarket": Shop("supermarket", "Supermarket",
                            item_ids=["item_coffee", "item_energy_drink"],
                            markup=1.0, buyback_ratio=0.0),
        "boutique":    Shop("boutique", "Boutique",
                            item_ids=["item_political_suit"],
                            markup=1.0, buyback_ratio=0.5),
        "jewelry":     Shop("jewelry", "Jewelry Store",
                            item_ids=["item_premium_perfume"],
                            markup=1.2, buyback_ratio=0.4),
    }
```

A shop screen iterates `SHOPS[shop_id].item_ids` and offers buy/sell per item. The shop pattern only references item_ids — never assumes anything about item internals beyond what `Item` exposes (`value`, `sellable`, `name`).

---

## 12. Extension Hook: Custom Item Types

To add a new item type beyond the 5 standard ones (e.g., `Recipe`, `Vehicle`, `RealEstate`), subclass `Item` and register the new type string:

```renpy
init python:

    ITEM_TYPE_RECIPE = "recipe"
    ALL_ITEM_TYPES.append(ITEM_TYPE_RECIPE)


    class Recipe(Item):
        """Custom type: a cooking recipe that unlocks a new craftable item."""

        item_type = ITEM_TYPE_RECIPE

        def __init__(self, item_id, name, description,
                     unlocks_item_id, icon=None):
            super().__init__(item_id, name, description,
                             icon=icon, value=100,
                             stackable=False, unique=True,
                             sellable=True, discardable=True)
            self.unlocks_item_id = unlocks_item_id

        def learn(self):
            """Use action for recipes: add to player's known recipes."""
            if not hasattr(store, "known_recipes"):
                store.known_recipes = set()
            store.known_recipes.add(self.unlocks_item_id)
            renpy.notify("Learned: can now craft {}.".format(
                get_item(self.unlocks_item_id).name))


    # Add to catalogue as normal
    ITEM_CATALOGUE["item_recipe_choripan"] = Recipe(
        "item_recipe_choripan", "Choripán Recipe",
        "Granny's hand-written recipe.",
        unlocks_item_id="item_choripan_homemade",
    )
```

The inventory screen automatically handles unknown types — they appear under their type's tab if you add the tab in the tab bar (see Section 10). For full integration, also add a per-type action button (e.g., a "Learn" button for `Recipe`) in the inventory screen's per-item action block.

---

## 13. Full Integration Checklist

Before shipping a game built on this skill:

**Init**:
- [ ] `init_inventory(starting_items=...)` called at game start (and `after_load` if migrating saves).
- [ ] `init_equipment_slots(slots=...)` called if equipment is used.
- [ ] `ITEM_CATALOGUE` populated and validated (`for item_id, item in ITEM_CATALOGUE.items(): assert item.item_id == item_id`).

**Items**:
- [ ] Every item in `ITEM_CATALOGUE` has an `icon` (or default fallback exists).
- [ ] Consumables have either `effects` or `effect_callable` (not both empty).
- [ ] Equipment has `slot` set and `modifiers` defined (even if empty).
- [ ] Gifts have meaningful `base_trust_delta` and `affinity` for at least the main NPCs.
- [ ] Quest items have a real `quest_id` matching an entry in the quest registry.
- [ ] Key items are intentional (no accidental discardable=False).

**UI**:
- [ ] Inventory screen accessible from hub / phone overlay.
- [ ] Tabs cover all `item_type` values present in catalogue.
- [ ] `can_discard()` check used everywhere a discard button appears.
- [ ] Hidden key items not visible in player UI (`hidden=True` respected).

**Cross-skill integration**:
- [ ] `ItemObjective` in `renpy-quests.md` uses `player_inventory.has` (compatible by default).
- [ ] Quest completion calls `cleanup_quest_items_on_complete(quest_id)`.
- [ ] Shops use the `Shop` helper and don't bypass `Item.sellable`.
- [ ] Gifts use `give_gift()` (don't mutate trust inline scattered across the codebase).

**Persistence**:
- [ ] Save / load round-trip tested (Inventory survives save → load → save).
- [ ] `after_load` cleanup for items removed from catalogue between versions.
- [ ] Trophy items list in `persistent.*` if cross-save unlocks are needed.
