# Findings de auditoría de skills Ren'Py

Fecha: 2026-05-15.
Auditor: directo, secuencial.
Formato: por skill — bullets cortos: **Tiene** / **Gaps vs Politicuck** / **Acción**.

---

## 1. `main-renpy-skill.md` (entrypoint)

**Tiene** (bueno):
- Frontmatter Claude Code correcto (name/description/triggers).
- Project structure clara (game/, characters/, story/, screens/, systems/, images/, audio/).
- Naming conventions tabla.
- Character defs (con `image=`, `voice_tag`), LayeredImage.
- Script/label standards (flat labels, jump vs call, return-pattern).
- Flags/variables: default vs define vs persistent, todo centralizado en `systems/flags.rpy`.
- Audio helpers + transitions + transforms reutilizables.
- Branching patterns (anti-nesting).
- Quality checklist.
- Subsystem references + combination guide.

**Gaps vs Politicuck**:
- ❌ Sin variante mobile/Android (layout dual horizontal+vertical).
- ❌ Sin `gallery/` ni `minigames/` en project structure (Politicuck los necesita).
- ❌ Sin mención a debug mode (Sec 14 GDD).
- ❌ Sin i18n / translate blocks (mencionado post-MVP).
- ❌ Subsystem combination guide cubre "sandbox life-sim" pero no "sandbox political/corruption" como variante.
- ⚠️ Referencias internas usan `references/` subdir, repo actual los tiene en root. Verificar al empaquetar.

**Acción**: **MEJORAR**
- Agregar carpetas opcionales `gallery/` + `minigames/` en project structure.
- Agregar sección 14 nueva "Mobile / multi-platform" con notas sobre layout dual y `renpy.variant`.
- Agregar mención breve a debug mode con cross-ref a futuro skill.
- Agregar item de checklist para i18n setup básico.
- Agregar variante "Sandbox political / corruption sim" en combination guide.

---

## 2. `renpy-adult-vn.md` (sandbox VN)

**Tiene** (muy completo):
- Project structure sandbox completa con `story/quests/`, `story/scenes/`, `systems/character_system.rpy`, `systems/npc_schedule.rpy`.
- Classes `CharacterRelationship` (value 0-100, scenes, modify, progress_percent) y `SceneDefinition` (rel_required, items_required, flags_required, is_adult, requirements_met, missing_requirements).
- Scene registry: `try_unlock_scene`, `play_scene`, `scene_id_to_label`, `check_new_scenes_available`.
- Hub screen con `HubLocation`, `available_periods`, `unlock_condition`, `is_available`.
- `NPC_SCHEDULE` con periodo+location, helpers `npc_current_location`, `npc_is_here`, `npcs_at_location`.
- 4 patterns de content gating (relationship, item, scene trigger helper, flag).
- Gallery screen con tabs por NPC, thumbnails, locked-state con "???" + hint.
- Persistent.seen_scenes con sync `after_load`, persistent.achievements declarado.
- Adult content toggle (on/off) con fade-to-black alternativo.
- Integration checklist completo.

**Gaps vs Politicuck (GDD)**:
- ❌ **Multi-stage scene chains (NTR)**: `SceneDefinition` es flat. Politicuck necesita "secuencia" multi-etapa (regalo → cena → noche) con gating entre etapas y misión asociada.
- ❌ **Stats opaque to player**: el code asume value visible al jugador. GDD pide indicadores cualitativos ("le caés bien", "te mira con desprecio").
- ❌ **City-wide reputation / scandal meter**: falta stat global tipo Chismerío que afecta a TODOS los NPCs y a Imagen global.
- ❌ **NPC suspicion stat**: stats NPC del GDD son 4 (Confianza, Deseo cónyuge, Poder, Sospecha). El skill solo modela `value` (relationship).
- ❌ **Mini-game during scene** (paparazzi click interrupt): no hay pattern para mini-juegos in-scene que pueden cambiar el outcome.
- ❌ **NPC autonomous off-screen events** (cameos cónyuge): no hay pattern para "escenas que ocurren sin Dorni presente" con flag silencioso.
- ❌ **Achievement system completo**: persistent.achievements existe pero no hay pattern de definición, trigger, CG bonus, gallery integration.
- ❌ **Adult distribution policy**: no menciona content prohibido (TOS-safe), disclaimer +18 splash.
- ⚠️ **Adult toggle obligatorio**: el skill da toggle como única opción. Falta documentar variante "all-adult, no toggle" (decisión válida).
- ⚠️ **Sub-zonas dentro de hub locations**: HubLocation es atomic. Casino/congreso/quinta del GDD necesitan sub-menu. (Aunque esto puede ser cross-cut con renpy-maps.)

