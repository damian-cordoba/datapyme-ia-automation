# Auditoría final: requisitos vs. evidencia

> Fecha: 30/10/2026. **COMPLETO** = implementado y verificado con evidencia · **PENDIENTE** = falta hacerlo.

## 1. Consigna

| Requisito | Implementación | Evidencia | Estado |
|---|---|---|---|
| Caso de uso de negocio con lenguaje natural | Consultas comerciales de una consultora ficticia | `docs/01-caso-de-uso.md` | COMPLETO |
| Al menos 4 tecnologías integradas | n8n, Airtable, Claude, Slack y Gmail (5) | `evidence/n8n-credencial-*.png`, T06 | COMPLETO |
| Base de datos con campos de estado | `Leads.Estado` con 10 estados, incluidos *Pendiente*, *Procesado por IA* y *Aprobado por humano* | `evidence/airtable-esquema-leads-estados.png` | COMPLETO |
| Relaciones entre tablas | 4 vínculos entre 5 tablas | `docs/02-base-de-datos.md`, `evidence/airtable-esquema-*.png` | COMPLETO |
| Trigger inteligente | Formulario (webhook); sin consultas periódicas ni trigger de Airtable | Nodo 01, T10 | COMPLETO |
| El flujo procesa con IA desde la DB y entrega un output sin intervención manual | Lee el catálogo desde Airtable y corre desde el formulario público hasta el email | T10 (ejecución de producción) | COMPLETO |
| Motor de IA con la variable de respuesta mapeada y Max Tokens limitado | Se lee `content[].text` (ignorando el bloque *thinking*); `maxTokens` de 1.000 y 1.500 | Nodos 13, 14, 16, 17 | COMPLETO |
| Prompt dinámico con variables del sistema | Los prompts insertan nombre, empresa, mensaje, catálogo y clasificación | `prompts/`, `evidence/test-04-ficha-analisis-*.png` | COMPLETO |
| Rutas de error ante datos faltantes y fallos de API | Datos incompletos, duplicados, fallo de API, respuesta inválida, fallo de Gmail | T03, T05a, T05b | COMPLETO |
| Registro del error si falla la API de IA | `Ejecuciones` con etapa y mensaje del proveedor; lead en `Error` | `evidence/test-05a-airtable-error-api.png` | COMPLETO |
| HITL: espera aprobación antes de contactar al cliente | Slack *Send and Wait*; aprobar, rechazar, expirar y aprobación vencida | T06, T07, T08, T09 | COMPLETO |
| Slack/Gmail: hilos mapeados | Resultado publicado en el hilo del mensaje de aprobación; `Thread_ID` y `Message_ID` de Gmail guardados | `evidence/test-07-slack-hilo-resultado.png`, `evidence/test-06-airtable-lead-enviado.png` | COMPLETO |
| Nodos nombrados, variables dinámicas, sin datos hardcodeados | 53 nodos numerados y descriptivos; los únicos valores fijos son reglas de negocio (estados, canal, aprobador) | Workflow JSON | COMPLETO |
| Filtro contra bucles infinitos | Sin trigger de Airtable, máquina de estados y reintentos con tope | `docs/03-arquitectura-y-flujo.md` §4 | COMPLETO |
| Comparación de tipos correctos en los filtros | Booleano con booleano, texto con texto; campos Number en Airtable | Nodos IF | COMPLETO |
| Al menos 5 ejecuciones, incluido el camino infeliz | 11 ejecuciones reales | `tests/matriz-pruebas.md` | COMPLETO |
| PDF con el diagrama de arquitectura | `docs/arquitectura.pdf` | — | COMPLETO |
| JSON del flujo | `workflow/datapyme-consultas-comerciales.json`, revisado sin secretos | — | COMPLETO |
| Enlace de solo lectura a la base | Verificado en una ventana de incógnito | `evidence/base-publica-incognito.png` | COMPLETO |
| Capturas de evidencia | Más de 50 capturas reales con índice | `evidence/README.md` | COMPLETO |
| Video demo de 3 minutos sin credenciales | Guion listo | `docs/07-guion-video.md` | **PENDIENTE** (grabar y subir) |
| Repositorio de GitHub | Archivos listos | — | **PENDIENTE** (crear y subir) |

## 2. Rúbrica (5 criterios de 20 %)

| Criterio | Dónde está | Estado |
|---|---|---|
| Mapa de arquitectura | `docs/arquitectura.pdf` §2–3 · `docs/img/diagrama-1/2/3.png` | COMPLETO |
| Manual operativo de datos (Omni AI + tablas vinculadas + JSON de transferencia) | `docs/arquitectura.pdf` §4 · `docs/02-base-de-datos.md` · `docs/04-manual-datos.md` | COMPLETO |
| Optimización de costos (matriz de decisión por modelo) | `docs/arquitectura.pdf` §6 · `docs/05-matriz-costos.md` | COMPLETO |
| Seguridad y resiliencia (minimización + Error Handlers + HITL) | `docs/arquitectura.pdf` §7–8 · `docs/06-seguridad-y-resiliencia.md` | COMPLETO |
| Dashboard de control (enlace público con KPIs y tasa de errores) | Enlace en el README · `evidence/dashboard-publico-incognito.png` | COMPLETO |
