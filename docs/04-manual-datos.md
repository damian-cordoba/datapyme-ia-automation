# Manual operativo de estructuras de datos

> Estado: **Configurado y validado.** El esquema completo de tablas, campos y estados está en [`02-base-de-datos.md`](02-base-de-datos.md). Este documento explica cómo se generó y qué datos viajan entre las integraciones.

## 1. El "cerebro" relacional en Airtable

### Generación con Omni AI

La base **DataPyme - Consultas Comerciales** se generó con **Airtable Omni AI** a partir de un prompt en lenguaje natural que define las 5 tablas, cada campo con su tipo, las opciones de los campos de selección y los vínculos (prompt completo en [`02-base-de-datos.md` §4](02-base-de-datos.md)).

- Omni pidió aprobación para cada cambio de estructura y eliminó la tabla de ejemplo para que no quedara una tabla aislada (`evidence/airtable-omni-02-aprobaciones.png`, `airtable-omni-03-tablas-creadas.png`).
- Se le pidió una corrección: convertir `Aprobacion_ID` en fórmula (`evidence/airtable-omni-04-ap-formula.png`).
- El resultado se verificó campo por campo contra un checklist (`evidence/airtable-esquema-*.png`).
- El catálogo de servicios se importó desde [`data/servicios.csv`](../data/servicios.csv).

### Tablas vinculadas

```text
Servicios   1 ──< N  Analisis_IA     (servicio sugerido por la IA)
Leads       1 ──< N  Analisis_IA     (un lead puede tener más de un análisis)
Analisis_IA 1 ──< N  Aprobaciones    (cada decisión humana queda registrada)
Leads       1 ──< N  Ejecuciones     (cada corrida del flujo deja un registro)
```

| Tabla | Rol | Quién escribe |
|---|---|---|
| `Leads` | Una fila por consulta; su campo `Estado` es la máquina de estados del proceso | n8n (nodos 03b, 06, 10, 19, 24, 25, 31, 33–36b) |
| `Servicios` | Base de conocimiento que consulta la IA | Persona (catálogo) |
| `Analisis_IA` | Clasificación, borrador, versión (`Hash_Borrador`), modelos y tokens | n8n (nodo 18) |
| `Aprobaciones` | Decisión humana, quién aprobó y versión aprobada | n8n (nodo 29) |
| `Ejecuciones` | Registro de cada corrida: etapa, resultado y error; alimenta el dashboard | n8n (nodos 08, 21, 38) |

**Sincronización:** el flujo es la única fuente de escritura de las tablas operativas. Antes de la acción crítica se **relee** la base (nodos 27a–27b) en lugar de usar datos guardados en memoria; así un cambio hecho en Airtable mientras se esperaba la aprobación se detecta y bloquea el envío.

## 2. Esquemas JSON de transferencia

Los ejemplos usan datos ficticios de las pruebas.

### 2.1 Formulario → n8n (nodo 01)

```json
{
  "Nombre": "Martín Ruiz",
  "Email": "cliente@ejemplo.com",
  "Empresa": "Ferretería del Valle",
  "Mensaje": "Cada mes armamos a mano el reporte de ventas juntando exportaciones del sistema de facturación y planillas de 2 sucursales. Nos lleva dos días. Queremos automatizarlo.",
  "submittedAt": "2026-10-30T12:58:00.000-03:00",
  "formMode": "production"
}
```

### 2.2 Validación de entrada (salida del nodo 02)

```json
{
  "nombre": "Martín Ruiz",
  "email": "cliente@ejemplo.com",
  "email_recibido": "cliente@ejemplo.com",
  "empresa": "Ferretería del Valle",
  "mensaje": "Cada mes armamos a mano el reporte…",
  "fecha_recepcion": "2026-10-30T15:58:00.000Z",
  "es_valido": true,
  "errores": [],
  "errores_texto": "",
  "clave_dedup": "DD-b3d18363-191"
}
```

### 2.3 Clasificación que devuelve Claude Haiku 5.5 (validada en el nodo 14)