**Acción**: **MEJORAR** (skill clave para Politicuck, varios gaps importantes).

- Agregar sección **"Multi-stage scene chains"** con `SceneSequence` (lista ordenada de SceneDefinitions + gating entre etapas + integración con quest_manager).
- Agregar sección **"Opaque NPC stats — qualitative indicators"** con función `get_relationship_label(value)` y helpers visuales.
- Agregar sección **"Multi-dimensional NPC stats"** con dataclass-like `NPCRelationship` extendido (trust, desire_for_X, power_tier, suspicion).
- Agregar sección **"City-wide reputation / scandal meter"** con stat global, decay/raise hooks, thresholds, soborno helpers.
- Agregar sección **"Achievements system"** completo: definición declarativa, trigger, CG bonus, persistent + gallery cross-save.
- Agregar sección **"Mini-game interruptions during scenes"** con pattern de click/QTE in-scene + outcome branching.
- Agregar sección **"NPC autonomous events (off-screen cameos)"** con flag silencioso `npc_hint_N` y trigger por contexto.
- Agregar sección **"Adult content distribution"** con: TOS-safe content list (qué NO incluir), variante "no toggle" documentada, splash disclaimer +18 pattern.

---

## 3. `renpy-survival.md` (time/slots/stats vida)

**Tiene** (sólido):
- `SurvivalStats` class: energy/mood/fitness/social/money 0-100 (money uncapped, puede negativo).
- Helpers: `_clamp`, `modify`, `is_exhausted`, `is_broke`, `is_depressed`, `critical_stats`, `end_of_day_decay`.
- `TimeManager`: day, period_idx, week. Properties period, day_name, is_weekend. Methods advance, skip_to, label_for_period.
- 4 períodos default (morning/afternoon/evening/night).
- `ACTIONS_PER_PERIOD` dict.
- `spend_action(count)` + `on_new_day()` hook.
- Stat decay por período y end-of-day.
- `do_sleep()` restora energía.
- `check_survival_fail()` con 3 fails (exhaustion, despair, bankruptcy a -500).
- survival_hud screen con bars + actions left + critical warning.
- Integration example completo.

**Gaps vs Politicuck (GDD)**:
- ⚠️ **3 slots/día (1 action por slot)** vs el modelo "N actions per period" del skill. Políticamente: GDD usa mañana/tarde/noche = 3 slots, 1 acción c/u. El skill usa 4 periods con 3+3+2+1 actions. Distintos modelos coexisten — falta variante "1 action = 1 period".
- ⚠️ **Energía como modifier** (afecta calidad), no como limitador (gating duro). El skill solo modela limitador (menú deshabilitado si energy < X). Falta variante.
- ❌ **Gastos fijos semanales** (alquiler/servicios/comida): no hay scheduled bills automáticos por día N.
- ❌ **Sueldo por asistencia**: skill da $80 plano por trabajar. Falta tracker "slots trabajados esta semana" + despido por faltas.
- ❌ **Day-of-week gating de lugares/NPCs**: `is_weekend` existe pero no hay pattern de "lugar X solo abre Lun-Vie".
- ❌ **Debt ramp / escalating consequences**: skill hace hard game over a -$500. Politicuck quiere ramps (warning → prestamistas → mafia → game over). Falta pattern.
- ❌ **Skip slot voluntariamente**: skill solo skip_to vía sleep. Falta pattern "dormir antes de gastar todos los slots" o "saltar este slot".
- ⚠️ **Stats no-survival** (Influencia, Imagen, Chismerío): no pertenecen acá. Pero el skill podría linkear a un futuro skill de "thematic stats con tensión opuesta" o documentar que SurvivalStats es solo para vida básica.

**Acción**: **MEJORAR**
- Agregar variante **"One action per slot"** (3 slots = 3 acciones máximas/día).
- Agregar pattern **"Energy as modifier"** (no como limitador): low/mid/high tiers que modifican calidad sin gatear.
- Agregar pattern **"Scheduled bills"** (callbacks por día N para cobros automáticos).
- Agregar pattern **"Wage with attendance tracking"** (tracker semanal + despido).
- Agregar pattern **"Day-of-week gating"** para apertura de lugares/NPCs.
- Agregar pattern **"Debt ramp with narrative consequences"** (warning → loan sharks → mafia → game over).
- Agregar pattern **"Skip slot / sleep early"**.

