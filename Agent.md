
## Directivas

### Comunicación
- **PRIMERO**: verificar si skill `caveman` está instalado (`/home/<user>/.claude/skills/caveman/`).
  - Si instalado → activar con `/caveman full` y mantener activo toda la sesión.
  - Si NO instalado → advertir al equipo inmediatamente y usar forma breve y concisa (sin fluff, sin cortesías, fragmentos OK).
- Respuestas siempre cortas y directas. Sin saludos, sin relleno.

### Pre-análisis (obligatorio antes de ejecutar)
1. Leer tareas completas: las tareas que tienen el "(ok)".
2. Hacer preguntas necesarias para clarificar contexto y objetivos.
3. Dar sugerencias concretas de mejora antes de planificar.
4. Esperar confirmación del usuario.
5. Si se pide criticidad y/o en el momento del análisis: ser director sin ser irrespetuoso.

### Planificación
- trabajar los planes en el directorio '_plans'. Sino existe, crearlo.
- Crear archivo `plan_<nombre_tarea>.md` con pasos numerados, detallados y ordenados.
- Cada paso debe incluir: archivo(s) afectado(s), acción concreta, criterio de éxito.
- Revisar plan con usuario antes de ejecutar.

### Ejecución (Agent con Sonnet más reciente)
- Usar `Agent` tool con `model: "sonnet"` (equivale a `claude-sonnet-4-6`, modelo más reciente).
- Al inicio: crear tareas con `TaskCreate` por cada paso del plan.
- Al completar cada paso: llamar `TaskUpdate` con estado `completed` **inmediatamente** (no en batch).
- Si un paso falla: reportar error exacto, proponer solución, esperar aprobación antes de continuar.
- Al finalizar: resumen de 2 líneas máximo — qué cambió, qué sigue.

---
