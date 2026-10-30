# Evidencias

Capturas reales tomadas durante la construcción y las pruebas (n8n Cloud 2.45.6, Airtable, Slack). Ninguna es simulada.
Los secretos (tokens, API keys, signing secret) aparecen ocultos en todas las capturas.

## 1. Base de datos (Airtable)

| Archivo | Qué demuestra |
|---|---|
| `airtable-omni-01-prompt.png` | Prompt enviado a Omni AI para generar el esquema |
| `airtable-omni-02-aprobaciones.png` | Omni AI pidiendo aprobación para cada cambio de estructura |
| `airtable-omni-03-tablas-creadas.png` | Resumen de Omni AI: 5 tablas creadas, campos inversos de los vínculos |
| `airtable-omni-04-ap-formula.png` | Corrección pedida a Omni: `Aprobacion_ID` como fórmula |
| `airtable-esquema-leads.png` | Campos y tipos de la tabla `Leads` |
| `airtable-esquema-leads-estados.png` | Las 10 opciones del campo `Estado` (máquina de estados) |
| `airtable-esquema-servicios.png` | Campos de `Servicios` |
| `airtable-esquema-analisis-ia.png` | Campos de `Analisis_IA` |
| `airtable-esquema-aprobaciones.png` | Campos de `Aprobaciones` |
| `airtable-esquema-ejecuciones.png` | Campos de `Ejecuciones` |
| `airtable-servicios.png` | Catálogo de 5 servicios activos (base de conocimiento de la IA) |

## 2. Configuración y credenciales

| Archivo | Qué demuestra |
|---|---|
| `n8n-instancia-version.png` | Instancia n8n Cloud y versión 2.45.6 |
| `n8n-credencial-airtable.png` | Token de Airtable con conexión verificada y dominio restringido a `api.airtable.com` |
| `n8n-credencial-slack.png` | Credencial de Slack con conexión verificada, signing secret cargado y dominio restringido a `slack.com` |
| `n8n-credencial-gmail.png` | Cuenta de Gmail conectada por OAuth2 |
| `n8n-claude-gateway-credits.png` | Nodo de Anthropic usando Gateway credits (sin API key propia, costo $0 en el entorno académico) |
| `n8n-claude-opciones.png` | Opciones disponibles del nodo (System Message, Max Tokens, Prompt Caching, etc.) |
| `slack-scopes.png` | App de Slack con 4 permisos iniciales (mínimo privilegio). Después se agregó `groups:read`: 5 en total |
| `slack-interactivity-request-url.png` | Botones interactivos apuntando a la instancia de n8n |
| `slack-app-en-canal.png` | App agregada al canal `#aprobaciones-comerciales` |

## 3. Construcción del workflow y dashboard

| Archivo | Qué demuestra |
|---|---|
| `n8n-workflow-parte-a-importado.png` | Parte A importada: nodos numerados y nombrados |
| `n8n-nodo-06-mapeo-campos.png` | Mapeo con variables dinámicas (sin datos hardcodeados) |
| `n8n-nodo-26-config-aprobacion.png` | Nodo de aprobación: aprobador restringido, captura de quién responde, expiración |
| `n8n-nodo-40-hilo-slack.png` | Resultado publicado como respuesta en el hilo de Slack |
| `n8n-publicar-version.png` | Publicación del workflow como versión de producción |
| `dashboard-tasa-errores.png` | Gráfico de torta con la tasa de errores (11,11 % al momento de la captura) |
| `dashboard-compartir-web.png` | Publicación del dashboard en la web |
| `dashboard-publico-incognito.png` | Dashboard abierto sin iniciar sesión (ventana de incógnito) |
| `base-publica-incognito.png` | Base abierta en modo solo lectura sin iniciar sesión |

## 4. Pruebas (ver `tests/matriz-pruebas.md`)

| Prueba | Archivos | Resultado |
|---|---|---|
| T01 — Caso normal | `test-01-n8n-caso-normal.png`, `test-01-airtable-lead-pendiente.png` | Lead creado en `Pendiente` |
| T02 — Duplicado | `test-02-n8n-duplicado.png`, `test-02-airtable-ejecuciones.png` | Reenvío bloqueado, sin lead nuevo |
| T03 — Datos incompletos | `test-03-n8n-incompleto.png`, `test-03-airtable-leads.png` | Lead en `Datos incompletos`, errores registrados |
| T05a — Fallo real de la API de Claude | `test-05a-n8n-error-api.png`, `test-05a-n8n-detalle-error.png`, `test-05a-airtable-error-api.png`, `test-05a-airtable-leads-error.png`, `test-05a-airtable-analisis-vacio.png` | 3 intentos, lead en `Error`, sin análisis guardado |
| T05a — Corrección | `test-05a-correccion-sin-temperatura.png` | El nodo responde tras quitar el parámetro rechazado |
| T04 — Procesamiento con IA | `test-04-n8n-ia-ok.png`, `test-04-airtable-analisis.png`, `test-04-airtable-leads.png`, `test-04-ficha-analisis-1/2/3.png` | Análisis y borrador guardados, lead en `Procesado por IA` |
| T05b — Respuesta inválida de la IA (simulada) | `test-05b-n8n-reintento-y-error.png`, `test-05b-airtable-leads.png`, `test-05b-airtable-ejecuciones.png` | 1 reintento, luego `Error` con 2 intentos |
| T06 — Aprobación humana → envío | `test-06-slack-solicitud-aprobacion.png`, `test-06-gmail-email-recibido.png`, `test-06-n8n-ruta-aprobado.png`, `test-06-airtable-lead-enviado.png`, `test-06-airtable-aprobaciones.png` | Email enviado con el texto aprobado; lead `Enviado`; aprobación registrada con la versión |
| T07 — Rechazo humano | `test-07-slack-solicitud.png`, `test-07-slack-rechazado.png`, `test-07-slack-hilo-resultado.png`, `test-07-airtable-aprobaciones.png` | Ningún email; aviso en el hilo de Slack |
| T08 — Sin respuesta (expiración) | `test-08-slack-expirado.png`, `test-08-airtable-lead-expirado.png` | Lead `Expirado`; ningún email |
| T09 — Aprobación sobre una solicitud vencida | `test-09-slack-aprobacion-vencida-409.png` | n8n rechaza el clic (409); ningún email |
| T10 — Producción (formulario público) | `test-10-n8n-ejecucion-produccion.png`, `test-10-gmail-produccion.png` | Ejecución de producción exitosa con la versión v1.0 |
