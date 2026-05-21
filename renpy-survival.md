# Ren'Py Survival Stats & Time Cycle Reference

Use this when the project has survival mechanics: energy/money management, daily stat decay, action economy, and time-gated events. Common in sandbox/free-roam adult VNs and life-sim games.

---

## Table of Contents
1. [Architecture](#1-architecture)
2. [Survival Stats Class](#2-survival-stats-class)
3. [Time & Day Cycle](#3-time--day-cycle)
4. [Action Economy](#4-action-economy)
5. [Stat Decay](#5-stat-decay)
6. [Game-Over & Fail Conditions](#6-game-over--fail-conditions)
7. [Survival HUD](#7-survival-hud)
8. [Integration Example](#8-integration-example)
9. [Variant: One Action Per Slot](#9-variant-one-action-per-slot)
10. [Energy as Modifier (Not Limiter)](#10-energy-as-modifier-not-limiter)
11. [Scheduled Bills (Recurring Fixed Costs)](#11-scheduled-bills-recurring-fixed-costs)
12. [Wage with Attendance Tracking](#12-wage-with-attendance-tracking)
13. [Day-of-Week Gating](#13-day-of-week-gating)
14. [Debt Ramp with Narrative Consequences](#14-debt-ramp-with-narrative-consequences)
15. [Skip Slot / Sleep Early](#15-skip-slot--sleep-early)

---

## 1. Architecture

```
systems/
├── survival_system.rpy   # SurvivalStats class, decay logic
├── time_system.rpy       # TimeManager: day, period, week
└── flags.rpy             # default survival + time variables

screens/
├── survival_hud.rpy      # Always-visible stat bars
└── daily_summary.rpy     # End-of-day recap screen
```

**Design principle:** Survival stats are separate from RPG combat stats. They represent the player's daily life state — not battle power. Keep them in their own class and file.

---

## 2. Survival Stats Class

```renpy
# systems/survival_system.rpy

init python:

    class SurvivalStats:
        """
        Tracks the player's daily life stats.
        All stats are 0–100 unless noted otherwise.
        Money has no upper cap.
        """

        STAT_NAMES = ["energy", "mood", "fitness", "social", "money"]

        def __init__(self):
            self.energy   = 80    # Depletes from actions; restored by sleep
            self.mood     = 60    # Depletes from stress; boosted by fun activities
            self.fitness  = 40    # Increases from exercise; slow passive decay
            self.social   = 50    # Depletes from isolation; raised by interactions
            self.money    = 200   # Currency; no upper cap; can go negative (debt)

            # Derived / computed
            self._debt_warning = False

        # ── Clamped setter helpers ─────────────────────────────────────
        def _clamp(self, value, lo=0, hi=100):
            return max(lo, min(hi, value))

        def modify(self, stat, delta):
            """
            Apply a delta to a stat. Money is uncapped.
            Returns the actual change applied.
            """
            if stat == "money":
                old = self.money
                self.money += delta
                return self.money - old
            current = getattr(self, stat, None)
            if current is None:
                return 0
            new_val = self._clamp(current + delta)
            setattr(self, stat, new_val)
            return new_val - current

        def is_exhausted(self):
            return self.energy <= 0

        def is_broke(self):
            return self.money <= 0

        def is_depressed(self):
            return self.mood <= 10

        def critical_stats(self):
            """Returns list of stat names currently in critical range."""
            critical = []
            if self.energy  <= 15: critical.append("energy")
            if self.mood    <= 15: critical.append("mood")
            if self.fitness <= 10: critical.append("fitness")
            if self.social  <= 10: critical.append("social")
            if self.money   <= 0:  critical.append("money")
            return critical

        def end_of_day_decay(self):
            """Apply passive daily stat decay. Call once per in-game day."""
            self.modify("fitness", -2)   # fitness decays slowly without exercise
            self.modify("social",  -5)   # social decays if you don't interact
            self.modify("mood",    -3)   # mood drifts down without upkeep


default player_survival = None

init python:
    def init_survival_stats():
        store.player_survival = SurvivalStats()
```

---

## 3. Time & Day Cycle

```renpy
# systems/time_system.rpy

init python:

    # Time periods in order — defines the flow of a day
    TIME_PERIODS = ["morning", "afternoon", "evening", "night"]

    DAYS_OF_WEEK = ["Monday", "Tuesday", "Wednesday", "Thursday",
                    "Friday", "Saturday", "Sunday"]

    class TimeManager:
        """Manages in-game time: period of day, day number, day of week."""

        def __init__(self):
            self.day        = 1                # Absolute day count (Day 1, 2, ...)
            self.period_idx = 0                # Index into TIME_PERIODS
            self.week       = 1

        @property
        def period(self):
            return TIME_PERIODS[self.period_idx]

        @property
        def day_name(self):
            return DAYS_OF_WEEK[(self.day - 1) % 7]

        @property
        def is_weekend(self):
            return self.day_name in ("Saturday", "Sunday")

        def advance(self):
            """
            Move to the next time period.
            Returns True if a new day started.
            """
            self.period_idx += 1
            if self.period_idx >= len(TIME_PERIODS):
                self.period_idx = 0
                self.day += 1
                if (self.day - 1) % 7 == 0:
                    self.week += 1
                return True   # new day
            return False

        def skip_to(self, target_period):
            """Advance until reaching target_period. Returns number of periods skipped."""
            skipped = 0
            while self.period != target_period:
                self.advance()
                skipped += 1
                if skipped > len(TIME_PERIODS):
                    break   # safety guard
            return skipped

        def label_for_period(self):
            """Returns a display string for the current time."""
            return "Day {} ({}) — {}".format(
                self.day, self.day_name, self.period.capitalize()
            )


default game_time = None
default actions_remaining = 3   # actions available per time period

init python:
    def init_time():
        store.game_time = TimeManager()
        store.actions_remaining = 3
```

**Display macro (for story files):**

```renpy
# Show current time in narration
"[game_time.label_for_period()]"
# → "Day 3 (Wednesday) — Afternoon"
```

---

## 4. Action Economy

Each time period grants a fixed number of actions. Spending all actions advances to the next period.

```renpy
# systems/time_system.rpy (continued)

init python:

    ACTIONS_PER_PERIOD = {
        "morning":   3,
        "afternoon": 3,
        "evening":   2,
        "night":     1,   # limited — sleep is expected
    }

    def spend_action(count=1):
        """
        Spend N actions. If actions run out, advance time period.
        Returns True if a new day started.
        """
        new_day = False
        store.actions_remaining -= count
        while store.actions_remaining <= 0:
            overflow = store.actions_remaining   # negative = owed to next period
            new_day = store.game_time.advance() or new_day
            next_period = store.game_time.period
            store.actions_remaining = ACTIONS_PER_PERIOD.get(next_period, 3) + overflow
            if new_day:
                on_new_day()
        return new_day

    def on_new_day():
        """Hook called at the start of each new day."""
        store.player_survival.end_of_day_decay()
        renpy.notify("A new day begins: {}".format(store.game_time.label_for_period()))
        # Add daily income, trigger scheduled events, etc.
```

**Using actions in story labels:**

```renpy
label activity_exercise:
    "You head to the gym and work out."
    python:
        player_survival.modify("fitness", +8)
        player_survival.modify("energy",  -15)
        player_survival.modify("mood",    +5)
        new_day = spend_action(1)

    "You feel the burn. Fitness is improving."

    if new_day:
        call end_of_day_summary
    jump hub_main
```

---

## 5. Stat Decay

Decay keeps the survival loop active — players must maintain stats, not just accumulate them.

```renpy
# systems/survival_system.rpy

init python:

    # Decay rates per time period (applied when period advances)
    PERIOD_DECAY = {
        "morning":   {"energy": -5,  "mood": -3},
        "afternoon": {"energy": -8,  "mood": -5},
        "evening":   {"energy": -5,  "mood": -3},
        "night":     {"energy": -3},
    }

    def apply_period_decay(period):
        """Apply stat decay for the given period. Call when period ends."""
        decays = PERIOD_DECAY.get(period, {})
        for stat, delta in decays.items():
            player_survival.modify(stat, delta)

    # Sleep restores energy — call when player sleeps
    def do_sleep():
        player_survival.modify("energy", +70)
        player_survival.modify("mood",   +10)
```

**Integrating decay into time advancement:**

```renpy
# Extend spend_action to call decay:
init python:
    _original_spend = spend_action

    def spend_action(count=1):
        old_period = store.game_time.period
        new_day = _original_spend(count)
        # Decay only if the period actually changed
        if store.game_time.period != old_period:
            apply_period_decay(old_period)
        return new_day
```

---

## 6. Game-Over & Fail Conditions

```renpy
# systems/survival_system.rpy

init python:

    def check_survival_fail():
        """
        Returns a fail reason string, or None if the player is OK.
        Call at the start of each period or after major actions.
        """
        s = store.player_survival
        if s.energy <= 0:
            return "exhaustion"
        if s.mood <= 0:
            return "despair"
        if s.money <= -500:
            return "bankruptcy"
        return None
```

```renpy
# In the hub or period transition:

label check_player_state:
    python:
        fail_reason = check_survival_fail()

    if fail_reason == "exhaustion":
        "You collapse from exhaustion..."
        call label_game_over_exhaustion
    elif fail_reason == "despair":
        "The weight of it all becomes too much."
        call label_game_over_despair
    elif fail_reason == "bankruptcy":
        "The debt collectors come knocking."
        call label_game_over_bankruptcy
    return

label label_game_over_exhaustion:
    scene black with fade
    "GAME OVER — You burned out."
    "Try balancing work with rest."
    $ renpy.call_in_new_context("_game_menu")   # or jump to main menu
    return
```

**Soft-fail patterns (preferred over hard game-overs):**

```renpy
# Instead of instant game-over, reduce stats further and restrict options
label low_energy_warning:
    if player_survival.energy <= 20:
        "[player_name] is exhausted. Some activities aren't available."
        # Disable energy-costing actions in menus via conditionals
    return
```

---

## 7. Survival HUD

```renpy
# screens/survival_hud.rpy

screen survival_hud():
    zorder 10
    style_prefix "survival_hud"

    frame:
        xalign 0.0 yalign 0.0
        xoffset 15 yoffset 15
        padding (12, 10)
        vbox:
            spacing 6
            xsize 220

            # Day / period header
            text "[game_time.label_for_period()]" size 14 bold True

            # Stat bars
            hbox:
                spacing 6
                text "Energy" size 13 xsize 60
                bar value player_survival.energy range 100 xsize 120 ysize 12
                text "[player_survival.energy]" size 12

            hbox:
                spacing 6
                text "Mood" size 13 xsize 60
                bar value player_survival.mood range 100 xsize 120 ysize 12
                text "[player_survival.mood]" size 12

            hbox:
                spacing 6
                text "Fitness" size 13 xsize 60
                bar value player_survival.fitness range 100 xsize 120 ysize 12
                text "[player_survival.fitness]" size 12

            hbox:
                spacing 6
                text "Social" size 13 xsize 60
                bar value player_survival.social range 100 xsize 120 ysize 12
                text "[player_survival.social]" size 12

            text "$ [player_survival.money]" size 14 bold True color "#2ecc71"

            # Actions left
            text "Actions: [actions_remaining]" size 13 color "#f39c12"

    # Critical stat warning
    python:
        crits = player_survival.critical_stats()
    if crits:
        frame:
            xalign 0.5 yalign 0.0 yoffset 10
            background "#c0392b"
            padding (10, 6)
            text "Warning: [', '.join(crits)] critical!" size 15 color "#ffffff"
```

---

## 8. Integration Example

Full day loop connecting time, survival, and story:

```renpy
# systems/flags.rpy additions
default hub_visited_today = False

label start_of_day:
    $ hub_visited_today = False
    show screen survival_hud
    call check_player_state
    jump hub_main

label hub_main:
    scene bg_apartment_living_room with dissolve
    "Day [game_time.day] — [game_time.period.capitalize()]. What will you do?"

    menu:
        "Go to the gym" if player_survival.energy >= 15:
            jump activity_exercise
        "Go to work" if player_survival.energy >= 20:
            jump activity_work
        "Talk to [npc_name]" if npc_is_available("npc_name"):
            jump talk_to_npc
        "Rest / Sleep" if game_time.period == "night":
            jump activity_sleep
        "Check phone / Journal":
            call screen quest_journal_screen
            jump hub_main

label activity_work:
    "You head to work and put in a full shift."
    python:
        player_survival.modify("money",  +80)
        player_survival.modify("energy", -20)
        player_survival.modify("mood",   -10)
        spend_action(1)
    "You earned $80. Tiring, but necessary."
    jump hub_main

label activity_sleep:
    scene bg_bedroom with dissolve
    "You settle in for the night."
    python:
        do_sleep()
        game_time.skip_to("morning")
        store.actions_remaining = ACTIONS_PER_PERIOD["morning"]
        on_new_day()
    "A new day begins."
    jump start_of_day
```

---

## 9. Variant: One Action Per Slot

The base `TimeManager` allows multiple actions per period. For games that prefer a **strict 1-action-per-period** model (e.g., a 3-slot day: morning / afternoon / evening), simplify the action economy:

```renpy
# systems/time_system.rpy — alternate config

init python:

    # Override default ACTIONS_PER_PERIOD for a 1-per-slot model
    ACTIONS_PER_PERIOD = {
        "morning":   1,
        "afternoon": 1,
        "evening":   1,
    }

    # Only three periods total
    TIME_PERIODS = ["morning", "afternoon", "evening"]


    def spend_slot():
        """Convenience: advance one period (= use the slot). Returns True if new day."""
        return spend_action(1)
```

**In story labels** — every meaningful activity ends with `spend_slot()`:

```renpy
label activity_go_to_casino:
    scene bg_casino with dissolve
    "You head to the casino."
    # ... blackjack session ...
    $ spend_slot()
    jump hub_main
```

**Trivial actions** (check phone, look at map, talk briefly) don't call `spend_slot()` — they're free.

---

## 10. Energy as Modifier (Not Limiter)

Default pattern: energy gates actions (`if energy >= 15: do_thing`). An alternative is **energy as a quality modifier** — actions are always available, but low energy degrades the outcome (worse minigame windows, fewer dialogue options, NPC reactions).

```renpy
# systems/survival_system.rpy — energy modifier helper

init python:

    def energy_tier():
        """Categorize current energy into tiers for modifier lookup."""
        e = store.player_survival.energy
        if e >= 70:
            return "high"
        if e >= 30:
            return "mid"
        return "low"

    ENERGY_MODIFIERS = {
        "high": {"minigame_window_scale": 1.10, "stat_gain_bonus": +1,  "dialog_options": "all"},
        "mid":  {"minigame_window_scale": 1.00, "stat_gain_bonus": 0,   "dialog_options": "all"},
        "low":  {"minigame_window_scale": 0.80, "stat_gain_bonus": -1,  "dialog_options": "limited"},
    }

    def energy_modifier(key, default=None):
        """Fetch a current modifier value."""
        return ENERGY_MODIFIERS.get(energy_tier(), {}).get(key, default)
```

**In a minigame or stat-modifying action**:

```renpy
label minigame_blackjack_play:
    python:
        win_window_scale = energy_modifier("minigame_window_scale", 1.0)
        # Apply scale to the minigame's hit window or success threshold
        # ...

label talk_to_lorenzo:
    if energy_modifier("dialog_options") == "limited":
        "You're too drained to handle a charm offensive."
        # Fewer choice options available
        menu:
            "Just listen": jump talk_neutral
            "Leave": jump hub_main
    else:
        # Full menu
        menu:
            "Flatter him": jump talk_flatter
            "Negotiate hard": jump talk_negotiate
            "Just listen": jump talk_neutral
            "Leave": jump hub_main
```

**Energy depletion still happens** (per-action) but the gating shifts from "action available?" to "quality of outcome?".

---

## 11. Scheduled Bills (Recurring Fixed Costs)

For games with rent, subscriptions, weekly groceries — costs that auto-deduct on specific in-game days.

```renpy
# systems/billing_system.rpy

init python:

    class ScheduledBill:
        """A recurring fixed cost charged on a schedule."""

        def __init__(self, bill_id, name, amount, cadence="weekly", weekday=0):
            self.bill_id = bill_id
            self.name    = name
            self.amount  = amount
            self.cadence = cadence       # "weekly" or "monthly"
            self.weekday = weekday       # for weekly: 0=Mon, 6=Sun; for monthly: day of month

        def is_due_today(self, time_manager):
            if self.cadence == "weekly":
                return (time_manager.day - 1) % 7 == self.weekday
            if self.cadence == "monthly":
                return (time_manager.day - 1) % 30 == self.weekday
            return False


    SCHEDULED_BILLS = [
        ScheduledBill("rent",     "Rent",       8000, "weekly", weekday=0),    # Monday
        ScheduledBill("utilities","Utilities",  2000, "weekly", weekday=0),
        ScheduledBill("groceries","Groceries",  3000, "weekly", weekday=0),
    ]


    def charge_scheduled_bills(time_manager, survival):
        """Charge all bills due today. Returns list of (bill, paid_bool)."""
        results = []
        for bill in SCHEDULED_BILLS:
            if bill.is_due_today(time_manager):
                survival.modify("money", -bill.amount)
                results.append((bill, True))
                renpy.notify("{} charged: -${}".format(bill.name, bill.amount))
        return results
```

**Wire into `on_new_day` hook**:

```renpy
init python:
    _base_on_new_day = on_new_day

    def on_new_day():
        _base_on_new_day()
        charge_scheduled_bills(store.game_time, store.player_survival)
```

For **mudanza / property change**: replace the `SCHEDULED_BILLS` list with a higher-tier set when the player moves to a more expensive home. Helper:

```renpy
def set_residence_tier(tier):
    """tier: 'default', 'mid', 'high'. Updates rent/utility schedule."""
    global SCHEDULED_BILLS
    SCHEDULED_BILLS = RESIDENCE_BILLS[tier]
```

---

## 12. Wage with Attendance Tracking

A job that pays per attended slot, with consequences for skipping work:

```renpy
# systems/job_system.rpy

init python:

    class JobAttendance:
        """Tracks attendance, payments, warnings, and termination."""

        def __init__(self, slot_pay=3000, required_slots=("morning",),
                     consecutive_miss_warning=3, consecutive_miss_fire=5):
            self.slot_pay = slot_pay
            self.required_slots = required_slots
            self.consecutive_miss_warning = consecutive_miss_warning
            self.consecutive_miss_fire = consecutive_miss_fire
            self.consecutive_misses = 0
            self.terminated = False

        def attend(self, survival):
            """Player attended a required slot. Pay and reset miss counter."""
            if self.terminated:
                return 0
            survival.modify("money", self.slot_pay)
            self.consecutive_misses = 0
            return self.slot_pay

        def miss(self):
            """Player skipped a required slot. Returns warning/fire state."""
            if self.terminated:
                return "fired"
            self.consecutive_misses += 1
            if self.consecutive_misses >= self.consecutive_miss_fire:
                self.terminated = True
                return "fired"
            if self.consecutive_misses >= self.consecutive_miss_warning:
                return "warning"
            return "ok"


default player_job = None

init python:
    def init_job():
        store.player_job = JobAttendance()
```

**Hook into the day cycle** — at the end of a required-slot period, check if the player attended:

```renpy
init python:
    def check_attendance_at_period_end(period_just_ended):
        if not store.player_job or store.player_job.terminated:
            return
        if period_just_ended in store.player_job.required_slots:
            # Did the player actually go to work this slot?
            if not getattr(store, "attended_this_slot", False):
                state = store.player_job.miss()
                if state == "warning":
                    renpy.notify("Boss is angry. Don't skip again.")
                elif state == "fired":
                    renpy.notify("You've been fired.")
            store.attended_this_slot = False


# In activity_work label:
label activity_work:
    "You head to the office..."
    python:
        store.player_job.attend(store.player_survival)
        store.attended_this_slot = True
        spend_slot()
    jump hub_main
```

**Recovering the job**: require Influence/connections (cross-skill with reputation / political stats) to get rehired.

---

## 13. Day-of-Week Gating

Locations or NPCs that are only available on certain weekdays.

```renpy
# systems/time_system.rpy — additional helper

init python:

    def is_day_in(*allowed_days):
        """Check if today is one of the allowed weekday names."""
        return store.game_time.day_name in allowed_days


    # Use in HubLocation.unlock_condition or directly in menus
```

**Example — locations open only certain days**:

```renpy
# Casino premium opens Friday/Saturday/Sunday only
HUB_LOCATIONS["casino_premium"].unlock_condition = lambda: is_day_in("Friday", "Saturday", "Sunday")

# Congress in session only Mon-Fri
HUB_LOCATIONS["congress"].unlock_condition = lambda: not store.game_time.is_weekend


# Or inline in menu
label hub_main:
    menu:
        "Go to casino premium" if is_day_in("Friday", "Saturday", "Sunday"):
            jump activity_casino_premium
        "Go to congress" if not store.game_time.is_weekend:
            jump activity_congress
```

---

## 14. Debt Ramp with Narrative Consequences

Instead of a hard `money <= -500 → GAME_OVER`, model debt as **escalating narrative tiers** that the player can recover from:

```renpy
# systems/debt_system.rpy

init python:

    DEBT_TIERS = [
        # (debt_threshold_negative, tier_id, label_to_trigger)
        (-150000, "fatal",        "event_debt_fatal"),         # game over branch
        (-75000,  "mafia",        "event_debt_mafia"),
        (-25000,  "loan_sharks",  "event_debt_loan_sharks"),
        (-5000,   "warning",      "event_debt_warning"),
        (0,       "clean",        None),
    ]


    def current_debt_tier(survival):
        for threshold, tier_id, label in DEBT_TIERS:
            if survival.money <= threshold:
                return (tier_id, label)
        return ("clean", None)


    def check_debt_progression(survival):
        """
        Call after every money modification.
        Triggers narrative events when crossing into a new debt tier.
        Persists the highest tier reached for narrative state.
        """
        tier, label = current_debt_tier(survival)
        last_tier = getattr(store, "_last_debt_tier", "clean")
        if tier != last_tier:
            store._last_debt_tier = tier
            if label and tier != "clean":
                renpy.call_in_new_context(label)
```

**Trigger labels** define escalating narrative consequences:

```renpy
label event_debt_warning:
    "A letter from the landlord. Late fees if you don't catch up by Friday."
    return

label event_debt_loan_sharks:
    scene bg_apartment with dissolve
    "A man in a track suit is at your door."
    "\"You and I need to chat about that money you owe.\""
    # Offer them a loan (with weekly interest) or refuse
    menu:
        "Accept the loan (high interest)":
            python:
                store.player_survival.modify("money", +10000)
                store.has_loan_shark_debt = True
            "He hands you cash. \"See you next week.\""
        "Refuse":
            "\"Wrong answer.\""
            $ store.player_survival.modify("energy", -30)
            "You wake up sore the next morning."
    return

label event_debt_mafia:
    # Forced events: beatings, asset seizure, threats to family
    "Things have escalated. They came for Vicky's car."
    return

label event_debt_fatal:
    # Game over cinematic
    scene black with fade
    "They came at midnight. You don't get up."
    $ renpy.full_restart()
    return
```

**Wire into modify**:

```renpy
init python:
    _base_modify = SurvivalStats.modify

    def survival_modify_with_debt_check(self, stat, delta):
        result = _base_modify(self, stat, delta)
        if stat == "money":
            check_debt_progression(self)
        return result

    SurvivalStats.modify = survival_modify_with_debt_check
```

---

## 15. Skip Slot / Sleep Early

Let the player voluntarily end a slot or even the whole day without spending an action on a meaningful activity.

```renpy
init python:

    def skip_current_slot():
        """End the current slot voluntarily. Advances time without an activity."""
        store.actions_remaining = 0
        return spend_action(0)   # forces period advancement
```

**In hub**:

```renpy
label hub_main:
    menu:
        # ... regular hub options ...
        "Skip this slot (do nothing)":
            $ skip_current_slot()
            jump hub_main
        "Sleep until tomorrow" if game_time.period != "morning":
            jump activity_sleep
```

**Trade-off**: skipping slots loses opportunity income but may be necessary when energy is low, money is too tight to act, or all available activities require unmet stats.