```json
{
  "tipo_servicio": "automatizacion_reportes",
  "servicio_sugerido": "Automatización de reportes con Power Query",
  "prioridad": "media",
  "nivel_sugerido": "intermedio",
  "resumen": "Ferretería del Valle quiere automatizar el reporte mensual de ventas que hoy consolida a mano.",
  "datos_faltantes": ["Sistema de facturación y formato de exportación", "Volumen de registros por mes"],
  "motivo": "Necesidad concreta sin urgencia explícita; dos fuentes de datos.",
  "requiere_revision_humana": true
}
```

| Campo | Regla de validación |
|---|---|
| `tipo_servicio` | `sistema_excel`, `dashboard_bi`, `automatizacion_reportes`, `capacitacion` u `otro` |
| `servicio_sugerido` | Nombre exacto de un servicio activo del catálogo, coherente con `tipo_servicio`; `ninguno` si el tipo es `otro` |
| `prioridad` | `alta`, `media` o `baja` |
| `nivel_sugerido` | `basico`, `intermedio` o `completo` |
| `resumen` | Texto de 10 a 200 caracteres |
| `datos_faltantes` | Lista de hasta 5 textos |
| `motivo` | Texto de hasta 300 caracteres |
| `requiere_revision_humana` | Siempre `true` |

### 2.4 Borrador que devuelve Claude Sonnet 5.5 (validado en el nodo 17)

```json
{
  "asunto": "Automatizar el reporte mensual de ventas de Ferretería del Valle",
  "cuerpo": "Hola Martín:\n\nGracias por escribirnos…\n\nSaludos cordiales,\nEquipo DataPyme Consultores"
}
```

Reglas: asunto de 5 a 120 caracteres; cuerpo de 300 a 1.800 caracteres; sin precios, monedas ni marcadores entre corchetes. Luego se calcula `Hash_Borrador` (por ejemplo `HB-d8eef5cb-1210`), la huella del texto exacto.

### 2.5 Respuesta de la aprobación en Slack (nodo 26)

```json
{
  "data": {
    "approved": true,
    "respondedAt": "2026-10-30T16:00:00.000Z",
    "channel": "C0…",
    "messageId": "1793…123456",
    "responder": { "id": "U0…", "name": "…", "username": "…", "email": "…" }
  }
}
```

Si nadie responde en 45 minutos, el nodo continúa **sin** `data.approved` y el nodo 28 lo interpreta como `expirado`. El `messageId` se usa como hilo de Slack para publicar el resultado.

### 2.6 Decisión verificada (salida del nodo 28 → registro en `Aprobaciones`)

```json
{
  "ruta": "enviar",
  "decision": "aprobado",
  "problemas_texto": "",
  "hash_aprobado": "HB-d8eef5cb-1210",
  "aprobador": "usuario (U0…)",
  "fecha_decision": "2026-10-30T16:00:00.000Z",
  "slack_ts": "1793…123456",
  "lead_id": "rec…",
  "analisis_id": "rec…",
  "email": "cliente@ejemplo.com",
  "asunto": "Automatizar el reporte mensual…",
  "cuerpo": "Hola Martín: …"
}
```

`ruta` puede ser `enviar`, `rechazado`, `expirado` o `bloqueado`. El email se arma con el `asunto` y el `cuerpo` **releídos de Airtable**, no con los datos guardados en memoria.

### 2.7 Envío por Gmail (nodo 32) → lead `Enviado` (nodo 33)

```json
{ "id": "19a3…", "threadId": "1a18d8a9fb1b…", "labelIds": ["SENT"] }
```

`id` se guarda en `Leads.Message_ID` y `threadId` en `Leads.Thread_ID`: el hilo queda mapeado para continuar la conversación.

### 2.8 Registro de ejecución (nodos 08, 21 y 38 → `Ejecuciones`)

```json
{
  "Execution_ID": "12-HITL",
  "Lead": ["rec…"],
  "Workflow": "principal",
  "Etapa": "Aprobado por humano → email enviado",
  "Resultado": "ok",
  "Mensaje_Error": "Gmail id … | thread … | aprobado por … | versión HB-…",
  "Fecha": "2026-10-30T18:49:52.000Z"
}
```

Sufijos de `Execution_ID`: sin sufijo = recepción (Parte A), `-IA` = procesamiento con IA (Parte B), `-HITL` = aprobación y envío (Parte C). Así una misma ejecución deja hasta tres registros, uno por etapa.
