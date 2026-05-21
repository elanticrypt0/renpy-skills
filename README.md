# renpy-skills

Experimental notes and guidelines for working with Ren'Py.

## Structure

* `renpy-rpg.md` — RPG systems (stats, battles, progression, HUD)
* `renpy-maps.md` — World navigation and location systems
* `renpy-minigames.md` — Reusable minigame patterns and implementations
* `renpy-screens.md` — Screen language, UI patterns, and components
* `renpy-survival.md` — Survival stats, time/day cycle, action economy, stat decay
* `renpy-quests.md` — Quest/mission system, objectives, rewards, quest chains
* `renpy-adult-vn.md` — Sandbox/adult VN: relationship trees, scene unlocking, gallery, NPC scheduling

## Purpose

This repository serves as a structured reference for organizing knowledge about Ren'Py, focusing on scalable project architecture and reusable patterns.

### Core topics covered

* Canonical project folder structure (VN and sandbox VN variants)
* Naming conventions (labels, variables, flags, images, screens)
* Character definition patterns
* Story/script standards (label hierarchy, branching, Python usage)
* Centralized flags and variables (`systems/flags.rpy`)
* Image and sprite naming (auto-detection, `LayeredImage`)
* Transforms and transitions (named, reusable, no magic numbers)
* Audio management (including fade helpers)
* Save system and chapter boundary handling
* Branching patterns (flat labels vs nested menus)
* Survival stats with daily decay and action economy
* Time/day cycle (morning/afternoon/evening/night, days of week)
* Quest system with state machines, objectives, and reward chains
* Relationship trees with scene unlock conditions
* NPC scheduling by time period and location
* Gallery screen for scene replay
* Adult content toggle and alternate scene versions
* Quality checklist for consistency and maintainability

Content is exploratory and may evolve over time.

## License

Apache 2.0
