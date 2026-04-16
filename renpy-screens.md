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

Full inventory screen with item use and discard:

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
                            # Item icon (optional)
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