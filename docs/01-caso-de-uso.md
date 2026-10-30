# Paso 1 — Caso de uso

> Estado: **Implementado y validado** (ver [`tests/matriz-pruebas.md`](../tests/matriz-pruebas.md)).

## Negocio (ficticio)

**DataPyme Consultores** — consultora *ficticia* de sistemas de gestión a medida para pymes argentinas: sistemas en Excel (VBA, Power Query, Power Pivot), dashboards en Power BI y automatización de reportes. Todos los datos del negocio y de los clientes usados en este proyecto son ficticios.

## Problema de negocio

Las consultas comerciales llegan por el formulario web con información desordenada: a veces describen bien el problema, otras veces les falta el rubro, el volumen de datos o el plazo. Hoy cada consulta se lee, se clasifica y se responde a mano:

- Las respuestas tardan y dependen de que alguien revise el formulario.
- La calidad de la primera respuesta es despareja (a veces genérica, a veces sin pedir los datos que faltan).
- No queda registro de qué consultas llegaron, cómo se clasificaron ni cuáles se respondieron.

## Proceso a automatizar

Recepción → clasificación → redacción de la primera respuesta → **aprobación humana** → envío al cliente → registro.

## Usuario objetivo

- **Responsable comercial** de la consultora: aprueba o rechaza cada respuesta desde Slack.
- **Cliente potencial**: recibe la respuesta por email.

## Objetivo del sistema

Que cada consulta válida quede clasificada, con una respuesta redactada y lista para aprobar, sin trabajo manual previo; y que **ningún email salga al cliente sin aprobación explícita**.

## Flujo funcional

| Etapa | Qué pasa | Quién lo hace |
|---|---|---|
| Entrada | El cliente completa el formulario (nombre, email, empresa, mensaje) | Cliente |
| Validación | Se verifican campos obligatorios y formato del email | n8n |
| Memoria | Se busca si el email ya consultó antes (historial y duplicados) | n8n + Airtable |
| Clasificación IA | Tipo de servicio, prioridad, nivel sugerido, datos faltantes, resumen → JSON | Claude (modelo económico) |
| Redacción IA | Borrador de respuesta basado **solo** en el catálogo de servicios | Claude (modelo de mayor calidad) |
| Aprobación | Se envía el borrador a Slack con botones Aprobar / Rechazar y el flujo espera | Responsable comercial |
| Salida | Si se aprueba, se envía el email al cliente por Gmail | n8n + Gmail |
| Registro | Estado final, decisión y resultado del envío quedan en Airtable | n8n + Airtable |

## Decisiones que toma la IA (y las que no)

La IA **clasifica y redacta**; **no decide** el envío.

- `tipo_servicio`: sistema_excel · dashboard_bi · automatizacion_reportes · capacitacion · otro
- `prioridad`: alta · media · baja
- `nivel_sugerido`: basico · intermedio · completo
- `datos_faltantes`: lista de información que conviene pedirle al cliente
- `resumen`: una línea

La respuesta al cliente **no cotiza precios**: propone una llamada y pide los datos faltantes. Así evitamos que el modelo invente montos.

## Acción crítica y punto HITL

**Acción crítica:** enviar un email a un tercero (el cliente).
**Control:** el envío solo se ejecuta si existe una aprobación humana registrada para ese lead y para esa versión exacta del borrador.

## Tecnologías (4 categorías + 1 canal adicional)

| Categoría | Herramienta |
|---|---|
| Orquestador | n8n Cloud |
| Base de datos / memoria | Airtable |
| IA | Anthropic Claude |
| Canal de salida | Gmail (cliente) |
| Canal de aprobación | Slack (interno) |

## Cómo se comprueba que funciona

No hay datos reales de negocio, así que el éxito se mide con **métricas técnicas observables en las pruebas**:

| Métrica | Cómo se mide |
|---|---|
| Ejecuciones completadas | Registros en la tabla Ejecuciones con estado final |
| Tasa de JSON válido de la IA | Respuestas que pasan la validación / total |
| Consultas incompletas detenidas | Casos con datos faltantes que no llegan a la IA |
| Envíos sin aprobación | Debe ser **0** |
| Duplicados bloqueados | Reenvíos del mismo formulario que no se reprocesan |
| Tiempo hasta la aprobación | Diferencia entre "Esperando aprobación" y la decisión |

## Fuera de alcance

- Cotización automática de precios.
- Seguimiento posterior (drips) o WhatsApp.
- RAG vectorial: el catálogo es chico y estructurado; se recupera desde Airtable (ver justificación en la documentación de arquitectura).