---

## 4. `renpy-maps.md` (hotspots/navegación)

**Tiene**:
- `Location` class con map_pos fractional (0.0-1.0), icon, entry_label, unlock_condition, connections.
- `LOCATIONS` registry dict + `get_available_locations(from_loc_id)`.
- `world_map_screen` con imagebuttons posicionados (idle/hover/insensitive/dim icons).
- Travel logic + entry_label dispatch.
- Mini-map / area HUD pattern para sub-rooms (Room class con exits).
- Tooltip + current location indicator.

**Gaps vs Politicuck (GDD)**:
- ⚠️ **Connections obligatorias**: skill modela viaje entre puntos conectados (RPG-like). Politicuck necesita "flat sandbox" (all unlocked = reachable).
- ❌ **Hotspots animados**: no hay pattern para hotspot que brilla si hay NPC esperando o evento pendiente.
- ❌ **Sub-zonas dentro de location**: skill tiene `Room` para mini-map de mazmorra, pero no para sub-menu single location (casino → blackjack/bar/baño).
- ❌ **Locked hotspot con reason tooltip**: tiene icons dim/insensitive pero no muestra requisitos faltantes ("Requiere Influencia ≥ 25").
- ⚠️ **Travel cost optional**: skill no contempla costo de viaje. Politicuck: gratis MVP pero documentar opción.
- ❌ **Cross-ref a NPC scheduling** (renpy-adult-vn): qué NPCs/lugares solo aparecen ciertos days/periods.

**Acción**: **MEJORAR**
- Agregar variante **"Flat sandbox map"** (sin connections, all unlocked = reachable).
- Agregar pattern **"Animated hotspot indicators"** (glow si evento esperando, ícono notif).
- Agregar sección **"Sub-zones within a location"** (sub-menu para casino/congreso/quinta).
- Agregar pattern **"Locked hotspot with reason tooltip"** (mostrar requisitos faltantes en hover).
- Agregar pattern **"Travel cost (optional)"** (slot/$/energía cuando aplica).
- Cross-ref a `renpy-adult-vn` (NPC scheduling) y a `renpy-survival` (day-of-week).

---

## 5. `renpy-quests.md` (misiones)

**Tiene** (muy completo):
- Estados: HIDDEN, AVAILABLE, ACTIVE, COMPLETE, FAILED.
- `Quest` class con try_unlock, start, complete, fail, all_objectives_met, check_auto_complete, chain_next.
- 6 tipos de `Objective`: Talk, Item, Stat, Counter, Location, Flag — cada uno con `check()` o `mark_complete()`.
- Quest registry como data (`_build_quests()` → dict).
- `QuestManager` con is_active/is_complete/is_available, active_quests/available_quests, tick_unlock_conditions, check_all_stat_objectives.
- quest_journal_screen con paneles Active y Available.
- 5 tipos de `Reward`: Stat, Money, Item, Relationship, SceneUnlock.
- Quest chains via `chain_next`.
- Integration patterns con triggers (dialogue, stat change, conditional).

**Gaps vs Politicuck (GDD)**:
- ❌ **Misiones con etapas progresivas** (regalo → cena → noche): skill modela objectives plana. Politicuck necesita stages con sub-objectives + recompensa parcial + escena disparada al cerrar cada etapa.
- ❌ **Misiones procedurales con templates**: skill solo tiene quests guionadas (one-shot). Politicuck necesita pool de templates con generator (`traer_objeto`, `entregar_sobre`, `sacar_foto`, etc.).
- ❌ **Deadlines suaves**: skill sin concepto de plazo. Politicuck quiere deadline opcional con penalización al expirar (no cancela, baja recompensa / +Sospecha).
- ❌ **Tipos de misión categorizados** (NTR / NPC / procedural / trama): skill sin field `category` para filtros UI (4 pestañas en agenda).
- ❌ **Quest cancellation**: no hay método `cancel()` con on_cancel callback. Politicuck: Vicky permite cancelar NTR con penalización.
- ❌ **Multi-stage rewards**: rewards aplican solo al completar quest. Falta partial reward por stage.
- ⚠️ **Repetible vs one-shot**: skill asume one-shot. Recados procedurales se regeneran. Falta flag `repeatable` + reset.

