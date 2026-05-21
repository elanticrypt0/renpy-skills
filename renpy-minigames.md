# Ren'Py Minigames Reference

Use this when the project includes embedded minigames: puzzles, rhythm segments, card games, typing challenges, etc.

---

## Table of Contents
1. [Minigame Architecture Principle](#1-minigame-architecture-principle)
2. [Pattern: Screen-Based Minigame](#2-pattern-screen-based-minigame)
3. [Timed Reaction / Button Prompt](#3-timed-reaction--button-prompt)
4. [Puzzle Grid (Match/Slide)](#4-puzzle-grid-matchslide)
5. [Card / Hand Game](#5-card--hand-game)
6. [Passing Results Back to Story](#6-passing-results-back-to-story)
7. [Difficulty & Balancing Hooks](#7-difficulty--balancing-hooks)
8. [Universal Continuous Difficulty (0.0–1.0)](#8-universal-continuous-difficulty-0010)
9. [MinigameResult Dataclass](#9-minigameresult-dataclass)
10. [Blackjack Pattern (Complete)](#10-blackjack-pattern-complete)
11. [Memory Pattern (Timed Solitaire)](#11-memory-pattern-timed-solitaire)
12. [Minigame Session Economy (Buy-in + Caps)](#12-minigame-session-economy-buy-in--caps)
13. [Stat Modifiers During Minigame](#13-stat-modifiers-during-minigame)
14. [Narrative Event Interrupts](#14-narrative-event-interrupts)

---

## 1. Minigame Architecture Principle

Every minigame follows the same three-layer pattern:

```
Layer 1: State Class (Python)
    → All game logic. No display. Pure data manipulation.
    → Initialized before showing the screen.

Layer 2: Screen (Screen Language)
    → Renders the state. Calls state methods on interaction.
    → Uses renpy.restart_interaction() to update display.

Layer 3: Story Integration (Labels)
    → Calls screen, reads _return, routes to win/lose labels.
    → Gives rewards, narrates outcome.
```

**Never mix display logic and game logic.** State classes should be fully testable with no Ren'Py-specific calls.

---

## 2. Pattern: Screen-Based Minigame

This template works for most minigames. Copy and extend it.

```renpy
# systems/minigame_base.rpy

init python:

    class MinigameState:
        """Base class for minigame state objects."""

        def __init__(self):
            self.over    = False
            self.victory = False
            self.score   = 0

        def is_over(self):
            return self.over

        def win(self):
            self.over    = True
            self.victory = True

        def lose(self):
            self.over    = True
            self.victory = False
```

---

## 3. Timed Reaction / Button Prompt

A common "Quick Time Event" style minigame: press the correct button before time runs out.

```renpy
# systems/minigame_qte.rpy

init python:

    import random

    class QTEState(MinigameState):
        """Quick Time Event — press the right button in time."""

        ACTIONS = ["left", "right", "up", "down"]

        def __init__(self, rounds=5, time_limit=2.0):
            super().__init__()
            self.rounds      = rounds
            self.time_limit  = time_limit
            self.current     = 0           # round index
            self.sequence    = [random.choice(self.ACTIONS) for _ in range(rounds)]
            self.successes   = 0
            self.time_left   = time_limit

        def current_prompt(self):
            if self.current < len(self.sequence):
                return self.sequence[self.current]
            return None

        def attempt(self, action):
            if self.is_over():
                return
            if action == self.sequence[self.current]:
                self.successes += 1
                self.score += 10
            self.current += 1
            self.time_left = self.time_limit
            if self.current >= self.rounds:
                if self.successes >= (self.rounds // 2 + 1):
                    self.win()
                else:
                    self.lose()

        def tick(self, dt):
            """Call from screen timer; dt in seconds."""
            if self.is_over():
                return
            self.time_left -= dt
            if self.time_left <= 0:
                self.attempt("__timeout__")   # always wrong, triggers next round


default _qte_state = None

screen qte_screen(state):
    modal True
    zorder 60
    key "game_key_[state.current_prompt()]" action Function(state.attempt, state.current_prompt())

    frame:
        xalign 0.5 yalign 0.4
        vbox:
            spacing 16
            xalign 0.5

            text "Round [state.current + 1] / [state.rounds]" xalign 0.5 size 22

            text "Press: [state.current_prompt() or '...']":
                xalign 0.5 size 36 bold True color "#f1c40f"

            bar value AnimatedValue(state.time_left, state.time_limit, 0.05):
                xsize 300 ysize 16 xalign 0.5

            text "Score: [state.score]" xalign 0.5 size 18

    # Auto-advance timer using a repeating timer action
    timer 0.05 repeat True action [
        Function(state.tick, 0.05),
        If(state.is_over(), Return(state.victory), renpy.restart_interaction),
    ]

    hbox:
        xalign 0.5 yalign 0.9 spacing 20
        for action_name in ["left", "right", "up", "down"]:
            textbutton action_name.capitalize():
                action Function(state.attempt, action_name)
                sensitive not state.is_over()
```

**Story integration:**

```renpy
label minigame_escape_qte:
    e "Quick — dodge the attacks!"

    python:
        store._qte_state = QTEState(rounds=6, time_limit=1.8)

    call screen qte_screen(_qte_state)
    $ qte_result = _return

    if qte_result:
        e happy "Impressive! You escaped unscathed."
        jump forest_path_clear
    else:
        e worried "You took a hit..."
        $ player_stats.take_damage(20)
        jump forest_path_injured
```

---

## 4. Puzzle Grid (Match/Slide)

A tile-matching or sliding puzzle.

```renpy
# systems/minigame_puzzle.rpy

init python:

    import random

    class GridPuzzleState(MinigameState):
        """Simple match-pairs memory puzzle on a grid."""

        def __init__(self, cols=4, rows=4):
            super().__init__()
            self.cols        = cols
            self.rows        = rows
            n_pairs          = (cols * rows) // 2
            values           = list(range(n_pairs)) * 2
            random.shuffle(values)
            self.grid        = [values[i*cols:(i+1)*cols] for i in range(rows)]
            self.revealed    = [[False]*cols for _ in range(rows)]
            self.matched     = [[False]*cols for _ in range(rows)]
            self._selected   = []    # list of (row, col) currently face-up, max 2
            self.moves       = 0

        def select(self, row, col):
            if self.matched[row][col] or (row, col) in self._selected:
                return
            if len(self._selected) >= 2:
                # Hide previous unmatched pair
                for r, c in self._selected:
                    if not self.matched[r][c]:
                        self.revealed[r][c] = False
                self._selected = []

            self.revealed[row][col] = True
            self._selected.append((row, col))

            if len(self._selected) == 2:
                self.moves += 1
                r1, c1 = self._selected[0]
                r2, c2 = self._selected[1]
                if self.grid[r1][c1] == self.grid[r2][c2]:
                    self.matched[r1][c1] = True
                    self.matched[r2][c2] = True
                    self._selected = []
                    self.score += 10
                    if all(self.matched[r][c] for r in range(self.rows) for c in range(self.cols)):
                        self.win()

        def is_face_up(self, row, col):
            return self.revealed[row][col] or self.matched[row][col]

        def value_at(self, row, col):
            return self.grid[row][col]


default _puzzle_state = None

screen grid_puzzle_screen(state):
    modal True
    zorder 60

    frame:
        xalign 0.5 yalign 0.5
        vbox:
            spacing 8
            xalign 0.5

            text "Match all pairs! Moves: [state.moves]" xalign 0.5 size 20

            for row in range(state.rows):
                hbox:
                    spacing 6
                    xalign 0.5
                    for col in range(state.cols):
                        if state.is_face_up(row, col):
                            frame:
                                xsize 80 ysize 80
                                background "#2ecc71" if state.matched[row][col] else "#3498db"
                                text str(state.value_at(row, col)):
                                    xalign 0.5 yalign 0.5 size 28 bold True
                        else:
                            imagebutton:
                                idle Frame("#7f8c8d", 4, 4)
                                xsize 80 ysize 80
                                action [Function(state.select, row, col), renpy.restart_interaction]
                                sensitive not state.is_over()

            if state.is_over():
                textbutton "Continue" action Return(state.victory) xalign 0.5
```

---

## 5. Card / Hand Game

```renpy
# systems/minigame_cards.rpy

init python:

    import random

    class Card:
        def __init__(self, suit, value):
            self.suit  = suit
            self.value = value   # 1–13

        def display_name(self):
            faces = {1: "A", 11: "J", 12: "Q", 13: "K"}
            v = faces.get(self.value, str(self.value))
            return "{} {}".format(v, self.suit)

        def numeric_value(self):
            return min(self.value, 10)   # Blackjack-style cap


    class Deck:
        SUITS = ["♠", "♥", "♦", "♣"]

        def __init__(self):
            self.cards = [Card(s, v) for s in self.SUITS for v in range(1, 14)]
            random.shuffle(self.cards)

        def draw(self):
            return self.cards.pop() if self.cards else None


    class HighCardState(MinigameState):
        """Simple higher-card wins minigame."""

        def __init__(self):
            super().__init__()
            deck            = Deck()
            self.player_card = deck.draw()
            self.enemy_card  = deck.draw()
            self.revealed    = False

        def reveal(self):
            self.revealed = True
            pv = self.player_card.numeric_value()
            ev = self.enemy_card.numeric_value()
            if pv > ev:
                self.win()
                self.score = 100
            elif pv < ev:
                self.lose()
            else:
                # Tie — treat as win (house rules)
                self.win()
                self.score = 50


default _card_state = None

screen high_card_screen(state):
    modal True
    zorder 60

    frame:
        xalign 0.5 yalign 0.5
        vbox:
            spacing 20
            xalign 0.5
            text "High Card Challenge" xalign 0.5 size 24 bold True

            hbox:
                spacing 40
                xalign 0.5
                vbox:
                    text "You" xalign 0.5 size 18
                    frame:
                        xsize 100 ysize 140
                        background "#ecf0f1"
                        text state.player_card.display_name() xalign 0.5 yalign 0.5 size 22
                vbox:
                    text "Opponent" xalign 0.5 size 18
                    frame:
                        xsize 100 ysize 140
                        background "#ecf0f1" if state.revealed else "#2c3e50"
                        text (state.enemy_card.display_name() if state.revealed else "?"):
                            xalign 0.5 yalign 0.5 size 22

            if not state.revealed:
                textbutton "Reveal" action [Function(state.reveal), renpy.restart_interaction] xalign 0.5
            else:
                text ("You win!" if state.victory else ("Tie!" if state.player_card.numeric_value() == state.enemy_card.numeric_value() else "You lose...")):
                    xalign 0.5 size 24 bold True
                textbutton "Continue" action Return(state.victory) xalign 0.5
```

---

## 6. Passing Results Back to Story

All minigames return a boolean through `Return()`. The calling label reads `_return`:

```renpy
label minigame_card_duel:
    "The stranger deals the cards."

    python:
        store._card_state = HighCardState()

    call screen high_card_screen(_card_state)

    if _return:
        "You win the duel."
        $ gold += 50
        jump town_square_hub
    else:
        "You lost everything."
        $ gold = max(0, gold - 30)
        jump town_square_hub
```

For scored results, read the state object after the screen returns:

```renpy
label minigame_puzzle_chest:
    python:
        store._puzzle_state = GridPuzzleState(cols=4, rows=4)

    call screen grid_puzzle_screen(_puzzle_state)

    $ puzzle_score = _puzzle_state.score
    $ puzzle_moves = _puzzle_state.moves

    if _return:
        "Chest unlocked! (Score: [puzzle_score], Moves: [puzzle_moves])"
        jump chest_opened
    else:
        "The mechanism resets."
        jump chest_locked
```

---

## 7. Difficulty & Balancing Hooks

Centralize difficulty configuration so it's easy to tune:

```renpy
# systems/flags.rpy

default game_difficulty = "normal"   # "easy", "normal", "hard"

# systems/minigame_config.rpy

init python:

    MINIGAME_CONFIG = {
        "easy": {
            "qte_rounds":     4,
            "qte_time_limit": 2.5,
            "puzzle_cols":    4,
            "puzzle_rows":    3,
        },
        "normal": {
            "qte_rounds":     6,
            "qte_time_limit": 2.0,
            "puzzle_cols":    4,
            "puzzle_rows":    4,
        },
        "hard": {
            "qte_rounds":     8,
            "qte_time_limit": 1.4,
            "puzzle_cols":    6,
            "puzzle_rows":    4,
        },
    }

    def minigame_cfg(key):
        """Fetch config value for current difficulty."""
        diff = store.game_difficulty
        return MINIGAME_CONFIG.get(diff, MINIGAME_CONFIG["normal"])[key]
```

**Usage:**

```renpy
python:
    store._qte_state = QTEState(
        rounds=minigame_cfg("qte_rounds"),
        time_limit=minigame_cfg("qte_time_limit"),
    )
```

---

## 8. Universal Continuous Difficulty (0.0–1.0)

Instead of discrete `"easy" / "normal" / "hard"` levels, use a **continuous float in [0.0, 1.0]** that each minigame interprets through a pure derivation function. This makes balancing easier (tune one number per session) and lets you scale difficulty smoothly with player progress or context.

```renpy
# systems/minigame_difficulty.rpy

init python:

    def lerp(a, b, d):
        """Linear interpolation between a and b by factor d (clamped to [0, 1])."""
        d = max(0.0, min(1.0, d))
        return a + (b - a) * d


    # Reference ranges (informational; each minigame derives its own config):
    # 0.00–0.20  tutorial / very easy
    # 0.20–0.45  easy-normal
    # 0.45–0.75  normal-hard
    # 0.75–1.00  hard-extreme
```

**Each minigame defines a pure derivation function** `derive_X_config(d)`:

```renpy
init python:

    def derive_qte_config(d):
        """Universal difficulty → QTE-specific params."""
        return {
            "rounds":     int(round(lerp(4, 10, d))),
            "time_limit": lerp(2.5, 0.8, d),
        }

    def derive_memory_config(d):
        """Universal difficulty → memory grid params."""
        if d < 0.25:
            grid = (3, 2)
        elif d < 0.50:
            grid = (4, 3)
        elif d < 0.80:
            grid = (4, 4)
        else:
            grid = (5, 4)
        return {
            "grid":          grid,
            "time_limit":    lerp(90, 30, d),
            "miss_penalty":  lerp(0, 4, d),
            "pre_peek_sec":  lerp(3, 0, d),
            "trap_cards":    2 if d >= 0.80 else 0,
        }
```

**Calling from a story label** — the caller decides the difficulty for this session:

```renpy
label minigame_memory_at_casino:
    python:
        cfg = derive_memory_config(0.55)   # mid-high difficulty
        store._memory_state = TimedMemoryState(**cfg)
    call screen timed_memory_screen(_memory_state)
    $ result = _return
    # ...
```

**Adding a new minigame**: write `derive_newgame_config(d)` returning the params. No global tables, no enum changes.

---

## 9. MinigameResult Dataclass

Returning just `True` / `False` from a minigame is too thin for complex integrations. Use a richer `MinigameResult` so callers can read payout, stat deltas, and narrative flags.

```renpy
# systems/minigame_base.rpy

init python:

    class MinigameResult:
        """Standardized return type for minigames."""

        def __init__(self, outcome="loss", payout=0, stats_delta=None, narrative_flags=None):
            self.outcome         = outcome           # "win", "loss", "draw", "abort"
            self.payout          = payout            # net change in money for this session
            self.stats_delta     = stats_delta or {} # {stat_name: delta} to apply on caller side
            self.narrative_flags = narrative_flags or []  # list of flag strings to set on caller


    def apply_minigame_result(result):
        """Helper to apply a result to the global player state."""
        if result.payout != 0:
            store.player_survival.modify("money", result.payout)
        for stat, delta in result.stats_delta.items():
            store.player_survival.modify(stat, delta)
        for flag in result.narrative_flags:
            setattr(store, flag, True)
```

**Pattern for minigame state to produce a result**:

```renpy
init python:

    class MinigameState:
        # ... existing fields ...
        def build_result(self):
            """Override per minigame to construct a MinigameResult from current state."""
            return MinigameResult(
                outcome="win" if self.victory else "loss",
                payout=0,
                stats_delta={},
                narrative_flags=[],
            )
```

**Caller usage**:

```renpy
label minigame_blackjack_play:
    python:
        store._bj_state = BlackjackState(buy_in=30000, cap_gain=50000, difficulty=0.4)
    call screen blackjack_screen(_bj_state)
    python:
        result = _bj_state.build_result()
        apply_minigame_result(result)
    if result.outcome == "win":
        "You walk away with ${}.".format(result.payout)
    elif result.outcome == "draw":
        "You broke even."
    else:
        "You lost it all."
```

---

## 10. Blackjack Pattern (Complete)

Full blackjack implementation with hit / stand / double / split, multi-deck shoe, soft-17 toggle, configurable natural payout (3:2 vs 6:5), and an optional "rigged deck" knob for satirical or narrative contexts.

```renpy
# systems/minigame_blackjack.rpy

init python:

    import random

    BJ_RANKS = list(range(1, 14))   # 1=A, 11=J, 12=Q, 13=K
    BJ_SUITS = ["♠", "♥", "♦", "♣"]


    class BJCard:
        def __init__(self, rank, suit):
            self.rank = rank
            self.suit = suit

        def value(self, ace_high=False):
            if self.rank == 1:
                return 11 if ace_high else 1
            if self.rank >= 10:
                return 10
            return self.rank

        def name(self):
            faces = {1: "A", 11: "J", 12: "Q", 13: "K"}
            return faces.get(self.rank, str(self.rank))

        def display(self):
            return "{}{}".format(self.name(), self.suit)


    class BJShoe:
        """Multi-deck shoe for blackjack."""
        def __init__(self, n_decks=1, rigged_probability=0.0):
            self.n_decks   = n_decks
            self.rigged_p  = rigged_probability
            self._reset()

        def _reset(self):
            self.cards = []
            for _ in range(self.n_decks):
                for r in BJ_RANKS:
                    for s in BJ_SUITS:
                        self.cards.append(BJCard(r, s))
            random.shuffle(self.cards)

        def draw(self, prefer_high_for_dealer=False):
            """
            With rigged_p probability, dealer draws an unusually good card.
            Used for satirical "house always wins" hands. Otherwise random.
            """
            if prefer_high_for_dealer and random.random() < self.rigged_p:
                # Pick a face card or 10
                candidates = [i for i, c in enumerate(self.cards) if c.value() == 10]
                if candidates:
                    idx = random.choice(candidates)
                    return self.cards.pop(idx)
            if not self.cards:
                self._reset()
            return self.cards.pop()


    def hand_value(cards):
        """
        Compute the best (highest <= 21) value of a blackjack hand.
        Aces count as 11 unless that would bust.
        """
        total = sum(c.value() for c in cards)
        aces  = sum(1 for c in cards if c.rank == 1)
        # Upgrade aces from 1 → 11 while it doesn't bust
        while aces > 0 and total + 10 <= 21:
            total += 10
            aces  -= 1
        return total


    def is_soft(cards):
        """True if the hand counts at least one ace as 11."""
        total = sum(c.value() for c in cards)
        for _ in range(sum(1 for c in cards if c.rank == 1)):
            if total + 10 <= 21:
                return True
            total += 1
        return False


    def is_natural(cards):
        return len(cards) == 2 and hand_value(cards) == 21


    class BlackjackState(MinigameState):
        """
        Full blackjack with hit/stand/double/split.
        - buy_in / cap_gain define the session economy.
        - difficulty (0-1) tunes deck count, soft-17 rule, payout ratio, rigged-deck probability.
        """

        def __init__(self, buy_in=5000, cap_gain=10000, difficulty=0.4):
            super().__init__()
            self.buy_in  = buy_in
            self.cap_gain = cap_gain
            self.chips    = buy_in
            cfg = derive_blackjack_config(difficulty)
            self.cfg      = cfg
            self.shoe     = BJShoe(n_decks=cfg["n_decks"], rigged_probability=cfg["rigged_p"])
            self.bet      = cfg["min_bet"]
            self.hand     = []
            self.dealer   = []
            self.split    = None   # second hand if split
            self.doubled  = False
            self.in_round = False
            self.message  = ""

        def can_play(self):
            return self.chips >= self.cfg["min_bet"] and not self.is_over()

        def deal(self):
            if not self.can_play():
                self.lose()
                return
            self.in_round = True
            self.chips   -= self.bet
            self.hand    = [self.shoe.draw(), self.shoe.draw()]
            self.dealer  = [self.shoe.draw(prefer_high_for_dealer=True), self.shoe.draw()]
            self.doubled = False
            self.split   = None
            # Natural check
            if is_natural(self.hand) and not is_natural(self.dealer):
                payout = int(self.bet * (1 + self.cfg["natural_payout"]))
                self.chips += payout
                self.message = "Blackjack!"
                self._end_round()
            elif is_natural(self.dealer):
                self.message = "Dealer blackjack."
                self._end_round()

        def hit(self, on_split=False):
            target = self.split if on_split else self.hand
            target.append(self.shoe.draw())
            if hand_value(target) > 21:
                if on_split:
                    self.split = None
                else:
                    self.message = "Bust."
                    self._end_round()

        def stand(self):
            self._dealer_play()
            self._resolve_round()

        def double_down(self):
            if not self.cfg["allow_double"] or self.chips < self.bet:
                return
            self.chips  -= self.bet
            self.bet    *= 2
            self.doubled = True
            self.hit()
            if hand_value(self.hand) <= 21:
                self.stand()

        def split_hand(self):
            if (not self.cfg["allow_split"] or
                len(self.hand) != 2 or self.hand[0].rank != self.hand[1].rank or
                self.chips < self.bet):
                return
            self.chips -= self.bet
            self.split = [self.hand.pop(), self.shoe.draw()]
            self.hand.append(self.shoe.draw())

        def _dealer_play(self):
            """Dealer hits until 17. Soft-17 toggle from config."""
            while True:
                val = hand_value(self.dealer)
                if val > 21:
                    return
                if val < 17:
                    self.dealer.append(self.shoe.draw())
                    continue
                if val == 17 and is_soft(self.dealer) and self.cfg["hit_soft_17"]:
                    self.dealer.append(self.shoe.draw())
                    continue
                return

        def _resolve_round(self):
            player_val = hand_value(self.hand)
            dealer_val = hand_value(self.dealer)

            if player_val > 21:
                self.message = "Bust."
            elif dealer_val > 21 or player_val > dealer_val:
                self.chips += self.bet * 2
                self.message = "You win ${}.".format(self.bet)
            elif player_val == dealer_val:
                self.chips += self.bet
                self.message = "Push."
            else:
                self.message = "Dealer wins."

            self._end_round()

        def _end_round(self):
            self.in_round = False
            # Cap-gain enforcement: if winnings exceed cap, force player out
            if self.chips - self.buy_in >= self.cap_gain:
                self.message += " (table limit reached — cashout)"
                self.win()
            elif self.chips < self.cfg["min_bet"]:
                self.message += " (busted out)"
                self.lose()

        def cash_out(self):
            """Player voluntarily ends session."""
            if self.chips >= self.buy_in:
                self.win()
            else:
                self.lose()

        def build_result(self):
            """Convert chips → MinigameResult."""
            net = self.chips - self.buy_in
            return MinigameResult(
                outcome="win" if net > 0 else "draw" if net == 0 else "loss",
                payout=net,
            )


    def derive_blackjack_config(d):
        """Universal difficulty → blackjack params."""
        return {
            "n_decks":         int(round(lerp(1, 6, d))),
            "hit_soft_17":     d >= 0.5,
            "allow_double":    True,
            "allow_split":     d < 0.8,                    # restrict at high difficulty
            "natural_payout":  lerp(1.5, 1.2, d),          # 3:2 → 6:5 as difficulty rises
            "rigged_p":        lerp(0.0, 0.15, d),         # satirical knob
            "min_bet":         max(100, int(d * 1000)),
        }
```

The screen for blackjack follows the standard 3-layer pattern from Section 2 — render `state.hand`, `state.dealer`, expose buttons for `hit/stand/double/split/cash_out`. Use `renpy.restart_interaction` after each player action.

---

## 11. Memory Pattern (Timed Solitaire)

Solitaire memory match against a timer. Optional "trap cards" with no pair, and a pre-peek period where all cards are briefly visible.

```renpy
# systems/minigame_memory_timed.rpy

init python:

    import random

    class TimedMemoryState(MinigameState):
        """
        Memory match against the clock.
        - grid: (cols, rows)
        - time_limit: seconds
        - pre_peek_sec: cards visible at start
        - trap_cards: number of cards with no pair (decoy)
        - miss_penalty: seconds removed per unsuccessful flip
        """

        def __init__(self, grid=(4, 4), time_limit=60.0,
                     pre_peek_sec=1.0, trap_cards=0,
                     miss_penalty=2.0):
            super().__init__()
            self.cols, self.rows = grid
            cells = self.cols * self.rows
            n_pairs = (cells - trap_cards) // 2
            values  = list(range(n_pairs)) * 2 + [-1] * trap_cards
            random.shuffle(values)
            self.grid = [values[i*self.cols:(i+1)*self.cols] for i in range(self.rows)]
            self.revealed     = [[False]*self.cols for _ in range(self.rows)]
            self.matched      = [[False]*self.cols for _ in range(self.rows)]
            self._selected    = []
            self.moves        = 0
            self.misses       = 0
            self.time_left    = time_limit
            self.time_total   = time_limit
            self.pre_peek_sec = pre_peek_sec
            self.miss_penalty = miss_penalty
            self.pairs_needed = n_pairs
            self.pairs_found  = 0

        def is_face_up(self, r, c):
            if self.pre_peek_sec > 0:
                return True   # all visible during peek
            return self.revealed[r][c] or self.matched[r][c]

        def select(self, r, c):
            if self.pre_peek_sec > 0 or self.is_over():
                return
            if self.matched[r][c] or (r, c) in self._selected:
                return

            if len(self._selected) >= 2:
                for rr, cc in self._selected:
                    if not self.matched[rr][cc]:
                        self.revealed[rr][cc] = False
                self._selected = []

            self.revealed[r][c] = True
            self._selected.append((r, c))

            if len(self._selected) == 2:
                self.moves += 1
                r1, c1 = self._selected[0]
                r2, c2 = self._selected[1]
                v1 = self.grid[r1][c1]
                v2 = self.grid[r2][c2]
                if v1 == v2 and v1 != -1:
                    self.matched[r1][c1] = True
                    self.matched[r2][c2] = True
                    self.pairs_found += 1
                    self._selected = []
                    if self.pairs_found >= self.pairs_needed:
                        self.win()
                else:
                    self.misses += 1
                    self.time_left -= self.miss_penalty

        def tick(self, dt):
            if self.is_over():
                return
            if self.pre_peek_sec > 0:
                self.pre_peek_sec = max(0, self.pre_peek_sec - dt)
                return
            self.time_left -= dt
            if self.time_left <= 0:
                self.lose()

        def time_bonus_payout(self, cap):
            """Linear: % of time left → % of cap_gain."""
            if not self.victory:
                return 0
            pct = max(0, self.time_left) / self.time_total
            return int(cap * pct)

        def build_result(self):
            return MinigameResult(
                outcome="win" if self.victory else "loss",
                payout=0,   # caller calculates with time_bonus_payout(cap)
            )
```

The screen follows the pattern from Section 4 with an added timer bar and pre-peek indicator.

---

## 12. Minigame Session Economy (Buy-in + Caps)

Casinos and gambling minigames need an economy framing: pay buy-in to enter, win up to a per-session cap, with optional weekly accumulation caps to prevent infinite farming.

```renpy
# systems/minigame_economy.rpy

init python:

    class MinigameTable:
        """A table configuration: buy-in + cap of winnings per session."""

        def __init__(self, table_id, name, buy_in, cap_gain, difficulty,
                     unlock_condition=None):
            self.table_id        = table_id
            self.name            = name
            self.buy_in          = buy_in
            self.cap_gain        = cap_gain
            self.difficulty      = difficulty
            self.unlock_condition = unlock_condition

        def is_unlocked(self):
            return self.unlock_condition is None or self.unlock_condition()


    BLACKJACK_TABLES = {
        "basic":   MinigameTable("basic",   "Basic table",     5000,    10000, 0.20),
        "mid":     MinigameTable("mid",     "Mid table",      30000,    50000, 0.40),
        "high":    MinigameTable("high",    "High table",     70000,   100000, 0.55),
        "vip":     MinigameTable("vip",     "VIP table",     120000,   250000, 0.70,
                                 unlock_condition=lambda: store.influence >= 50),
        "high_roller": MinigameTable("high_roller", "High Roller", 300000, 500000, 0.85,
                                     unlock_condition=lambda: store.influence >= 70),
    }


    # Weekly cap of net minigame gains to prevent farming.
    default weekly_gambling_cap = 200000
    default weekly_gambling_used = 0


    def play_table(table_id, state_factory):
        """
        Enter a table, charge buy-in, run minigame, return result.
        Honors per-session cap (handled in state) and weekly cap.
        """
        table = BLACKJACK_TABLES.get(table_id)
        if not table or not table.is_unlocked():
            renpy.notify("Table unavailable.")
            return None
        if store.player_survival.money < table.buy_in:
            renpy.notify("Not enough money for buy-in.")
            return None

        store.player_survival.modify("money", -table.buy_in)
        state = state_factory(table.buy_in, table.cap_gain, table.difficulty)
        return state


    def apply_table_winnings(result):
        """Apply minigame result to money, honoring the weekly cap."""
        if result.payout <= 0:
            apply_minigame_result(result)
            return
        # Cap weekly net gambling income
        remaining = max(0, store.weekly_gambling_cap - store.weekly_gambling_used)
        actual_payout = min(result.payout, remaining)
        if actual_payout < result.payout:
            renpy.notify("House noticed you — winnings capped this week.")
        result.payout = actual_payout
        store.weekly_gambling_used += actual_payout
        apply_minigame_result(result)


    def reset_weekly_gambling_cap():
        """Call on Monday rollover."""
        store.weekly_gambling_used = 0
```

---

## 13. Stat Modifiers During Minigame

Player stats (energy, charisma, etc.) can modify a minigame's parameters at runtime. Apply them when constructing the state, before showing the screen.

```renpy
init python:

    def apply_player_modifiers(cfg, minigame_name):
        """
        Mutate a derived config dict in-place based on current player stats.
        Returns the modified cfg.
        """
        # Energy tier modifier (see renpy-survival.md section 10)
        e_tier = energy_tier()   # "high" / "mid" / "low"
        if minigame_name == "memory":
            if e_tier == "high":
                cfg["pre_peek_sec"] = cfg.get("pre_peek_sec", 0) + 1
            elif e_tier == "low":
                cfg["time_limit"]   *= 0.8
                cfg["miss_penalty"] += 1

        if minigame_name == "blackjack":
            if e_tier == "low":
                # Player has slower reflexes — UI delays slightly (signaled to screen)
                cfg["ui_delay_sec"] = 0.4

        # Influence modifier — e.g., bribe the dealer
        if minigame_name == "blackjack" and getattr(store, "influence", 0) >= 50:
            cfg["bribe_dealer_available"] = True

        return cfg


# Usage
python:
    cfg = derive_memory_config(0.55)
    cfg = apply_player_modifiers(cfg, "memory")
    store._memory_state = TimedMemoryState(**{k:v for k,v in cfg.items() if k in TimedMemoryState.__init__.__code__.co_varnames})
```

---

## 14. Narrative Event Interrupts

Sometimes mid-minigame, a story event should fire (an NPC sits at the table, a bouncer approaches, paparazzi appears). Use a callback hook between rounds/moves.

```renpy
init python:

    class MinigameInterruptHook:
        """
        Callback registry. Each minigame state checks for fired interrupts
        between turns / rounds / moves and calls renpy.call() if triggered.
        """

        def __init__(self):
            self.pending = []  # list of (condition_callable, label_to_call)

        def register(self, condition, label):
            self.pending.append((condition, label))

        def check_and_fire(self):
            still_pending = []
            for cond, label in self.pending:
                if cond():
                    renpy.call_in_new_context(label)
                else:
                    still_pending.append((cond, label))
            self.pending = still_pending


default minigame_interrupts = None

init python:
    def init_minigame_interrupts():
        store.minigame_interrupts = MinigameInterruptHook()
```

**Wire into your minigame state**:

```renpy
# At the end of BlackjackState._end_round, or between QTE rounds:
if store.minigame_interrupts:
    store.minigame_interrupts.check_and_fire()
```

**Registering an event before the minigame**:

```renpy
label minigame_blackjack_with_event:
    python:
        # Register: after 2 rounds, an NPC sits down
        rounds_played = [0]
        store.minigame_interrupts.register(
            condition=lambda: rounds_played[0] >= 2,
            label="event_npc_joins_table",
        )
        store._bj_state = BlackjackState(buy_in=30000, cap_gain=50000, difficulty=0.4)
    # ... continue minigame
```

The interrupt label can show dialogue, modify stats, and the minigame resumes after `return`.