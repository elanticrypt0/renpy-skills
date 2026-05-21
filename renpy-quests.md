# Ren'Py Quest / Mission System Reference

Use this when the project has structured missions, objectives, or storyline tasks the player must track and complete. Applies to sandbox VNs, RPGs, and any game with a "things to do" loop.

---

## Table of Contents
1. [Architecture](#1-architecture)
2. [Quest State Machine](#2-quest-state-machine)
3. [Objective Types](#3-objective-types)
4. [Quest Registry](#4-quest-registry)
5. [Quest Manager](#5-quest-manager)
6. [Quest Journal Screen](#6-quest-journal-screen)
7. [Rewards & Side Effects](#7-rewards--side-effects)
8. [Quest Chains](#8-quest-chains)
9. [Integration Patterns](#9-integration-patterns)
10. [Multi-Stage Quests](#10-multi-stage-quests)
11. [Procedural Quests from Templates](#11-procedural-quests-from-templates)
12. [Soft Deadlines & Expiration](#12-soft-deadlines--expiration)
13. [Quest Categories & UI Filtering](#13-quest-categories--ui-filtering)
14. [Quest Cancellation](#14-quest-cancellation)
15. [Repeatable Quests](#15-repeatable-quests)

---

## 1. Architecture

```
systems/
├── quest_system.rpy      # Quest, Objective, QuestManager classes
├── quest_data.rpy        # All quest definitions (data only)
└── flags.rpy             # default quest_manager

screens/
└── quest_journal.rpy     # Journal / mission log screen

story/
└── quests/
    ├── quest_intro.rpy   # One file per major quest's story labels
    └── quest_npc_a.rpy
```

**Design principle:** Quests are data (state + objectives + rewards). Story labels are called *by* quest events, not the other way around. The quest manager is the source of truth.

---

## 2. Quest State Machine

```renpy
# systems/quest_system.rpy

init python:

    # Quest states
    QUEST_HIDDEN    = "hidden"     # Player doesn't know it exists yet
    QUEST_AVAILABLE = "available"  # Can be picked up
    QUEST_ACTIVE    = "active"     # In progress
    QUEST_COMPLETE  = "complete"   # Successfully finished
    QUEST_FAILED    = "failed"     # Failed (optional, not all quests can fail)


    class Quest:
        """
        Represents a single quest with objectives, rewards, and state.
        """

        def __init__(self, quest_id, title, description,
                     objectives=None, rewards=None,
                     unlock_condition=None,
                     on_complete=None,
                     on_fail=None,
                     chain_next=None):
            self.quest_id         = quest_id
            self.title            = title
            self.description      = description
            self.objectives       = objectives or []   # list of Objective instances
            self.rewards          = rewards or []      # list of Reward instances
            self.state            = QUEST_HIDDEN
            self.unlock_condition = unlock_condition   # callable() → bool, or None
            self.on_complete      = on_complete        # callable(), called on completion
            self.on_fail          = on_fail            # callable(), called on failure
            self.chain_next       = chain_next         # quest_id to auto-unlock on completion

        # ── State transitions ──────────────────────────────────────────
        def try_unlock(self):
            """Check condition and move to available if met."""
            if self.state == QUEST_HIDDEN:
                if self.unlock_condition is None or self.unlock_condition():
                    self.state = QUEST_AVAILABLE
                    return True
            return False

        def start(self):
            if self.state == QUEST_AVAILABLE:
                self.state = QUEST_ACTIVE
                for obj in self.objectives:
                    obj.reset()
                return True
            return False

        def complete(self):
            if self.state == QUEST_ACTIVE:
                self.state = QUEST_COMPLETE
                if self.on_complete:
                    self.on_complete()
                return True
            return False

        def fail(self):
            if self.state == QUEST_ACTIVE:
                self.state = QUEST_FAILED
                if self.on_fail:
                    self.on_fail()
                return True
            return False

        # ── Progress ───────────────────────────────────────────────────
        def all_objectives_met(self):
            return all(obj.is_complete() for obj in self.objectives)

        def progress_text(self):
            done = sum(1 for o in self.objectives if o.is_complete())
            return "{}/{}".format(done, len(self.objectives))

        def check_auto_complete(self):
            """Call after updating objectives. Auto-completes if all are met."""
            if self.state == QUEST_ACTIVE and self.all_objectives_met():
                self.complete()
                return True
            return False
```

---

## 3. Objective Types

```renpy
# systems/quest_system.rpy (continued)

init python:

    class Objective:
        """Base objective class."""

        def __init__(self, description):
            self.description = description
            self._complete   = False

        def is_complete(self):
            return self._complete

        def mark_complete(self):
            self._complete = True

        def reset(self):
            self._complete = False

        def status_text(self):
            return "[x]" if self._complete else "[ ]"


    class TalkObjective(Objective):
        """Complete by talking to a specific character."""

        def __init__(self, description, character_id):
            super().__init__(description)
            self.character_id = character_id

        def on_talk(self, character_id):
            if character_id == self.character_id:
                self.mark_complete()


    class ItemObjective(Objective):
        """
        Complete by having a specific item in inventory.

        Assumes `store.player_inventory` exposes `.has(item_id, qty)`.
        Compatible with both the minimal Inventory (renpy-rpg.md § 4) and the
        full Inventory (renpy-inventory.md). For quest-bound items that should
        be auto-removed on completion, use `QuestItem` from renpy-inventory.md.
        """

        def __init__(self, description, item_id, qty=1):
            super().__init__(description)
            self.item_id = item_id
            self.qty     = qty

        def check(self):
            """Re-check from inventory. Call after any inventory change."""
            if store.player_inventory and store.player_inventory.has(self.item_id, self.qty):
                self.mark_complete()


    class StatObjective(Objective):
        """Complete when a stat reaches a threshold."""

        def __init__(self, description, stat_name, threshold, stat_object="player_survival"):
            super().__init__(description)
            self.stat_name   = stat_name
            self.threshold   = threshold
            self.stat_object = stat_object

        def check(self):
            obj = getattr(store, self.stat_object, None)
            if obj and getattr(obj, self.stat_name, 0) >= self.threshold:
                self.mark_complete()


    class CounterObjective(Objective):
        """Complete by doing something N times."""

        def __init__(self, description, required_count):
            super().__init__(description)
            self.required = required_count
            self.current  = 0

        def increment(self, amount=1):
            self.current = min(self.required, self.current + amount)
            if self.current >= self.required:
                self.mark_complete()

        def reset(self):
            super().reset()
            self.current = 0

        def status_text(self):
            marker = "[x]" if self._complete else "[ ]"
            return "{} {}/{}".format(marker, self.current, self.required)


    class LocationObjective(Objective):
        """Complete by visiting a specific location."""

        def __init__(self, description, location_id):
            super().__init__(description)
            self.location_id = location_id

        def on_arrive(self, location_id):
            if location_id == self.location_id:
                self.mark_complete()


    class FlagObjective(Objective):
        """Complete when a boolean flag becomes True."""

        def __init__(self, description, flag_name):
            super().__init__(description)
            self.flag_name = flag_name

        def check(self):
            if getattr(store, self.flag_name, False):
                self.mark_complete()
```

---

## 4. Quest Registry

Define all quests as data in `quest_data.rpy`, separate from logic.

```renpy
# systems/quest_data.rpy

init python:

    def _build_quests():
        return {

            # ── Introductory quest ────────────────────────────────────
            "quest_intro_settle_in": Quest(
                quest_id    = "quest_intro_settle_in",
                title       = "Settling In",
                description = "You've just moved in. Get your bearings.",
                objectives  = [
                    TalkObjective("Introduce yourself to your neighbor", "npc_neighbor"),
                    LocationObjective("Find the local store", "loc_shop"),
                    ItemObjective("Buy groceries", "item_groceries", qty=1),
                ],
                rewards     = [
                    StatReward("mood", 15),
                    MoneyReward(50),
                ],
                chain_next  = "quest_meet_the_residents",
            ),

            # ── Character-specific quest ──────────────────────────────
            "quest_meet_the_residents": Quest(
                quest_id    = "quest_meet_the_residents",
                title       = "Meet the Residents",
                description = "Get to know the people in your building.",
                objectives  = [
                    TalkObjective("Meet Sarah",  "npc_sarah"),
                    TalkObjective("Meet Monica", "npc_monica"),
                    TalkObjective("Meet Diane",  "npc_diane"),
                ],
                rewards     = [
                    StatReward("social", 20),
                ],
                unlock_condition = lambda: store.quest_manager.is_complete("quest_intro_settle_in"),
            ),

        }

    # Built at init time; reassigned to store.quest_manager
    _QUEST_DATA = _build_quests()
```

---

## 5. Quest Manager

```renpy
# systems/quest_system.rpy (continued)

init python:

    class QuestManager:
        """Central controller for all quests."""

        def __init__(self, quest_dict):
            self.quests = quest_dict   # {quest_id: Quest}

        def get(self, quest_id):
            return self.quests.get(quest_id)

        def is_active(self, quest_id):
            q = self.get(quest_id)
            return q is not None and q.state == QUEST_ACTIVE

        def is_complete(self, quest_id):
            q = self.get(quest_id)
            return q is not None and q.state == QUEST_COMPLETE

        def is_available(self, quest_id):
            q = self.get(quest_id)
            return q is not None and q.state == QUEST_AVAILABLE

        def active_quests(self):
            return [q for q in self.quests.values() if q.state == QUEST_ACTIVE]

        def available_quests(self):
            return [q for q in self.quests.values() if q.state == QUEST_AVAILABLE]

        def start(self, quest_id):
            q = self.get(quest_id)
            if q:
                result = q.start()
                if result:
                    renpy.notify("New quest: {}".format(q.title))
                return result
            return False

        def try_complete(self, quest_id):
            q = self.get(quest_id)
            if q and q.check_auto_complete():
                renpy.notify("Quest complete: {}".format(q.title))
                # Auto-unlock and start chain
                if q.chain_next:
                    next_q = self.get(q.chain_next)
                    if next_q:
                        next_q.state = QUEST_AVAILABLE
                        self.start(q.chain_next)
                return True
            return False

        def tick_unlock_conditions(self):
            """Call periodically to auto-unlock available quests."""
            for q in self.quests.values():
                q.try_unlock()

        def check_all_stat_objectives(self):
            """Re-check stat/item/flag objectives. Call after any stat change."""
            for quest in self.active_quests():
                for obj in quest.objectives:
                    if hasattr(obj, "check"):
                        obj.check()
                self.try_complete(quest.quest_id)


default quest_manager = None

init python:
    def init_quest_manager():
        store.quest_manager = QuestManager(_QUEST_DATA)
        store.quest_manager.tick_unlock_conditions()
```

---

## 6. Quest Journal Screen

```renpy
# screens/quest_journal.rpy

screen quest_journal_screen():
    modal True
    zorder 50

    frame:
        xalign 0.5 yalign 0.5
        xsize 700 ysize 500
        padding (20, 20)

        vbox:
            spacing 12

            text "Quest Journal" size 24 bold True xalign 0.5

            hbox:
                spacing 20
                xalign 0.5

                # Active quests panel
                frame:
                    xsize 300 ysize 380
                    padding (12, 10)
                    vbox:
                        spacing 8
                        text "Active Quests" size 18 bold True

                        python:
                            active = store.quest_manager.active_quests()
                        if active:
                            for quest in active:
                                vbox:
                                    spacing 4
                                    text quest.title size 15 bold True color "#f1c40f"
                                    text quest.description size 13 color "#ecf0f1"
                                    for obj in quest.objectives:
                                        text "{} {}".format(obj.status_text(), obj.description) size 12
                        else:
                            text "No active quests." size 14 color "#888"

                # Available quests panel
                frame:
                    xsize 300 ysize 380
                    padding (12, 10)
                    vbox:
                        spacing 8
                        text "Available" size 18 bold True

                        python:
                            available = store.quest_manager.available_quests()
                        if available:
                            for quest in available:
                                vbox:
                                    spacing 4
                                    text quest.title size 15 color "#2ecc71"
                                    text quest.description size 13 color "#bdc3c7"
                                    textbutton "Start Quest":
                                        action [
                                            Function(store.quest_manager.start, quest.quest_id),
                                            renpy.restart_interaction,
                                        ]
                        else:
                            text "None available." size 14 color "#888"

            textbutton "Close" action Return() xalign 0.5
```

---

## 7. Rewards & Side Effects

```renpy
# systems/quest_system.rpy (continued)

init python:

    class Reward:
        """Base reward class. Subclass for each reward type."""
        def apply(self):
            pass

    class StatReward(Reward):
        def __init__(self, stat, amount):
            self.stat   = stat
            self.amount = amount
        def apply(self):
            store.player_survival.modify(self.stat, self.amount)
            renpy.notify("+{} {}".format(self.amount, self.stat.capitalize()))

    class MoneyReward(Reward):
        def __init__(self, amount):
            self.amount = amount
        def apply(self):
            store.player_survival.modify("money", self.amount)
            renpy.notify("Received ${}".format(self.amount))

    class ItemReward(Reward):
        def __init__(self, item_id, qty=1):
            self.item_id = item_id
            self.qty     = qty
        def apply(self):
            if store.player_inventory:
                store.player_inventory.add(self.item_id, self.qty)
            renpy.notify("Received: {}".format(self.item_id))

    class RelationshipReward(Reward):
        def __init__(self, character_id, amount):
            self.character_id = character_id
            self.amount       = amount
        def apply(self):
            key = "rel_{}".format(self.character_id)
            current = getattr(store, key, 0)
            setattr(store, key, min(100, current + self.amount))
            renpy.notify("+{} relationship with {}".format(self.amount, self.character_id))

    class SceneUnlockReward(Reward):
        """Unlocks a gallery scene."""
        def __init__(self, scene_id):
            self.scene_id = scene_id
        def apply(self):
            if self.scene_id not in store.unlocked_scenes:
                store.unlocked_scenes.append(self.scene_id)
            renpy.notify("New scene unlocked!")


    def apply_quest_rewards(quest):
        """Apply all rewards for a completed quest."""
        for reward in quest.rewards:
            reward.apply()
```

**Wiring rewards to quest completion:**

```renpy
# In Quest.complete():
    def complete(self):
        if self.state == QUEST_ACTIVE:
            self.state = QUEST_COMPLETE
            apply_quest_rewards(self)   # ← add this line
            if self.on_complete:
                self.on_complete()
            return True
        return False
```

---

## 8. Quest Chains

Quests that automatically trigger the next quest on completion. Set `chain_next` in the quest definition:

```renpy
Quest(
    quest_id   = "quest_sarah_01",
    title      = "Sarah's First Request",
    ...
    chain_next = "quest_sarah_02",    # auto-unlocks when quest_sarah_01 completes
)

Quest(
    quest_id         = "quest_sarah_02",
    title            = "Sarah's Secret",
    unlock_condition = lambda: store.quest_manager.is_complete("quest_sarah_01"),
    ...
)
```

For parallel quest trees (multiple characters progressing independently):

```renpy
# Each character has their own quest chain prefix:
# quest_sarah_01 → quest_sarah_02 → quest_sarah_03 (ending)
# quest_monica_01 → quest_monica_02 → quest_monica_03
# These progress independently; use relationship values to gate each
```

---

## 9. Integration Patterns

**Trigger objective on dialogue:**

```renpy
label talk_to_sarah:
    e "Hi! You must be the new neighbor."
    python:
        # Mark talk objective complete
        q = quest_manager.get("quest_intro_settle_in")
        if q and q.state == QUEST_ACTIVE:
            for obj in q.objectives:
                if isinstance(obj, TalkObjective) and obj.character_id == "npc_sarah":
                    obj.mark_complete()
            quest_manager.try_complete("quest_intro_settle_in")

        # Trigger quest unlock check
        quest_manager.tick_unlock_conditions()
    jump hub_main
```

**Trigger stat objective after modifying stats:**

```renpy
label activity_exercise:
    python:
        player_survival.modify("fitness", +8)
        quest_manager.check_all_stat_objectives()   # re-check stat-gated objectives
    jump hub_main
```

**Conditional dialogue based on quest state:**

```renpy
label talk_to_shop_owner:
    if quest_manager.is_active("quest_intro_settle_in"):
        "The shop owner points you to the grocery section."
        python:
            # Complete the item purchase objective manually
            q = quest_manager.get("quest_intro_settle_in")
            if q:
                for obj in q.objectives:
                    if isinstance(obj, ItemObjective) and obj.item_id == "item_groceries":
                        player_inventory.add("item_groceries")
                        obj.check()
                quest_manager.try_complete("quest_intro_settle_in")
    elif quest_manager.is_complete("quest_intro_settle_in"):
        "The shop owner greets you like a regular."
    else:
        "The shop owner nods at you."
```

---

## 10. Multi-Stage Quests

For quests with **progressive stages** where each stage has its own objectives, partial rewards, and triggers a scene on completion. Common in courtship/seduction arcs, multi-step heists, or any story with discrete acts.

```renpy
# systems/quest_system.rpy (continued)

init python:

    class QuestStage:
        """A single stage of a multi-stage quest."""

        def __init__(self, stage_id, title, objectives=None,
                     partial_rewards=None, scene_label=None):
            self.stage_id        = stage_id
            self.title           = title
            self.objectives      = objectives or []
            self.partial_rewards = partial_rewards or []
            self.scene_label     = scene_label   # called when stage completes

        def is_complete(self):
            return all(obj.is_complete() for obj in self.objectives)

        def reset(self):
            for obj in self.objectives:
                obj.reset()


    class StagedQuest(Quest):
        """A Quest with sequential stages instead of a flat objective list."""

        def __init__(self, quest_id, title, description, stages,
                     rewards=None, unlock_condition=None,
                     on_complete=None, on_fail=None, chain_next=None):
            super().__init__(quest_id, title, description,
                             objectives=None, rewards=rewards,
                             unlock_condition=unlock_condition,
                             on_complete=on_complete, on_fail=on_fail,
                             chain_next=chain_next)
            self.stages       = stages   # list of QuestStage
            self.current_idx  = 0

        def current_stage(self):
            if self.state != QUEST_ACTIVE or self.current_idx >= len(self.stages):
                return None
            return self.stages[self.current_idx]

        def advance_stage(self):
            """
            Check if current stage is done. If yes: apply partial rewards,
            call scene, move to next stage. If all stages done: complete quest.
            """
            stage = self.current_stage()
            if not stage or not stage.is_complete():
                return False
            # Apply partial rewards
            for reward in stage.partial_rewards:
                reward.apply()
            # Call scene label if present
            if stage.scene_label:
                renpy.call_in_new_context(stage.scene_label)
            # Advance
            self.current_idx += 1
            if self.current_idx >= len(self.stages):
                self.complete()
            return True

        def all_objectives_met(self):
            """Override: staged quest is complete when last stage clears."""
            return self.current_idx >= len(self.stages)

        def progress_text(self):
            return "Stage {}/{}".format(min(self.current_idx + 1, len(self.stages)),
                                        len(self.stages))


    # Wire into manager.check_all_stat_objectives
    _base_check = QuestManager.check_all_stat_objectives

    def check_all_stat_objectives_staged(self):
        _base_check(self)
        for quest in self.active_quests():
            if isinstance(quest, StagedQuest):
                quest.advance_stage()

    QuestManager.check_all_stat_objectives = check_all_stat_objectives_staged
```

**Defining a staged quest**:

```renpy
StagedQuest(
    quest_id="quest_lorenzo_courtship",
    title="The Lorenzo Approach",
    description="Win Lorenzo's trust through a series of careful steps.",
    stages=[
        QuestStage("stage_gift",
                   "Send a premium gift",
                   objectives=[ItemObjective("Acquire premium perfume", "item_perfume"),
                               FlagObjective("Send the gift", "lorenzo_gift_sent")],
                   partial_rewards=[StatReward("influence", 2)],
                   scene_label="scene_lorenzo_stage_01"),
        QuestStage("stage_dinner",
                   "Arrange a private dinner",
                   objectives=[StatObjective("Have 5000 cash", "money", 5000),
                               FlagObjective("Book the venue", "lorenzo_dinner_booked")],
                   partial_rewards=[StatReward("influence", 3)],
                   scene_label="scene_lorenzo_stage_02"),
        QuestStage("stage_private",
                   "Private evening at the quinta",
                   objectives=[StatObjective("Relationship 40+", "value", 40,
                                              stat_object="characters.lorenzo")],
                   partial_rewards=[StatReward("influence", 10),
                                    SceneUnlockReward("lorenzo_unlocked")],
                   scene_label="scene_lorenzo_stage_03"),
    ],
)
```

---

## 11. Procedural Quests from Templates

For repeatable "errand" quests (run this package, find this object, photograph this person). Define templates with variables filled at runtime.

```renpy
# systems/quest_templates.rpy

init python:

    import random

    class QuestTemplate:
        """
        Template for procedurally generating quests.
        Variables in title/description (e.g., "{npc}", "{item}") are filled at generation.
        """

        def __init__(self, template_id, title_pattern, description_pattern,
                     objective_factory, reward_factory,
                     variables_spec, deadline_days=None):
            self.template_id        = template_id
            self.title_pattern      = title_pattern
            self.description_pattern= description_pattern
            self.objective_factory  = objective_factory   # callable(vars) → list[Objective]
            self.reward_factory     = reward_factory      # callable(vars) → list[Reward]
            self.variables_spec     = variables_spec      # dict of {var_name: callable() → value}
            self.deadline_days      = deadline_days


    def generate_procedural_quest(template_id):
        """Generate a new quest instance from a template. Returns the Quest or None."""
        template = QUEST_TEMPLATES.get(template_id)
        if not template:
            return None

        # Fill variables
        variables = {name: factory() for name, factory in template.variables_spec.items()}

        # Generate unique quest_id
        proc_id = "proc_{}_{}".format(template_id, store.game_time.day if store.game_time else 0)

        quest = Quest(
            quest_id    = proc_id,
            title       = template.title_pattern.format(**variables),
            description = template.description_pattern.format(**variables),
            objectives  = template.objective_factory(variables),
            rewards     = template.reward_factory(variables),
        )
        quest.state = QUEST_AVAILABLE
        quest.repeatable = True  # see Section 15
        if template.deadline_days:
            quest.deadline_day = store.game_time.day + template.deadline_days

        store.quest_manager.quests[proc_id] = quest
        return quest


    # Define templates
    QUEST_TEMPLATES = {

        "fetch_item": QuestTemplate(
            template_id="fetch_item",
            title_pattern="Bring {item}",
            description_pattern="Get {item} from {location} and bring it to {npc}.",
            objective_factory=lambda v: [
                ItemObjective("Acquire " + v["item_name"], v["item_id"]),
                TalkObjective("Deliver to " + v["npc_name"], v["npc_id"]),
            ],
            reward_factory=lambda v: [MoneyReward(v["payout"])],
            variables_spec={
                "item":       lambda: random.choice(["whiskey", "cigars", "wine"]),
                "item_name":  lambda: "premium bottle",
                "item_id":    lambda: "item_premium_bottle",
                "location":   lambda: random.choice(["the corner store", "downtown shop"]),
                "npc":        lambda: random.choice(list(NPC_STATS.keys())),
                "npc_name":   lambda: "the contact",
                "npc_id":     lambda: random.choice(list(NPC_STATS.keys())),
                "payout":     lambda: random.randint(2000, 5000),
            },
            deadline_days=3,
        ),

        "photograph": QuestTemplate(
            template_id="photograph",
            title_pattern="Discreet photo: {target}",
            description_pattern="Get a photograph of {target} at {location}.",
            objective_factory=lambda v: [
                FlagObjective("Got the photo", "flag_proc_photo_" + v["target_id"]),
                TalkObjective("Deliver to client", "npc_client"),
            ],
            reward_factory=lambda v: [MoneyReward(v["payout"])],
            variables_spec={
                "target":     lambda: random.choice(list(NPC_STATS.keys())),
                "target_id":  lambda: "tmp",
                "location":   lambda: random.choice(["the bar", "the office", "the casino"]),
                "payout":     lambda: random.randint(3000, 7000),
            },
            deadline_days=5,
        ),
    }


    def maybe_spawn_procedural_quest(probability=0.5):
        """
        Call on new day or hub entry. May spawn a new procedural quest.
        Caps the active procedural quest count so the journal doesn't flood.
        """
        proc_active = [q for q in store.quest_manager.active_quests()
                       if q.quest_id.startswith("proc_")]
        if len(proc_active) >= 3:
            return None
        if random.random() < probability:
            template_id = random.choice(list(QUEST_TEMPLATES.keys()))
            return generate_procedural_quest(template_id)
        return None
```

---

## 12. Soft Deadlines & Expiration

A quest can have a **soft deadline** — passing it doesn't cancel the quest, but degrades the reward and applies a penalty.

```renpy
# systems/quest_system.rpy

init python:

    # Add fields to Quest
    Quest.deadline_day      = None     # absolute day count; None = no deadline
    Quest.on_expire         = None     # callable(); called when deadline passes
    Quest.expired           = False
    Quest.expire_penalty_relationship = None   # (npc_id, delta) tuple, optional


    def quest_is_expired(quest):
        if quest.deadline_day is None or quest.expired:
            return quest.expired
        if store.game_time.day > quest.deadline_day:
            quest.expired = True
            # Apply penalty
            if quest.on_expire:
                quest.on_expire()
            if quest.expire_penalty_relationship:
                npc_id, delta = quest.expire_penalty_relationship
                npc_modify(npc_id, "trust", delta)
            renpy.notify("Quest deadline passed: {}".format(quest.title))
        return quest.expired


    def tick_quest_deadlines():
        """Call on new day. Marks expired quests and degrades their rewards."""
        for quest in store.quest_manager.active_quests():
            quest_is_expired(quest)


    # Halve rewards on expired quests
    _base_apply = apply_quest_rewards

    def apply_quest_rewards_with_decay(quest):
        if quest.expired:
            # Apply only half the rewards
            for reward in quest.rewards:
                if hasattr(reward, "amount"):
                    original = reward.amount
                    reward.amount = original // 2
                    reward.apply()
                    reward.amount = original
                else:
                    reward.apply()
        else:
            _base_apply(quest)

    apply_quest_rewards = apply_quest_rewards_with_decay
```

**Wire into day loop**:

```renpy
init python:
    _base_on_new_day = on_new_day
    def on_new_day():
        _base_on_new_day()
        tick_quest_deadlines()
```

**UI**: in the journal, mark deadlines visually:

```renpy
# Inside quest journal screen, for each quest:
python:
    days_left = (q.deadline_day - store.game_time.day) if q.deadline_day else None

if days_left is not None:
    if days_left < 0:
        text "EXPIRED" size 12 color "#e74c3c"
    elif days_left <= 2:
        text "Due in {} days".format(days_left) size 12 color "#f39c12"
    else:
        text "Due day {}".format(q.deadline_day) size 12 color "#888"
```

---

## 13. Quest Categories & UI Filtering

Tag quests with a category for journal filtering (e.g., a 4-tab UI: Main / Side / Procedural / Story).

```renpy
init python:

    # Add field
    Quest.category = "side"   # "main", "side", "procedural", "story"


    # Helper to filter
    def quests_by_category(category, state=None):
        out = []
        for q in store.quest_manager.quests.values():
            if q.category != category:
                continue
            if state and q.state != state:
                continue
            out.append(q)
        return out
```

**Screen pattern with tabs**:

```renpy
default selected_quest_tab = "main"

screen quest_journal_tabbed():
    modal True
    zorder 50
    frame:
        xalign 0.5 yalign 0.5
        xsize 780 ysize 540
        padding (20, 16)
        vbox:
            spacing 12
            text "Journal" size 24 bold True xalign 0.5

            # Tab bar
            hbox:
                spacing 14 xalign 0.5
                for cat in ["main", "side", "procedural", "story"]:
                    textbutton cat.capitalize():
                        action SetScreenVariable("selected_quest_tab", cat)
                        text_color ("#f1c40f" if selected_quest_tab == cat else "#888")

            # Filtered list
            python:
                visible = quests_by_category(selected_quest_tab, state=QUEST_ACTIVE)
            viewport:
                scrollbars "vertical"
                mousewheel True
                ysize 400
                vbox:
                    spacing 10
                    for quest in visible:
                        vbox:
                            spacing 4
                            text quest.title size 16 bold True
                            text quest.description size 13 color "#bbb"
                            for obj in (quest.objectives if hasattr(quest, "objectives") else []):
                                text "  {} {}".format(obj.status_text(), obj.description) size 12

            textbutton "Close" action Return() xalign 0.5
```

---

## 14. Quest Cancellation

Player can voluntarily abandon a quest (with penalty). Useful for arcs where the player decides "this isn't worth it anymore".

```renpy
init python:

    # Extend Quest
    Quest.on_cancel       = None     # callable(); custom penalty
    Quest.cancellable     = False    # default: not cancellable

    QUEST_CANCELLED = "cancelled"

    def cancel_quest(quest_id, default_penalty_callback=None):
        q = store.quest_manager.get(quest_id)
        if not q or not q.cancellable or q.state != QUEST_ACTIVE:
            return False
        q.state = QUEST_CANCELLED
        if q.on_cancel:
            q.on_cancel()
        elif default_penalty_callback:
            default_penalty_callback()
        renpy.notify("Quest cancelled: {}".format(q.title))
        return True
```

**Player UI**:

```renpy
# In the journal entry for a cancellable quest
if quest.cancellable and quest.state == QUEST_ACTIVE:
    textbutton "Abandon quest":
        action [
            Show("confirm", message="Abandon this quest? Penalties may apply.",
                  yes_action=[Function(cancel_quest, quest.quest_id), Hide("confirm"), renpy.restart_interaction],
                  no_action=Hide("confirm")),
        ]
```

---

## 15. Repeatable Quests

For procedural errands or daily tasks. Once completed, the quest can be reset and re-offered.

```renpy
init python:

    Quest.repeatable        = False
    Quest.completion_count  = 0
    Quest.last_completed_day = None


    def reset_repeatable_quest(quest):
        """Reset a completed repeatable quest so it can be picked up again."""
        if not quest.repeatable:
            return False
        if quest.state != QUEST_COMPLETE:
            return False
        quest.last_completed_day = store.game_time.day
        quest.completion_count += 1
        quest.state = QUEST_HIDDEN
        # Reset all objectives
        if hasattr(quest, "objectives"):
            for obj in quest.objectives:
                obj.reset()
        if hasattr(quest, "stages"):
            quest.current_idx = 0
            for stage in quest.stages:
                stage.reset()
        return True


    def tick_repeatable_quests(cooldown_days=2):
        """Periodically reset completed repeatables that have cooled down."""
        for q in store.quest_manager.quests.values():
            if (q.repeatable
                and q.state == QUEST_COMPLETE
                and q.last_completed_day is not None
                and store.game_time.day - q.last_completed_day >= cooldown_days):
                reset_repeatable_quest(q)
                q.try_unlock()
```

Wire `tick_repeatable_quests()` into the new-day hook alongside `tick_quest_deadlines()`.