**Acción**: **MEJORAR**
- Agregar pattern **"Multi-stage quests"** (`StagedQuest(Quest)` con `stages: list[QuestStage]`, cada stage con objectives + partial reward + scene_label).
- Agregar pattern **"Procedural quests from templates"** (`QuestTemplate` class + `generate_procedural_quest(template_id)` con variable filling).
- Agregar pattern **"Soft deadlines"** (`deadline_days` field, `on_expire` callback, helper `is_expired()`).
- Agregar campo **`category`** en `Quest` (string tag para filtros UI).
- Agregar método **`Quest.cancel()`** con `on_cancel` callback + penalty.
- Agregar **`Reward.partial`** flag para recompensas mid-stage.
- Agregar **`Quest.repeatable`** flag + reset semántica para procedurales.

---

## 6. `renpy-minigames.md` (minijuegos)

**Tiene**:
- Arquitectura 3 capas (State Python / Screen / Story Integration).
- `MinigameState` base class.
- 3 patterns concretos: **QTE** (timed reaction), **GridPuzzle** (memory match), **HighCard** (cards muy simple).
- Memory match con grid configurable (cols×rows).
- Cards con `Deck`, `Card`, suits, values.
- Passing results via `Return` + `_return`.
- Difficulty hooks discretos (easy/normal/hard).

**Gaps vs Politicuck (GDD)**:
- ❌ **Dificultad universal 0-1 continua**: skill usa 3 niveles discretos. GDD pide número 0-1 + `derive_xxx_config(d)` con `lerp(min, max, d)` para interpolación.
- ❌ **Blackjack completo**: skill solo tiene "HighCard" trivial. GDD pide blackjack real con hit/stand/double/split, multi-deck, soft 17 toggle, natural 3:2 vs 6:5, "mazo amañado" satírico.
- ⚠️ **Memoria con tiempo límite**: `GridPuzzleState` no tiene timer. GDD pide tiempo + cartas trampa + pre-peek por Influencia.
- ❌ **Mesa con buy-in fijo + cap de ganancia**: skill no modela "session" de minijuego con economía (entrar pagando buy-in, jugar manos múltiples, salir con saldo).
- ❌ **Cap semanal anti-farm**: nada implementado.
- ❌ **Stat modifiers durante minijuego**: skill sin hooks para Energy bajo = peor performance / Influence = ventaja.
- ❌ **Narrative event during minigame**: NPC interrupt mid-session (NPC entra al casino entre manos) no contemplado.
- ⚠️ **Win/lose binary**: skill devuelve solo bool. Politicuck necesita devolver `MinigameResult` con outcome / payout / stats_delta / narrative_flags.

**Acción**: **MEJORAR**
- Agregar sección **"Universal continuous difficulty (0.0–1.0)"** con `lerp(a,b,d)` helper + ejemplo `derive_xxx_config(d)`.
- Agregar sección **"Blackjack pattern"** completo (hit/stand/double/split, multi-deck, soft 17 toggle, 3:2 vs 6:5, rigged-deck satirical knob).
- Mejorar **"Memory pattern"** con timer, trap cards, pre-peek (modificador externo), scoring por tiempo restante.
- Agregar sección **"Minigame session economy"** (buy-in fijo + cap de ganancia + cap semanal anti-farm).
- Agregar sección **"Stat modifiers during minigame"** (energy tier → bonus/penalty windows).
- Agregar pattern **"Narrative event interrupt"** durante sesión (callback hook entre rondas).
- Agregar **`MinigameResult` dataclass** (outcome, payout, stats_delta, narrative_flags) en vez de bool.

---

## 7. `renpy-screens.md` (UI/HUD)

**Tiene**:
- Screen Language fundamentals (zorder, modal, tag, containers, display elements, interactions).
- Styles + `style_prefix` con tabla de precedencia.
- Actions built-in completas (Jump/Call/Return/ShowMenu/SetVariable/ToggleVariable/Show/Hide/Preference/Function/If).
- Inventory screen completo con viewport + scrollbars.
- Dialogue customization (`gui.textbox_*` + override `screen say`).
- Animated UI (transforms pulse/notify, custom notify override).
- Modal/overlay confirmation pattern.
- Accessibility basics (allow_skipping, tooltips, text size mínimos).

