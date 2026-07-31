# AgendaBot — Documentación técnica

## 1. Introducción

AgendaBot es un bot conversacional en Telegram que permite agendar citas, gestionar tareas, hábitos y recordatorios sin depender de plataformas de pago. La lógica vive en un workflow de n8n (Community Edition) y los datos se almacenan en Google Sheets, sin entrenamiento de modelos, embeddings ni RAG.

## 2. Arquitectura general

```
Telegram (usuario) ──▶ Telegram Trigger (n8n)
                          │
                          ▼
                  Extraer entrada (Code)
                          │
                          ▼
              Leer sesión (Google Sheets: SESSIONS)
                          │
                          ▼
              Armar objeto sesión (Code)
                          │
                          ▼
          Router (lógica principal) (Code) ── switch/case por pantalla + paso
                          │
          ┌───────────────┼───────────────────┐
          ▼               ▼                    ▼
  Guardar sesión    Registrar log        Switch Acción
  (SESSIONS)         (LOGS)                    │
                                   ┌────────────┼─────────────┬──────────────┐
                                   ▼            ▼              ▼              ▼
                            Preparar cita  Preparar tarea  Preparar hábito  Preparar recordatorio
                                   │            │              │              │
                                   ▼            ▼              ▼              ▼
                            Guardar en    Guardar en      Guardar en      Guardar en
                              CITAS         TAREAS          HABITOS       RECORDATORIOS
                          │
                          ▼
              Enviar respuesta Telegram
```

El **Router** es un único nodo Code que centraliza toda la máquina de estados: recibe la pantalla actual (`pantalla_actual`), el paso del wizard (`paso_actual`) y los datos parciales (`datos_parciales`, JSON serializado) desde la hoja `SESSIONS`, procesa el texto entrante y devuelve la siguiente pantalla, el siguiente paso, los datos actualizados y, si corresponde, una `accion` (por ejemplo `GUARDAR_CITA`) que el nodo `Switch Acción` usa para decidir en qué hoja escribir.

## 3. Modelo de datos (Google Sheets — documento `AgendaBot_DB`)

### Implementado en el workflow actual

**SESSIONS** — estado conversacional por usuario (upsert por `telegram_user`)
| Columna | Descripción |
|---|---|
| telegram_user | ID de Telegram del usuario (clave) |
| pantalla_actual | Pantalla/estado actual del wizard |
| paso_actual | Paso actual dentro de esa pantalla |
| datos_parciales | JSON con los datos capturados hasta el momento |
| timestamp_ultima_interaccion | Fecha/hora del último mensaje |

**LOGS** — registro de cada interacción (append)
| Columna | Descripción |
|---|---|
| timestamp | Fecha/hora del evento |
| telegram_user | Usuario que interactuó |
| pantalla | Pantalla desde la que se eligió la opción |
| opcion_elegida | Texto/número enviado por el usuario |
| resultado | Acción resuelta por el router (o `NINGUNA`) |

**CITAS**
| Columna | Descripción |
|---|---|
| id_cita | `CITA-` + timestamp |
| fecha | YYYY-MM-DD |
| hora | HH:MM (24h) |
| nombre | Nombre del cliente |
| motivo | Motivo de la cita |
| canal | Presencial / Virtual / Llamada |
| estado | `programada` al crear |
| creado_por | telegram_user |
| timestamp_creacion | Fecha/hora de creación |

**TAREAS**
| Columna | Descripción |
|---|---|
| id_tarea | `TAREA-` + timestamp |
| titulo | Título de la tarea |
| prioridad | Alta / Media / Baja |
| estado | pendiente / en_progreso / completada / cancelada |
| fecha_objetivo | YYYY-MM-DD o `SIN` |
| creado_por | telegram_user |

**HABITOS**
| Columna | Descripción |
|---|---|
| id_habito | `HABITO-` + timestamp |
| nombre | Nombre del hábito |
| frecuencia | Diario / Semanal / Mensual |
| hora_recordatorio | HH:MM |
| estado | activo / inactivo |

**RECORDATORIOS** *(hoja añadida por el workflow; no estaba en el listado original del Artículo 4)*
| Columna | Descripción |
|---|---|
| ID | `REC-` + timestamp |
| DESCRIPCION | Mensaje del recordatorio |
| FECHA_LIMIT | YYYY-MM-DD |
| HORA | HH:MM |
| ESTADO | `pendiente` por defecto |
| PRIORIDAD | `Media` por defecto |
| MEDIO | `Telegram` |
| NOTIFICADO | SI / NO |

### Definidas en la especificación, sin implementar todavía

- **LISTAS** (`id_lista`, `nombre_lista`, `tipo`, `creado_por`)
- **ITEMS_LISTA** (`id_item`, `id_lista`, `item`, `estado`)
- **USUARIOS** (`telegram_user`, `nombre`, `rol`, `permitido`)

Estas tres hojas deben crearse igualmente en `AgendaBot_DB` para cumplir el Artículo 4, aunque el router todavía no las lee ni las escribe (las opciones 5 y 8 del menú principal responden con mensajes de placeholder o de solo lectura desde Sheets).

