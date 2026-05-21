# Ren'Py RPG Subsystem Reference

Use this when the project includes stats, leveling, skills, equipment, or RPG-like progression.

## When NOT to use this skill

Don't apply this skill to **life-sim / political-sim / sandbox VN** projects, even if they have numeric stats. RPG patterns (HP/ATK/DEF, leveling, combat turns) are a poor fit for narrative stats like *Influence*, *Reputation*, *Energy*, or *Mood*. For those:

- **Survival-style stats** (energy, mood, fitness, money, daily decay) → use `renpy-survival.md`.
- **Multi-dimensional narrative reputation** (trust, suspicion, public image) → use `renpy-adult-vn.md` § Multi-Dimensional NPC Stats and § City-Wide Reputation.
- **Stats opaque to the player** (qualitative indicators instead of numbers) → use `renpy-adult-vn.md` § Opaque NPC Stats.

The pieces of this skill that **do** translate to non-RPG games are: the `Inventory` / `Item` / `ITEM_CATALOGUE` pattern in Section 4 (reusable as-is) and the stat-check helper in Section 6 (the soft check pattern works for any threshold).

---

## Table of Contents
1. [Stats Architecture](#1-stats-architecture)
2. [Leveling & XP](#2-leveling--xp)
3. [Skills & Abilities](#3-skills--abilities)
4. [Equipment & Items](#4-equipment--items)
5. [Combat (Turn-Based)](#5-combat-turn-based)
6. [Stat Checks in Dialogue](#6-stat-checks-in-dialogue)
7. [HUD Integration](#7-hud-integration)

---

## 1. Stats Architecture

Define all stat data as Python classes in `systems/stats_system.rpy`.

```renpy
# systems/stats_system.rpy

init python:

    class CharacterStats:
        """Holds all RPG stats for a single character."""

        def __init__(self, name, base_hp=100, base_atk=10, base_def=5, base_spd=10):
            self.name = name

            # Base values (modified by equipment, permanent upgrades)
            self.base_hp  = base_hp
            self.base_atk = base_atk
            self.base_def = base_def
            self.base_spd = base_spd

            # Current values (fluctuate during scenes/combat)
            self.current_hp = base_hp
            self.level = 1
            self.xp = 0
            self.xp_to_next = 100

            # Buffs: {stat_name: value}, cleared between scenes
            self._buffs = {}

        # ── Derived Stats ──────────────────────────────────────────
        @property
        def max_hp(self):
            return self.base_hp + self._buffs.get("hp", 0)

        @property
        def attack(self):
            return self.base_atk + self._buffs.get("atk", 0)

        @property
        def defense(self):
            return self.base_def + self._buffs.get("def", 0)

        # ── Methods ────────────────────────────────────────────────
        def take_damage(self, amount):
            """Apply damage after defense mitigation. Returns actual damage dealt."""
            actual = max(1, amount - self.defense)
            self.current_hp = max(0, self.current_hp - actual)
            return actual

        def heal(self, amount):
            self.current_hp = min(self.max_hp, self.current_hp + amount)

        def is_alive(self):
            return self.current_hp > 0

        def apply_buff(self, stat, value):
            self._buffs[stat] = self._buffs.get(stat, 0) + value

        def clear_buffs(self):
            self._buffs = {}

        def full_restore(self):
            self.current_hp = self.max_hp
            self.clear_buffs()


# ── Initialize Player Stats ─────────────────────────────────────────
default player_stats = None   # Set during game init or name screen

init python:
    def init_player_stats(name):
        """Call this when the player is created / game starts."""
        store.player_stats = CharacterStats(name, base_hp=120, base_atk=12, base_def=6)
```

---

## 2. Leveling & XP

```renpy
# systems/stats_system.rpy (continued)

init python:

    def gain_xp(stats, amount):
        """Grant XP and handle level-ups. Returns list of level-up messages."""
        messages = []
        stats.xp += amount

        while stats.xp >= stats.xp_to_next:
            stats.xp -= stats.xp_to_next
            stats.level += 1
            stats.xp_to_next = int(stats.xp_to_next * 1.4)

            # Stat growth on level-up
            stats.base_hp  += 10
            stats.base_atk += 2
            stats.base_def += 1
            stats.full_restore()

            messages.append("Level Up! Now level {}.".format(stats.level))

        return messages
```

**In story files:**

```renpy
label quest_01_reward:
    python:
        msgs = gain_xp(player_stats, 50)
        for msg in msgs:
            renpy.notify(msg)

    "You gained 50 experience."
    jump chapter_02
```

---

## 3. Skills & Abilities

```renpy
# systems/stats_system.rpy

init python:

    class Skill:
        """A usable skill or ability."""

        def __init__(self, name, description, mp_cost=0, cooldown=0):
            self.name        = name
            self.description = description
            self.mp_cost     = mp_cost
            self.cooldown    = cooldown
            self._cd_remaining = 0

        def is_ready(self):
            return self._cd_remaining == 0

        def use(self):
            if not self.is_ready():
                return False
            self._cd_remaining = self.cooldown
            return True

        def tick_cooldown(self):
            if self._cd_remaining > 0:
                self._cd_remaining -= 1

    # Skill definitions (data-driven)
    SKILL_CATALOGUE = {
        "heal":       Skill("Heal",        "Restore 30 HP.",           mp_cost=10),
        "shield":     Skill("Shield",      "Raise DEF by 5 for 2 turns.", mp_cost=8, cooldown=2),
        "power_strike": Skill("Power Strike", "Deal 150% ATK damage.", mp_cost=15, cooldown=1),
    }

# Player starts with some skills
default player_skills = ["heal"]
```

---

## 4. Equipment & Items

> **Full inventory system → see `renpy-inventory.md`** for the canonical reference: 5 item types (Consumable / Equipment / Gift / QuestItem / KeyItem), `ITEM_CATALOGUE` pattern, persistence, equipping, gifting, shop integration, and inventory screen with type filters.
>
> The minimal RPG-flavored version below is self-contained — sufficient for a small dungeon-crawler or combat-first project. For sandbox VNs, life-sim, dating sims, or anything with gifts/equipment/quest items, prefer the full skill.

Minimal Item + Inventory (RPG-flavored, suitable for combat games with potions and weapons):

```renpy
# systems/inventory_system.rpy

init python:

    class Item:
        """Minimal item class. For richer typing, use renpy-inventory.md."""

        def __init__(self, item_id, name, description,
                     item_type="misc", value=0, effect=None):
            self.item_id     = item_id
            self.name        = name
            self.description = description
            self.item_type   = item_type  # "consumable", "equipment", "key", "misc"
            self.value       = value
            self.effect      = effect     # callable(stats) → None, or None

        def use(self, target_stats):
            if self.effect and callable(self.effect):
                self.effect(target_stats)
                return True
            return False


    ITEM_CATALOGUE = {
        "potion":      Item("potion",      "Healing Potion",  "Restore 50 HP.",   "consumable", 30, lambda s: s.heal(50)),
        "elixir":      Item("elixir",      "Full Elixir",     "Fully restore HP.", "consumable", 150, lambda s: s.full_restore()),
        "iron_sword":  Item("iron_sword",  "Iron Sword",      "+5 ATK.",           "equipment",  200),
        "map_fragment":Item("map_fragment","Map Fragment",     "A torn map piece.", "key",        0),
    }


    class Inventory:
        """Minimal inventory. For richer typing, use renpy-inventory.md."""

        def __init__(self):
            self._items = {}  # {item_id: quantity}

        def add(self, item_id, qty=1):
            self._items[item_id] = self._items.get(item_id, 0) + qty

        def remove(self, item_id, qty=1):
            if self._items.get(item_id, 0) < qty:
                return False
            self._items[item_id] -= qty
            if self._items[item_id] == 0:
                del self._items[item_id]
            return True

        def has(self, item_id, qty=1):
            return self._items.get(item_id, 0) >= qty

        def count(self, item_id):
            return self._items.get(item_id, 0)

        def all_items(self):
            return [(ITEM_CATALOGUE[k], v) for k, v in self._items.items() if k in ITEM_CATALOGUE]


default player_inventory = None

init python:
    def init_inventory():
        store.player_inventory = Inventory()
        store.player_inventory.add("potion", 2)
```

**When to upgrade to `renpy-inventory.md`**: if you need any of — gift affinity per NPC, quest-bound items that can't be sold, equipment with slot management and modifier hooks, hidden key items, max capacity, type-filtered inventory UI, shop helpers, persistent trophy items. The migration path is straightforward: the minimal `Item` / `Inventory` classes above are API-compatible subsets of the full ones.

---

## 5. Combat (Turn-Based)

Keep combat logic in Python; display it with screens.

```renpy
# systems/stats_system.rpy

init python:

    class BattleState:
        """Manages a single turn-based battle encounter."""

        def __init__(self, player_stats, enemies):
            self.player   = player_stats
            self.enemies  = enemies          # list of CharacterStats
            self.turn     = 0
            self.log      = []               # list of strings for battle log
            self.over     = False
            self.victory  = False

        def player_attack(self, target_index=0):
            target = self.enemies[target_index]
            dmg = target.take_damage(self.player.attack)
            self.log.append("{} hit {} for {} damage!".format(
                self.player.name, target.name, dmg))
            self._check_enemy_deaths()
            if not self.over:
                self._enemy_turn()

        def player_use_skill(self, skill_id, target_index=0):
            # Expand per skill as needed
            skill = SKILL_CATALOGUE.get(skill_id)
            if not skill or not skill.use():
                self.log.append("Cannot use skill right now.")
                return
            # Apply skill effect (extend this per skill)
            self.log.append("{} used {}!".format(self.player.name, skill.name))
            self._enemy_turn()

        def _enemy_turn(self):
            for enemy in self.enemies:
                if enemy.is_alive():
                    dmg = self.player.take_damage(enemy.attack)
                    self.log.append("{} hit {} for {} damage!".format(
                        enemy.name, self.player.name, dmg))
            if not self.player.is_alive():
                self.over = True
                self.victory = False

        def _check_enemy_deaths(self):
            all_dead = all(not e.is_alive() for e in self.enemies)
            if all_dead:
                self.over = True
                self.victory = True

        def get_alive_enemies(self):
            return [e for e in self.enemies if e.is_alive()]
```

**Launching a battle in story:**

```renpy
label fight_forest_guardian:
    python:
        guardian = CharacterStats("Forest Guardian", base_hp=80, base_atk=14, base_def=3)
        battle = BattleState(player_stats, [guardian])
        store.current_battle = battle

    call screen battle_screen
    # battle_screen sets _return to "victory" or "defeat"

    if _return == "victory":
        $ gain_xp(player_stats, 80)
        "You defeated the Forest Guardian!"
        jump forest_path_clear
    else:
        "You were overwhelmed..."
        jump game_over_forest
```

---

## 6. Stat Checks in Dialogue

```renpy
# Soft check — different dialogue but no hard block
label interrogate_guard:
    if player_stats.attack >= 15:
        "You flex menacingly. The guard steps aside."
        $ met_guard_intimidated = True
    elif player_inventory.has("forged_pass"):
        "You hand over the forged pass. He waves you through."
    else:
        "The guard blocks your path."
        jump quest_failed_checkpoint

# Threshold check helper (optional utility)
init python:
    def stat_check(stat_value, difficulty):
        """Returns True if stat meets difficulty threshold."""
        return stat_value >= difficulty
```

---

## 7. HUD Integration

```renpy
# screens/stats.rpy

screen stats_hud():
    zorder 10
    frame:
        xalign 0.0 yalign 0.0
        xoffset 15 yoffset 15
        padding (10, 8)
        vbox:
            spacing 4
            text "[player_stats.name]" size 16 bold True
            hbox:
                spacing 6
                text "HP" size 14 color "#e74c3c"
                bar value player_stats.current_hp range player_stats.max_hp xsize 100 ysize 12
                text "[player_stats.current_hp]/[player_stats.max_hp]" size 13
            text "LV [player_stats.level]  XP [player_stats.xp]/[player_stats.xp_to_next]" size 13

# Show HUD when entering RPG segments
label rpg_zone_enter:
    show screen stats_hud
    # ...

label rpg_zone_exit:
    hide screen stats_hud
```