**Gaps vs Politicuck (GDD)**:
- ❌ **HUD layout pattern** (top + bottom bars con stats / día / energía / dinero): el skill no documenta el HUD genérico (survival skill lo tiene parcial, pero falta el pattern aquí).
- ❌ **Responsive / multi-platform layouts**: no menciona `renpy.variant` ("pc" vs "small"). GDD pide layout dual.
- ❌ **Themed overlay UI** (phone/tablet con apps grid + nav buttons): pattern temático Politicuck para PhoneOverlay.
- ❌ **Stat change notification queue**: pill flotante con cola "+5 Confianza X" stackeable.
- ❌ **Auto-hide HUD**: pattern para ocultar HUD en cinemáticas/diálogo y mostrarlo en gameplay.
- ❌ **Tabs / sub-menu within a screen**: galería con tabs por NPC necesita pattern.
- ⚠️ **Notify duration hard-coded** (2.5s): debería ser parametrizable.

**Acción**: **MEJORAR**
- Agregar sección **"HUD layout pattern (top/bottom bars)"** con mockup completo.
- Agregar sección **"Responsive layouts (PC vs mobile)"** con `renpy.variant` y dual screens.
- Agregar sección **"Themed overlay UI (phone-style)"** con grid de apps + bottom nav.
- Agregar sección **"Stat change notification queue"** con animaciones secuenciales y cola.
- Agregar pattern **"Auto-hide HUD by context"** (Show/Hide hooks en labels).
- Agregar pattern **"Tabs/sub-menu within a screen"** con SetScreenVariable + condition.

---

## 8. `renpy-rpg.md` (stats RPG con leveling)

**Tiene**:
- `CharacterStats` con HP/ATK/DEF/SPD + level + xp + buffs.
- `gain_xp` con level-up + stat growth.
- `Skill` class con cooldown + mp_cost + `SKILL_CATALOGUE`.
- `Item` class + `ITEM_CATALOGUE` + `Inventory` (add/remove/has/count/all_items).
- `BattleState` turn-based combat con player_attack, skills, enemy_turn.
- Stat checks in dialogue (soft check + `stat_check` helper).
- `stats_hud` screen.

**Aplicabilidad a Politicuck**:
- ✅ **Inventory + Item + ITEM_CATALOGUE**: reutilizable directamente para Politicuck (inventario de regalos/consumibles/docs).
- ✅ **Stat checks in dialogue**: pattern aplica (Politicuck usa `if influencia >= 25: ...`).
- ❌ **CharacterStats (HP/ATK/DEF) + leveling + combat**: NO aplica a Politicuck (no es RPG, no hay combate, no hay leveling). El sistema de tres stats triple (Dinero/Influencia/Imagen) es narrativo, no RPG.

**Gaps vs Politicuck**: 
- ⚠️ Ningún gap REAL que requiera modificar este skill. La pieza "Inventory" se reutiliza. El resto no aplica.
- ⚠️ El skill no documenta explícitamente "cuándo NO usar este skill" — gente podría forzar CharacterStats para life-sim donde no encaja.

**Acción**: **DEJAR_COMO_ESTÁ** con micro-mejora.
- Agregar al inicio una nota corta: **"Cuándo NO usar este skill"** — life-sim / political-sim / VN narrativo: usar `renpy-survival` + sistema custom de stats narrativos en su lugar.

⚠️ **Issue arquitectónico transversal**: `Inventory` está en `renpy-rpg`, pero `renpy-adult-vn`, `renpy-screens` y `renpy-quests` la asumen existente. Consideración para futuro: extraer Inventory a `renpy-inventory.md` o documentar mejor el cross-skill dep.

---

## Resumen consolidado

| Skill | Acción | Prioridad mejora |
|-------|--------|------------------|
| 1. `main-renpy-skill` | MEJORAR | media |
| 2. `renpy-adult-vn` | MEJORAR (mucho) | **alta** |
| 3. `renpy-survival` | MEJORAR | **alta** |
| 4. `renpy-maps` | MEJORAR | media |
| 5. `renpy-quests` | MEJORAR | **alta** |
| 6. `renpy-minigames` | MEJORAR (mucho) | **alta** |
| 7. `renpy-screens` | MEJORAR | media |
| 8. `renpy-rpg` | DEJAR (+ nota corta) | baja |

**Issue transversal**: `Inventory` está repartido. Considerar `renpy-inventory.md` separado en futuro (no en este pase).

**¿Crear skills nuevos?** Por ahora **NO**. Todos los gaps detectados encajan en los 8 existentes. Politicuck-específicos (Chismerío, NTR multi-etapa, paparazzi, etc.) pueden documentarse como **patterns dentro de los skills existentes** manteniendo generalidad.

