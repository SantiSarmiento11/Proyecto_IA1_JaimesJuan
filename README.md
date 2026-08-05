# 🤖 AgendaBot Services

AgendaBot Services es un chatbot desarrollado como proyecto académico utilizando **n8n Community Edition**, **Telegram** y **Google Sheets** para gestionar citas, tareas, recordatorios, hábitos y reportes de productividad de forma automatizada.

---

# 📌 Objetivo

El objetivo del proyecto es ofrecer un asistente conversacional que permita organizar información personal y automatizar tareas sin utilizar plataformas de pago ni servicios que requieran tarjeta de crédito.

---

# 🛠 Tecnologías utilizadas

- n8n Community Edition
- Telegram Bot
- Google Sheets
- JavaScript (Code Nodes de n8n)

---

# 📂 Arquitectura

El proyecto está compuesto por:

- Bot de Telegram
- Workflow principal de n8n
- Base de datos en Google Sheets (AgendaBot_DB)

Google Sheets contiene las siguientes hojas:

- CITAS
- TAREAS
- HABITOS
- LISTAS
- ITEMS_LISTA
- USUARIO
- LOGS
- SESSIONS

---

# 🚀 Funcionalidades

## 📅 Agenda

- Crear citas
- Consultar agenda
- Reprogramar citas
- Cancelar citas
- Marcar citas como completadas

---

## ✅ Tareas

- Crear tareas
- Consultar tareas
- Cambiar estado
- Eliminar tareas

---

## 🔔 Recordatorios

- Crear recordatorios
- Consultar recordatorios
- Eliminar recordatorios
- Notificaciones automáticas

---

## 💪 Hábitos

- Registrar hábitos
- Consultar hábitos
- Marcar cumplimiento
- Editar hábitos

---

## 📝 Listas

- Crear listas
- Agregar elementos
- Marcar elementos completados
- Eliminar elementos

---

## 📊 Reportes

El sistema genera diferentes reportes desde Telegram:

- Reporte diario
- Reporte semanal
- Productividad de tareas
- Cumplimiento de hábitos
- Estado de citas
- Productividad por usuario

---

# ⭐ Nueva funcionalidad: Productividad por Usuario

Se agregó un nuevo módulo dentro del menú **Reportes**.

Ruta:

```
Menú principal

6. Reportes

6. Productividad por usuario
```

Al seleccionar esta opción el bot:

- Lee la información desde Google Sheets.
- Consulta las hojas:
  - CITAS
  - TAREAS
  - LOGS
  - USUARIO
- Agrupa la información por usuario.
- Calcula métricas individuales.
- Calcula estadísticas generales.
- Genera automáticamente un reporte en Telegram.

---

# 📈 Métricas calculadas

Para cada usuario se calcula:

- Total de citas
- Citas completadas
- Citas canceladas
- Total de tareas
- Tareas completadas
- Tareas pendientes
- Total de interacciones con el bot

También se calcula un resumen general:

- Usuario más activo
- Total de citas registradas
- Total de tareas registradas
- Total de interacciones del bot

---

# 💬 Ejemplo del reporte

```
📊 Reporte de productividad (AgendaBot)

Resumen general

• Usuario más activo
• Total citas registradas
• Total tareas registradas
• Total interacciones

Detalle por usuario

@sebastian

• Citas: 3
• Completadas: 2
• Canceladas: 1

• Tareas: 5
• Hechas: 4
• Pendientes: 1

• Interacciones: 18

¿Qué deseas hacer ahora?

1. Volver al menú Reportes
2. Volver al menú principal
```

---

# 📊 Base de datos

Toda la información se almacena en Google Sheets.

Las principales hojas utilizadas son:

### CITAS

- id_cita
- fecha
- hora
- nombre
- motivo
- canal
- estado
- telegram_user

### TAREAS

- id_tarea
- titulo
- prioridad
- estado
- fecha_objetivo
- telegram_user

### LOGS

- timestamp
- telegram_user
- pantalla
- opcion_elegida
- resultado

### USUARIO

- telegram_user
- nombre
- rol
- permitido
- recordatorios_activos
- zona_horaria

---

# 📂 Estructura del proyecto

```
Proyecto_IA_Nivel1_ApellidoNombre/

README.md

docs/
    AgendaBot.md

workflows/
    AgendaBot_Workflow.json

evidencias/
    menu_reportes.png
    reporte_productividad.png
```

---

# 📸 Evidencias

Las evidencias del funcionamiento del sistema se encuentran en la carpeta **evidencias/**.

Incluyen:

- Menú principal
- Menú Reportes
- Reporte de productividad en Telegram
- Workflow implementado en n8n
- Base de datos en Google Sheets

---

# 👨‍💻 Autor

**Juan Sebastián Jaimes Rolón**

Proyecto desarrollado como parte del curso de Inteligencia Artificial utilizando n8n Community Edition, Telegram y Google Sheets.