## 4. Mapa de menús

### Menú principal (pantalla `MENU_PRINCIPAL`)
```
0. Ayuda
1. Agenda (citas)
2. Tareas
3. Recordatorios
4. Hábitos
5. Listas          → "🚧 Próximamente"
6. Reportes        → "🚧 Próximamente"
7. Configuración   → "🚧 Próximamente"
8. Administrador
```
Sugerencia por defecto: opción 1 (Agenda). Cualquier entrada fuera de `0-8` dispara el mensaje de opción inválida estándar (Artículo 7) y repite el menú.

### Menú Agenda (`MENU_AGENDA`)
```
1. Agendar una nueva cita   → wizard CITA_WIZARD (6 pasos)
2. Consultar tu agenda      → AGENDA_CONSULTAR
3. Reprogramar una cita     → AGENDA_REPROGRAMAR
4. Cancelar una cita        → AGENDA_CANCELAR
5. Marcar cita completada   → AGENDA_COMPLETAR
9. Volver al menú principal
```

### Menú Tareas (`MENU_TAREAS`)
```
1. Crear tarea nueva        → wizard TAREA_WIZARD (3 pasos)
2. Ver tareas pendientes    → lee hoja TAREAS y filtra por usuario/estado
3. Cambiar estado de tarea  → TAREA_ESTADO
9. Volver al menú principal
```

### Menú Recordatorios (`MENU_RECORDATORIOS`)
```
1. Crear recordatorio       → wizard RECORDATORIO_WIZARD (3 pasos)
2. Ver mis recordatorios    → lee hoja RECORDATORIOS
9. Volver al menú principal
```

### Menú Hábitos (`MENU_HABITOS`)
```
1. Crear nuevo hábito           → wizard HABITO_WIZARD (3 pasos)
2. Ver mis hábitos              → lee hoja HABITOS y filtra por usuario
3. Activar/Desactivar hábito    → HABITO_TOGGLE
9. Volver al menú principal
```

### Panel Administrador (`MENU_ADMIN`)
```
1. Ver usuarios       → mensaje de referencia a Google Sheets (no consulta USUARIOS aún)
2. Ver logs recientes → mensaje de referencia a Google Sheets (no consulta LOGS aún)
9. Volver al menú principal
```

En todas las pantallas de submenú, `9` cancela/vuelve, cumpliendo el principio "el bot siempre ofrece salida".

## 5. Flujos guiados (wizards)

### 5.1 Agendar nueva cita — 6 pasos (`CITA_WIZARD`)
| Paso | Pregunta | Validación |
|---|---|---|
| 1. PASO1_FECHA | Fecha (YYYY-MM-DD) | Regex de formato + no puede ser fecha pasada |
| 2. PASO2_HORA | Hora (HH:MM, 24h) | Regex `00-23:00-59` |
| 3. PASO3_NOMBRE | Nombre completo | Mínimo 2 caracteres |
| 4. PASO4_MOTIVO | Motivo | Mínimo 2 caracteres |
| 5. PASO5_CANAL | 1. Presencial / 2. Virtual / 3. Llamada | Debe ser 1, 2 o 3 |
| 6. PASO6_CONFIRMAR | Resumen + 1. Confirmar / 2. Editar / 3. Cancelar | — |

Al confirmar, se genera `id_cita` (`CITA-` + últimos 6 dígitos del timestamp), se dispara la acción `GUARDAR_CITA` y el flujo pasa a `POST_CITA` con las opciones "Volver a Agenda" / "Ir al menú principal". Escribir `9` en cualquier paso cancela y vuelve al menú Agenda.

### 5.2 Consultar agenda (`AGENDA_CONSULTAR`)
Pide una fecha (o `HOY`) y responde con las citas encontradas para esa fecha. *Estado actual: el nodo de router ya valida la fecha, pero la búsqueda contra la hoja `CITAS` no está cableada — siempre responde "no se encontraron citas". Falta conectar un nodo de lectura de `CITAS` filtrado por fecha.*

### 5.3 Reprogramar cita (`AGENDA_REPROGRAMAR`)
Pide ID de cita (`CITA-XXXXXX`) → nueva fecha (válida y futura) → nueva hora → confirmación. Genera la acción `ACTUALIZAR_CITA`. *Falta el nodo que aplique el `update` sobre la fila real en `CITAS`.*

### 5.4 Cancelar cita (`AGENDA_CANCELAR`)
Pide ID de cita → confirmación explícita (doble check, "esta acción no se puede deshacer") → acción `CANCELAR_CITA`. *Falta el nodo que actualice `estado` a `cancelada` en `CITAS`.*

### 5.5 Marcar cita como completada (`AGENDA_COMPLETAR`)
Pide ID de cita → confirmación → acción `COMPLETAR_CITA`. *Falta el nodo que actualice `estado` a `completada` en `CITAS`.*

### 5.6 Crear tarea — 3 pasos (`TAREA_WIZARD`)
Título (mín. 2 caracteres) → prioridad (1 Alta / 2 Media / 3 Baja) → fecha objetivo (YYYY-MM-DD o `SIN`). Genera `id_tarea` y dispara `GUARDAR_TAREA`.

