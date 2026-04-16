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