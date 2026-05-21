# Plan: Consolidar sistema Inventory en skill propio

## Contexto

El sistema Inventory está repartido:
- `renpy-rpg.md` § 4: define `Item` + `ITEM_CATALOGUE` + `Inventory` (canonical).
- `renpy-adult-vn.md`: usa `store.player_inventory.has/add/remove` sin definirlo.
- `renpy-quests.md`: `ItemObjective` usa `store.player_inventory.has`.
- `renpy-screens.md` § 4: `inventory_screen` completo.

Issue: cross-skill dependency implícita, sin single source of truth.

## Decisiones fijadas

- **Tipos de items**: 5 estándar (Consumable / Equipment / Gift / QuestItem / KeyItem) + extension hook documentado.
- **Effects**: híbridos — `effects: dict` (data-driven) + opcional `effect_callable` para lógica compleja.
- **renpy-rpg**: mantener versión mínima de Inventory + cross-ref a versión full.

## Estructura del nuevo `renpy-inventory.md`

1. Architecture (files structure)
2. Item Base Class
3. Item Subclasses (Consumable / Equipment / Gift / QuestItem / KeyItem)
4. ITEM_CATALOGUE Pattern (declarative)
5. Inventory Class (add/remove/has/count/all/by_type/filter)
6. Persistence (save/load via Ren'Py)
7. Equipment Slots & Equipping
8. Gifting Pattern (give to NPC + affinity)
9. Quest Items Integration
10. Inventory Screen (with type filters)
11. Shop Integration Helpers
12. Extension Hook: Custom Item Types
13. Full Integration Checklist

## Tasks

1. Crear `renpy-inventory.md` completo.
2. Slim down `renpy-rpg.md` § 4 + cross-ref.
3. Cross-ref en `renpy-adult-vn.md` (introducir player_inventory).
4. Cross-ref en `renpy-quests.md` (ItemObjective).
5. Reducir/cross-ref `renpy-screens.md` § 4 (inventory_screen va a renpy-inventory).
6. Agregar `renpy-inventory.md` a `main-renpy-skill.md` (subsystem refs + combination guides).

## Criterio de éxito

- `renpy-inventory.md` es self-contained, cubre los 5 tipos + persistence + equipment + gifting.
- Otros skills referencian `renpy-inventory.md` para detalles, definen solo lo que necesitan localmente.
- `renpy-rpg.md` § 4 queda mínimo (Item + Inventory básico) con cross-ref.
- Actualizar `main-renpy-skill.md` para listar el nuevo skill en subsystem references.
