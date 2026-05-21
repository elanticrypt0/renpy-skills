# Plan: Auditoría y mejora de skills Ren'Py existentes

## Contexto

Existen 8 archivos de skills genéricos en el repo (borradores sin testear). Política decidida:
- **Auditar y mejorar los 8 existentes** (no crear nuevos por ahora).
- **Mantenerlos genéricos / reusables** (no hardcodear "Politicuck"). Politicuck es el caso de prueba que define los gaps.
- Solo crear skill nuevo si aparece un gap genuino que no encaja en ninguno de los 8.

## Skills a auditar (8)

| # | Skill | Cubre en GDD Politicuck | Gaps esperados a chequear |
|---|-------|-------------------------|---------------------------|
| 1 | `main-renpy-skill` | Entrypoint, orquestador, decide cuándo usar cada sub-skill | ¿Frontmatter completo? ¿Tabla de decisión bien definida? ¿Trigger words actualizados? |
| 2 | `renpy-adult-vn` | Sandbox VN, gallery, NPC scheduling, scene unlocking, content gating | NTR-multi-etapa, paparazzi mini, contenido prohibido (TOS-safe), achievements cross-save |
| 3 | `renpy-maps` | Mapa con hotspots, navegación, sub-zonas | Hotspots animados, gating por stat, sub-zonas (casino/congreso/quinta) |
| 4 | `renpy-minigames` | Arquitectura modular, dificultad universal 0-1 | Blackjack con buy-in fijo + cap, Memoria solitario con tiempo, cap semanal anti-farm |
| 5 | `renpy-quests` | Misiones con etapas, journal | NTR multi-etapa con regalos, misiones procedurales (templates), deadlines suaves |
| 6 | `renpy-rpg` | Stats con leveling, equipment | Politicuck NO es RPG: ¿adaptar para "stats triples opacos"? ¿O dejar generic y crear pattern aparte? |
| 7 | `renpy-screens` | UI/HUD, screens custom | HUD dual horizontal/mobile, PhoneOverlay, stat-change notifications |
| 8 | `renpy-survival` | Stats supervivencia, time/day cycle | 3 slots/día, días de semana, energía como modifier (no limitador), gastos semanales |

## Proceso por skill

Para cada skill, 3 pasos:

1. **Leer completo** + mapear contenido a tabla `lo_que_tiene`
2. **Comparar vs needs del GDD** + listar `gaps` específicos
3. **Decidir acción**:
   - `mejorar`: agregar pattern al skill (manteniendo generalidad)
   - `dejar_como_está`: ya cubre needs adecuadamente
   - `gap_separado`: gap es muy específico, considerar nuevo skill al final

## Orden de auditoría

1. **main-renpy-skill** (primero — define orquestación, otros referencian a él)
2. **renpy-adult-vn** (más cerca de Politicuck, integra todo)
3. **renpy-survival** (slots/días/energía — base del sandbox)
4. **renpy-maps** (hotspots + sub-zonas)
5. **renpy-quests** (misiones)
6. **renpy-minigames** (blackjack + memoria)
7. **renpy-screens** (UI/HUD)
8. **renpy-rpg** (decidir si aplica o no a Politicuck)

## Entregables

- `_plans/audit_skills_findings.md`: doc con tabla de hallazgos por skill (lo que tiene, gaps, acción decidida)
- 0-8 archivos `.md` modificados con mejoras agregadas
- 0-N skills nuevos (solo si aparece gap genuino que no encaja en ningún existente)

## Criterio de éxito

- Cada skill auditado tiene su entrada en `audit_skills_findings.md`
- Cada gap detectado tiene acción decidida y ejecutada (o documentada como pendiente con justificación)
- Los skills modificados siguen siendo genéricos (no hardcodean "Politicuck", usan ejemplos como "VN sandbox adulto" / "stats con tensión opuesta" / etc.)
