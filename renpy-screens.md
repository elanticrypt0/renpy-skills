# Ren'Py Screen Language Deep-Dive Reference

Use this for complex UI work: custom menus, HUDs, inventory screens, dialogue boxes, transitions.

---

## Table of Contents
1. [Screen Language Fundamentals](#1-screen-language-fundamentals)
2. [Styles & Style Prefixes](#2-styles--style-prefixes)
3. [Actions & Interactions](#3-actions--interactions)
4. [Inventory Screen](#4-inventory-screen)
5. [Dialogue Customization](#5-dialogue-customization)
6. [Animated UI Elements](#6-animated-ui-elements)
7. [Modal & Overlay Screens](#7-modal--overlay-screens)
8. [Accessibility](#8-accessibility)
9. [HUD Layout Pattern (Top/Bottom Bars)](#9-hud-layout-pattern-topbottom-bars)
10. [Responsive Layouts (PC vs Mobile)](#10-responsive-layouts-pc-vs-mobile)
11. [Themed Overlay UI (Phone-Style)](#11-themed-overlay-ui-phone-style)
12. [Stat Change Notification Queue](#12-stat-change-notification-queue)
13. [Auto-Hide HUD by Context](#13-auto-hide-hud-by-context)
14. [Tabs / Sub-Menu Within a Screen](#14-tabs--sub-menu-within-a-screen)

---

## 1. Screen Language Fundamentals

```renpy
screen my_screen(arg1, arg2=None):
    # zorder controls draw order; higher = on top
    zorder 20
    # modal True blocks clicks to screens underneath
    modal True
    # tag groups mutually-exclusive screens (only one "menu" screen shows at a time)
    tag menu

    # Layout containers
    vbox:             # vertical stack
        spacing 10
        hbox:         # horizontal stack
            spacing 8
            grid 3 2: # n columns × m rows
                pass
            fixed:    # absolute positioning within a container
                pass

    # Display elements
    text "Hello [player_name]!" size 24 color "#ffffff"
    add "images/ui/panel.png"
    image "images/ui/button.png"

    # Interaction elements
    button:
        action Return("ok")
        text "OK"

    textbutton "Cancel" action Return("cancel")

    imagebutton:
        idle  "images/ui/btn_idle.png"
        hover "images/ui/btn_hover.png"
        action Jump("some_label")

    bar value SomeValue() range 100 xsize 200 ysize 16

    input value VariableInputValue("player_name") length 20
```

---

## 2. Styles & Style Prefixes

Define all styles in `gui.rpy` or a dedicated `screens/styles.rpy`. Never set raw style properties inline in screen definitions.

```renpy
# screens/styles.rpy

style hud_frame:
    background Frame("images/ui/hud_bg.png", 10, 10)
    padding (12, 10)
    xminimum 200

style hud_label:
    color "#f0e6d2"
    size 16
    font "fonts/NotoSans-Regular.ttf"

style hud_value:
    color "#f1c40f"
    size 16
    bold True

# Use style_prefix to apply a prefix to all child widgets in a screen/container
screen stats_panel():
    frame:
        style_prefix "hud"
        # All text, bars, buttons inside inherit hud_ prefix styles
        vbox:
            text "HP" style "hud_label"
            text "[player_stats.current_hp]" style "hud_value"
```

**Style precedence:** `style_prefix` < explicit `style=` < inline properties. Prefer `style_prefix` over per-widget style overrides.

---

## 3. Actions & Interactions

Built-in actions to prefer over custom Python:

```renpy
# Navigation
Jump("label_name")
Call("label_name")
Return()
Return("value")
MainMenu()
ShowMenu("save")
ShowMenu("load")
ShowMenu("preferences")
ShowMenu("history")

# Variable modification
SetVariable("my_var", new_value)
ToggleVariable("flag")
IncrementVariable("counter", 1)

# Screen management
Show("screen_name")
Hide("screen_name")
ShowTransient("screen_name")   # auto-hides after interaction

# Preferences
Preference("text speed", "max")
Preference("auto-forward", "toggle")

# Compound actions
[Action1(), Action2()]          # run in sequence
If(condition, Action1(), Action2())   # conditional

# Custom function (use sparingly)
Function(my_python_func, arg1, arg2)
```

---

## 4. Inventory Screen

> **Full type-filtered inventory screen** (tabs for Consumables / Equipment / Gifts / Quest / Key Items, with per-type action buttons) **→ see `renpy-inventory.md` § 10**.
>
> The minimal viewport pattern below is fine for small projects with only one item type; for anything richer, prefer the full skill.

Minimal scrollable inventory screen (single list, generic use/discard buttons):

```renpy
# screens/inventory.rpy

screen inventory_screen():
    modal True
    zorder 40
    tag menu

    frame:
        xalign 0.5 yalign 0.5
        xsize 700 ysize 500
        padding (20, 16)

        vbox:
            spacing 12

            text "Inventory" size 26 bold True xalign 0.5
            text "Gold: [gold]" size 18 xalign 0.0

            # Scrollable item list
            viewport:
                scrollbars "vertical"
                mousewheel True
                ysize 340
                vbox:
                    spacing 8
                    for item, qty in player_inventory.all_items():
                        hbox:
                            spacing 10
                            xfill True
                            add "images/items/[item.item_id].png" xsize 32 ysize 32 if renpy.loadable("images/items/" + item.item_id + ".png")
                            vbox:
                                text "[item.name]  ×[qty]" size 18
                                text item.description size 14 color "#aaaaaa"
                            null xfill True
                            if item.item_type == "consumable":
                                textbutton "Use":
                                    action [
                                        Function(use_item_from_inventory, item.item_id),
                                        renpy.restart_interaction,
                                    ]
                            textbutton "Discard":
                                action [
                                    Function(player_inventory.remove, item.item_id),
                                    renpy.restart_interaction,
                                ]

            textbutton "Close" action Return() xalign 0.5


init python:
    def use_item_from_inventory(item_id):
        """Use a consumable item on the player."""
        item = ITEM_CATALOGUE.get(item_id)
        if item and item.item_type == "consumable":
            if item.use(store.player_stats):
                store.player_inventory.remove(item_id)
                renpy.notify("{} used!".format(item.name))
```

**Patterns this minimal version is missing** (all covered in `renpy-inventory.md`): type tabs, equipment equip/unequip buttons, gift target picker, quest item discard guard, hidden key items filter.

---

## 5. Dialogue Customization

Override default dialogue in `gui.rpy`:

```renpy
# gui.rpy — dialogue box customization

define gui.textbox_height = 185
define gui.textbox_yalign = 1.0   # pin to bottom

define gui.name_xpos = 300
define gui.name_ypos = 0
define gui.name_xalign = 0.0

define gui.dialogue_xpos = 268
define gui.dialogue_ypos = 50
define gui.dialogue_width = 744

# Custom font
define gui.text_font = "fonts/NotoSerif-Regular.ttf"
define gui.name_text_font = "fonts/NotoSans-Bold.ttf"
define gui.text_size = 24
define gui.name_size = 22
```

For fully custom dialogue boxes, override `screen say` in `screens/`:

```renpy
# screens/dialogue.rpy

screen say(who, what):
    style_prefix "say"
    zorder 5

    if who is not None:
        window:
            style "namebox"
            text who style "say_label"

    window:
        id "window"
        text what id "what"

    # Quick menu inline
    use quick_menu
```

---

## 6. Animated UI Elements

```renpy
# Pulsing button
transform pulse:
    ease 0.6 zoom 1.05
    ease 0.6 zoom 1.0
    repeat

# Notification pop-in
transform notify_in:
    alpha 0.0 yoffset -20
    ease 0.25 alpha 1.0 yoffset 0

transform notify_out:
    alpha 1.0 yoffset 0
    ease 0.3 alpha 0.0 yoffset -10


# In screens — use ATL inside displayable containers
screen animated_button_example():
    imagebutton:
        idle  "images/ui/cta_button.png"
        hover "images/ui/cta_button_hover.png"
        action Jump("start_game")
        at pulse   # apply ATL transform


# Custom notify override (replaces default Ren'Py notify)
screen notify(message):
    zorder 999
    at notify_in
    on hide:
        at notify_out

    frame:
        xalign 0.5 yalign 0.1
        padding (20, 10)
        background "#2c3e5088"
        text message size 20 color "#ffffff"

    timer 2.5 action Hide("notify")
```

---

## 7. Modal & Overlay Screens

Pattern for confirmation dialogs:

```renpy
# screens/dialogs.rpy

screen confirm(message, yes_action, no_action=Return(False)):
    modal True
    zorder 200

    add "#00000066"   # dim the background

    frame:
        xalign 0.5 yalign 0.5
        padding (30, 24)
        xminimum 400

        vbox:
            spacing 20
            xalign 0.5
            text message xalign 0.5 size 22
            hbox:
                spacing 24
                xalign 0.5
                textbutton "Yes" action yes_action
                textbutton "No"  action no_action


# Usage in story
label quit_confirm:
    show screen confirm(
        "Return to main menu?",
        [Return(True), MainMenu()],
        Return(False),
    )
```

---

## 8. Accessibility

```renpy
# options.rpy — ensure these are set
define config.allow_skipping = True
define config.skip_unseen = False    # True only for developer/debug

# Always provide text alternatives for icon-only buttons
screen icon_button_example():
    imagebutton:
        idle  "images/ui/settings_icon.png"
        hover "images/ui/settings_icon_hover.png"
        action ShowMenu("preferences")
        tooltip "Settings"   # screen reader / hover tip

# Use gui.text_font and gui.text_size from gui.rpy — never hardcode fonts in screens
# Minimum recommended text size: 18px for UI labels, 22px for dialogue
```

---

## 9. HUD Layout Pattern (Top/Bottom Bars)

A reusable HUD pattern: settings + day badge on top, stats + phone access on bottom. Leaves the central area free for backgrounds, sprites, dialogue.

```renpy
# screens/hud.rpy

screen game_hud():
    zorder 10
    style_prefix "hud"

    # ── Top-left: settings ─────────────────────────────────────────
    imagebutton:
        idle  "images/ui/settings_icon.png"
        hover "images/ui/settings_icon_hover.png"
        action ShowMenu("preferences")
        xalign 0.0 yalign 0.0
        xoffset 12 yoffset 12

    # ── Top-right: day / period badge ──────────────────────────────
    frame:
        xalign 1.0 yalign 0.0
        xoffset -12 yoffset 12
        padding (12, 8)
        background "#00000099"
        hbox:
            spacing 8
            text "Day [game_time.day]" size 14 color "#ffffff"
            text "·" color "#888"
            text game_time.day_name size 14 color "#ffffff"
            text "·" color "#888"
            text game_time.period.capitalize() size 14 bold True color "#f1c40f"

    # ── Bottom-left: stats pills ────────────────────────────────────
    hbox:
        xalign 0.0 yalign 1.0
        xoffset 12 yoffset -12
        spacing 8

        frame:
            padding (10, 6)
            background "#1a1a1aaa"
            hbox:
                spacing 6
                text "$" color "#2ecc71" size 14 bold True
                text "[player_survival.money]" size 14 color "#ffffff"

        frame:
            padding (10, 6)
            background "#1a1a1aaa"
            hbox:
                spacing 6
                text "INF" color "#f1c40f" size 12 bold True
                text "[player_survival.influence if hasattr(player_survival, 'influence') else 0]" size 14 color "#ffffff"

        frame:
            padding (10, 6)
            background "#1a1a1aaa"
            hbox:
                spacing 6
                text "IMG" color "#3498db" size 12 bold True
                text "[player_survival.image if hasattr(player_survival, 'image') else 0]" size 14 color "#ffffff"

        frame:
            padding (10, 6)
            background "#1a1a1aaa"
            hbox:
                spacing 6
                text "⚡" color "#e74c3c" size 14 bold True
                text "[player_survival.energy]" size 14 color "#ffffff"

    # ── Bottom-right: phone access ─────────────────────────────────
    imagebutton:
        idle  "images/ui/phone_icon.png"
        hover "images/ui/phone_icon_hover.png"
        action ShowMenu("phone_overlay")
        xalign 1.0 yalign 1.0
        xoffset -12 yoffset -12
```

The HUD is shown via `show screen game_hud` at the start of gameplay and hidden during cinematics (see Section 13).

---

## 10. Responsive Layouts (PC vs Mobile)

Ren'Py supports per-platform screen variants through `renpy.variant`. Define **two layouts** and have Ren'Py auto-select.

```renpy
# screens/hud.rpy — variant-aware HUD

screen game_hud():
    if renpy.variant("small"):
        use game_hud_mobile()
    else:
        use game_hud_pc()


screen game_hud_pc():
    # Full horizontal layout (1920×1080), as in Section 9
    # ...


screen game_hud_mobile():
    # Vertical-friendly layout: stats wrap to two rows, larger touch targets
    zorder 10
    style_prefix "hud_mobile"

    imagebutton:
        idle "images/ui/settings_icon.png"
        action ShowMenu("preferences")
        xalign 0.0 yalign 0.0
        xoffset 12 yoffset 12
        xsize 56 ysize 56   # bigger touch target

    frame:
        xalign 1.0 yalign 0.0
        xoffset -12 yoffset 12
        padding (12, 8)
        background "#00000099"
        vbox:
            spacing 2
            text "Day [game_time.day]" size 16 color "#ffffff"
            text game_time.period.capitalize() size 14 color "#f1c40f"

    frame:
        xalign 0.5 yalign 1.0
        yoffset -12
        padding (12, 10)
        background "#1a1a1aaa"
        xsize 720
        vbox:
            spacing 6
            hbox:
                spacing 14 xalign 0.5
                text "$ [player_survival.money]" size 18 color "#2ecc71"
                text "⚡ [player_survival.energy]" size 18 color "#e74c3c"
            hbox:
                spacing 14 xalign 0.5
                text "INF [player_survival.influence]" size 16 color "#f1c40f"
                text "IMG [player_survival.image]" size 16 color "#3498db"

    imagebutton:
        idle "images/ui/phone_icon.png"
        action ShowMenu("phone_overlay")
        xalign 1.0 yalign 1.0
        xoffset -12 yoffset -12
        xsize 64 ysize 64
```

**Variant aliases**: Ren'Py treats Android/iOS as `"touch"` and `"small"`. PC desktops are the default. To test mobile layout on desktop, set `config.variants = ["small", "touch"]` in `options.rpy` temporarily.

---

## 11. Themed Overlay UI (Phone-Style)

For games that present journals / inventory / contacts inside a **diegetic phone** interface (the player's character has a smartphone). Apps grid + bottom nav + thematic frame.

```renpy
# screens/phone_overlay.rpy

default phone_current_app = "home"

screen phone_overlay():
    modal True
    zorder 60
    tag menu

    # Dimmer
    add "#00000088"

    # Phone frame
    frame:
        xalign 0.5 yalign 0.5
        xsize 540 ysize 880
        background "images/ui/phone_frame.png"
        padding (40, 80)

        # Status bar inside phone screen
        hbox:
            xalign 0.5 yfill False
            spacing 8
            text "[game_time.day_name] [game_time.period]" size 14 color "#ffffff"

        # Active app area
        frame:
            xalign 0.5 yalign 0.0
            yoffset 40
            xsize 460 ysize 700
            background "#0a0a0a"
            padding (12, 12)

            if phone_current_app == "home":
                use phone_home_grid
            elif phone_current_app == "missions":
                use phone_missions_app
            elif phone_current_app == "contacts":
                use phone_contacts_app
            elif phone_current_app == "gossip":
                use phone_gossip_app
            elif phone_current_app == "inventory":
                use phone_inventory_app
            elif phone_current_app == "stats":
                use phone_stats_app
            elif phone_current_app == "gallery":
                use phone_gallery_app

        # Bottom nav: back / home / close
        hbox:
            xalign 0.5 yalign 1.0
            yoffset -40 spacing 30
            textbutton "Back":
                action SetVariable("phone_current_app", "home")
                sensitive (phone_current_app != "home")
            textbutton "Home":
                action SetVariable("phone_current_app", "home")
            textbutton "Close":
                action Return()


screen phone_home_grid():
    grid 2 3:
        spacing 16
        xalign 0.5 yalign 0.5
        $ apps = [
            ("missions",  "Missions",  "ui/app_missions.png"),
            ("contacts",  "Contacts",  "ui/app_contacts.png"),
            ("gossip",    "Gossip",    "ui/app_gossip.png"),
            ("inventory", "Items",     "ui/app_inventory.png"),
            ("stats",     "Stats",     "ui/app_stats.png"),
            ("gallery",   "Gallery",   "ui/app_gallery.png"),
        ]
        for app_id, label, icon in apps:
            vbox:
                spacing 4
                xalign 0.5
                imagebutton:
                    idle "images/[icon]"
                    xsize 80 ysize 80
                    action SetVariable("phone_current_app", app_id)
                text label size 13 xalign 0.5 color "#ffffff"
```

Each `phone_X_app` screen is its own self-contained app. The frame, dimmer, status bar, and nav are shared.

---

## 12. Stat Change Notification Queue

Floating "+5 Trust", "-10 Money" notifications that appear near the stat being changed, stack vertically if multiple fire at once, and fade away.

```renpy
# systems/stat_notify.rpy

default stat_notify_queue = []

init python:

    class StatNotify:
        def __init__(self, text, color, anchor_pos, ts):
            self.text       = text
            self.color      = color
            self.anchor_pos = anchor_pos   # (x, y) in screen coords
            self.spawned_at = ts


    def push_stat_notify(text, color, anchor_pos):
        ts = renpy.get_game_runtime()
        store.stat_notify_queue.append(StatNotify(text, color, anchor_pos, ts))
        # Trim queue
        if len(store.stat_notify_queue) > 8:
            store.stat_notify_queue.pop(0)
        renpy.restart_interaction()


    def expire_stat_notify():
        """Remove notifies older than 2.5s. Call from the screen timer."""
        ts = renpy.get_game_runtime()
        store.stat_notify_queue = [n for n in store.stat_notify_queue
                                    if ts - n.spawned_at < 2.5]


transform notify_float:
    alpha 1.0 yoffset 0
    parallel:
        ease 2.0 yoffset -40
    parallel:
        pause 1.5
        ease 1.0 alpha 0.0


screen stat_notify_overlay():
    zorder 100
    timer 0.25 repeat True action Function(expire_stat_notify)

    python:
        notifs = list(store.stat_notify_queue)

    for i, n in enumerate(notifs):
        $ stagger_y = i * 22   # stack downward if multiple
        text n.text:
            size 16 bold True color n.color
            xpos n.anchor_pos[0] ypos (n.anchor_pos[1] + stagger_y)
            at notify_float
```

**Trigger from anywhere**:

```renpy
init python:
    def modify_stat_with_notify(stat, delta, anchor_pos=(50, 980)):
        store.player_survival.modify(stat, delta)
        color = "#2ecc71" if delta > 0 else "#e74c3c"
        sign = "+" if delta > 0 else ""
        push_stat_notify("{}{} {}".format(sign, delta, stat.title()), color, anchor_pos)


# Usage
$ modify_stat_with_notify("influence", +5, anchor_pos=(120, 980))
```

Add `show screen stat_notify_overlay` once at game start so it's always available.

---

## 13. Auto-Hide HUD by Context

Show the HUD during gameplay (hub, map, minigame, inventory) and **auto-hide** during cinematics, dialogues, and scene replays where it would distract.

```renpy
# systems/hud_visibility.rpy

init python:

    def show_hud():
        renpy.show_screen("game_hud")
        renpy.show_screen("stat_notify_overlay")

    def hide_hud():
        renpy.hide_screen("game_hud")
        renpy.hide_screen("stat_notify_overlay")
```

**Manual control in story labels**:

```renpy
label cutscene_start:
    $ hide_hud()
    scene bg_dramatic with dissolve
    "..."
    return

label cutscene_end:
    $ show_hud()
    return
```

**Convention**: define `auto_hide_hud_for_labels` in `options.rpy` and let a wrapper handle this for known cinematic prefixes.

```renpy
init python:

    AUTO_HIDE_PREFIXES = ("cinematic_", "ending_", "scene_intimate_")

    def label_callback(name, abnormal):
        if any(name.startswith(p) for p in AUTO_HIDE_PREFIXES):
            hide_hud()
        elif name in ("hub_main", "start_of_day"):
            show_hud()

    config.label_callback = label_callback
```

**Tecla `H` toggle** for player-driven hiding (screenshots, immersion):

```renpy
# script.rpy or screens/global_keys.rpy
screen global_keys():
    zorder 999
    key "h" action ToggleScreen("game_hud")
```

---

## 14. Tabs / Sub-Menu Within a Screen

Pattern for screens with multiple tabs (gallery sections, journal categories, settings groups). Uses `SetScreenVariable` so tab state is local to the screen instance.

```renpy
screen tabbed_view():
    modal True
    zorder 50

    default selected_tab = "tab_one"   # local default

    frame:
        xalign 0.5 yalign 0.5
        xsize 800 ysize 540
        padding (20, 16)
        vbox:
            spacing 12

            # Tab bar
            hbox:
                spacing 14
                xalign 0.5
                for tab_id, tab_label in [
                    ("tab_one",   "First"),
                    ("tab_two",   "Second"),
                    ("tab_three", "Third"),
                    ("tab_four",  "Fourth"),
                ]:
                    textbutton tab_label:
                        action SetScreenVariable("selected_tab", tab_id)
                        text_color ("#f1c40f" if selected_tab == tab_id else "#888")
                        text_size 16

            # Divider
            frame:
                xsize 760 ysize 2
                background "#333"

            # Active tab content
            frame:
                xsize 760 ysize 420
                if selected_tab == "tab_one":
                    use tab_one_content
                elif selected_tab == "tab_two":
                    use tab_two_content
                elif selected_tab == "tab_three":
                    use tab_three_content
                else:
                    use tab_four_content

            textbutton "Close" action Return() xalign 0.5
```

Each `tab_X_content` is its own `screen` definition. The tab bar pattern works for galleries, journals, options panels, etc.