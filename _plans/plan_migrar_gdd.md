# Plan: Migración GDD Prestige → Dorni (Sandbox VN Político Argentino)

## Decisiones fijadas

- **Concepto**: Many Dorni, aspirante político cornudo. Sátira política argentina.
- **Plataforma**: Ren'Py (multiplataforma).
- **Loop**: Sandbox urbano relacional. Inicio en depto malo.
- **NTR**: Mixto. Esposa cómplice — pide $/objetos/misiones para acceder a otras personas. Cada acceso = unlock.
- **Stats jugador**: **Dinero** ($), **Influencia**, **Imagen**.
- **MVP**: Dorni + esposa + 2 NPCs.
- **Minijuegos MVP**: Blackjack + Memoria. Escalado de $ por nivel jugador.
- **Save/Load**: Ren'Py built-in. No skill propio.

## Skills Tier 1 (crear en orden)

1. `renpy-vn-core` — scaffolding sandbox VN (estructura proyecto, screens base, characters)
2. `renpy-map-navigation` — mapa libre con hotspots clickeables (lugares)
3. `renpy-stats-system` — Dinero + Influencia + Imagen + stats NPC + UI
4. `renpy-quest-system` — misiones por etapas (esposa pide → grindear → desbloquear)
5. `renpy-minigames` — blackjack + memoria, escalado por nivel

## Skills Tier 2 (post-MVP, on-demand)

- `renpy-time-system` (día/noche, agenda NPC)
- `renpy-inventory-shop` (objetos/regalos, tiendas)
- `renpy-relations-npc` (gating eventos por threshold relación)
- `renpy-events-triggers` (eventos por contexto)
- `renpy-adult-gallery` (galería desbloqueable)
- `renpy-debug-menu` (debug testing)

---

## Migración GDD — sección por sección

Iterar con usuario. Cada sección: revisar viejo → proponer nuevo → confirmar → escribir.

### Orden propuesto

| # | Sección GDD | Acción | Estado |
|---|-------------|--------|--------|
| 1 | Concepto | Expandir (tono sátira AR, premisa, antagonistas, gancho) | ✅ completa |
| 2 | Moneda + Stats | Reescribir: $ + Influencia + Imagen, cómo suben/bajan, gating | ✅ completa |
| 3 | Plataforma + Estilo visual | Confirmar Ren'Py + estilo arte (dibujos VN, paleta sátira) | ✅ completa |
| 4 | Core Gameplay Loop | Diseñar loop sandbox (día tipo, decisiones, ciclo $ → unlocks) | ✅ completa |
| 5 | Mapa + Lugares | Listar lugares MVP (depto, casa, casino, congreso, etc.) | ✅ completa |
| 6 | Personajes MVP | Dorni + esposa + 2 NPCs (perfiles, arcos, motivaciones) | ✅ completa |
| 7 | Sistema de misiones | Esposa pide $/objetos/etapas, cada cumplimiento = unlock NPC | ✅ completa |
| 8 | Sistema económico | Fuentes de $ (minijuegos, trabajos, favores), gastos, gating | ✅ completa |
| 9 | Minijuegos | Blackjack + Memoria, escalado por nivel, integración Ren'Py | ✅ completa |
| 10 | Diálogos | Estructura VN, choices que mueven Influencia/Imagen | ✅ completa |
| 11 | Sistema NTR (mixto) | Eventos forzados (trama) vs opt-in (grindeo), consecuencias en Imagen | ✅ completa |
| 12 | Contenido adulto | Gallery, gating por progresión, paquetes por NPC | ✅ completa |
| 13 | Pantallas / UI | Listar screens (mapa, depto, diálogo, minijuego, stats, gallery) | ✅ completa |
| 14 | Modo Debug | Adaptar al nuevo contexto (skip events, max stats, unlock all) | ✅ completa |
| 15 | Scope MVP / Futuro | Reescribir según decisiones nuevas | ✅ completa |

### Migración GDD finalizada el 2026-05-15

Resultado: 18 secciones, 1620 líneas. Estructura coherente con concepto Politicuck. Limpieza completa de restos del concepto fotógrafo (Sesiones, Stats Modelo, Progresión Luna/Clara, Modelo DLC, Sistema guardado).

### Próximo paso

Empezar Skill Tier 1 #1: `renpy-vn-core` (scaffolding sandbox VN). Ver `MEMORY.md` para contexto del proyecto.

### Secciones a BORRAR del GDD viejo

- Sesiones fotográficas completas (modelo, contratación, costos sesión, flujo)
- Stats de la "Modelo" (humor/deseo/cansancio/osadía/belleza/notoriedad)
- Stats del "Fotógrafo" (notoriedad/profesionalismo/cansancio/equipo)
- Canales de venta (JustFans/Prints/Compradores privados)
- Sistema de locación + alquiler
- Negociación de minijuego pre-sesión
- Progresión Luna/Clara
- "La Sombra" como antagonista de memory game
- Sistema guardado (Ren'Py built-in)

### Sistemas reutilizables a extraer (doc aparte `_systems.md`)

- Dificultad universal 0.0–1.0 con función de derivación pura (aplicable a minijuegos nuevos)
- Estructura de minijuego modular (interfaz común de scoring/resultado)
- Modo Debug (concepto, flags)
- Exportar/Importar JSON save (si se desea por encima del save Ren'Py)

---

## Criterio de éxito por paso

Cada sección: usuario confirma contenido propuesto → escribir en `game_design_document.md` reemplazando/ampliando la sección correspondiente.

Al terminar las 15 secciones: GDD coherente con concepto Dorni, sin restos del concepto fotógrafo, listo para empezar a crear skills Tier 1.