### 5.7 Cambiar estado de tarea (`TAREA_ESTADO`)
ID de tarea (`TAREA-XXXXXX`) → nuevo estado (1 Pendiente / 2 En progreso / 3 Completada / 4 Cancelada) → acción `ACTUALIZAR_TAREA`. *Falta el nodo que aplique el `update` real sobre `TAREAS`.*

### 5.8 Crear recordatorio — 3 pasos (`RECORDATORIO_WIZARD`)
Mensaje → fecha (válida y futura) → hora. Genera `id_recordatorio` y dispara `GUARDAR_RECORDATORIO`.

### 5.9 Crear hábito — 3 pasos (`HABITO_WIZARD`)
Nombre → frecuencia (1 Diario / 2 Semanal / 3 Mensual) → hora de recordatorio. Genera `id_habito`, estado `activo`, dispara `GUARDAR_HABITO`.

### 5.10 Activar/Desactivar hábito (`HABITO_TOGGLE`)
ID de hábito → 1. Activar / 2. Desactivar → acción `ACTUALIZAR_HABITO`. *Falta el nodo que aplique el `update` real sobre `HABITOS`.*

En todos los wizards, cualquier respuesta inválida repite el paso actual con el mensaje de error específico (nunca avanza con datos incorrectos), y `9` cancela desde cualquier punto.

## 6. Automatizaciones (Artículo 11)

| Automatización requerida | Estado |
|---|---|
| Router principal por pantalla y opción numérica | ✅ Implementado (`Router (lógica principal)`) |
| Flujo guiado de agendamiento | ✅ Implementado (6 pasos, con validaciones) |
| Flujo de tareas con estados | ✅ Implementado (creación + cambio de estado) |
| Resumen diario por Telegram | ❌ No implementado — falta un Schedule Trigger + nodo de lectura de `CITAS`/`TAREAS`/`RECORDATORIOS` del día + envío por Telegram |
| Registro automático de logs | ✅ Implementado (`Registrar log (LOGS)` en cada interacción) |

## 7. Validaciones (Artículo 12)

| Validación requerida | Estado |
|---|---|
| Opción válida según menú | ✅ Regex por pantalla + mensaje estándar de opción inválida |
| Fecha y hora correctas | ✅ `esFechaValida()` y `esHoraValida()` con regex + comparación de fecha |
| No permitir agendar en el pasado | ✅ `esFechaValida()` compara contra la fecha actual |
| Evitar doble reserva | ❌ No implementado — no hay verificación de choque de horario en `CITAS` antes de guardar |
| Confirmación antes de guardar | ✅ Paso de confirmación explícito en el wizard de citas (y confirmaciones sí/no en cancelar/completar/reprogramar) |
| Control de permisos por rol | ❌ No implementado — la hoja `USUARIOS` no se consulta; cualquier usuario de Telegram accede a todas las opciones, incluida Administrador |

## 8. Comunicación humanizada (Artículo 5)

Cada respuesta del router sigue el patrón: contexto breve → opciones numeradas → indicación de cómo salir (`9. Cancelar` / `9. Volver`). El mensaje de bienvenida (`/start` o sesión nueva) saluda, explica el propósito del bot y muestra el menú principal con la sugerencia de la opción 1, tal como exige el Artículo 6.

## 9. Pruebas (Artículo 13) — pendiente

El workflow soporta la ejecución de las pruebas exigidas, pero no hay evidencia registrada todavía. Checklist sugerido para la carpeta `evidencias/`:

- [ ] 30 pruebas de navegación por menús (capturas de cada transición de pantalla)
- [ ] 10 agendamientos completos de extremo a extremo (captura del wizard + fila creada en `CITAS`)
- [ ] 10 errores controlados (opción inválida, fecha pasada, hora inválida, ID inexistente, etc.)
- [ ] 10 pruebas de recordatorios (creación y listado)
- [ ] 10 pruebas de permisos — bloqueadas hasta implementar el control de rol descrito en la sección 7

Cada prueba debe quedar respaldada por su fila correspondiente en `LOGS` y una captura de pantalla de la conversación en Telegram.

## 10. Brechas frente a la especificación (resumen)

Para llegar al cumplimiento total de las Capítulos III, VII, VIII y IX falta:

1. Crear las hojas `LISTAS`, `ITEMS_LISTA` y `USUARIOS` en `AgendaBot_DB` y cablear el menú de Listas y el control de permisos.
2. Conectar lectura/actualización real de `CITAS` para consultar, reprogramar, cancelar y completar citas (hoy el router confirma la acción pero no siempre persiste el cambio).
3. Conectar lectura/actualización real de `TAREAS` para el cambio de estado (`TAREA_ESTADO`).
4. Conectar lectura/actualización real de `HABITOS` para activar/desactivar (`HABITO_TOGGLE`).
5. Añadir un Schedule Trigger para el resumen diario por Telegram.
6. Añadir validación de choque de horario (doble reserva) antes de guardar una cita.
7. Registrar las evidencias de las 70 pruebas exigidas en `evidencias/`.
