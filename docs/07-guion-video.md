# Guion del video demostrativo (≈ 3 minutos)

> Estado: **Guion listo.** El video se graba con OBS o con la grabación de pantalla de Windows (**Win + Alt + R** con la Xbox Game Bar).

## Antes de grabar (checklist)

- [ ] Workflow v3 **publicado** en n8n (botón *Published* en verde).
- [ ] Pestañas abiertas en este orden: **1)** formulario (Production URL), **2)** n8n con el workflow, **3)** Slack en `#aprobaciones-comerciales`, **4)** Gmail, **5)** Airtable (tabla Leads), **6)** dashboard público.
- [ ] **Ocultar secretos:** no abrir ninguna credencial en n8n ni la página *OAuth & Permissions* de Slack. Las credenciales no aparecen en ningún paso del guion.
- [ ] Cerrar la barra de favoritos del navegador (**Ctrl + Shift + B**) y cualquier otra ventana (chats, streams).
- [ ] Datos de la consulta preparados para copiar y pegar (abajo).
- [ ] Probar el audio del micrófono.

Consulta para el video:

- **Nombre:** Carolina Vega
- **Email:** tu Gmail
- **Empresa:** Librería Vega
- **Mensaje:** *Tenemos dos locales y llevamos las ventas en planillas separadas. Queremos ver en un solo tablero qué libros se venden más y cuándo reponer.*

## Guion

| Tiempo | Pantalla | Qué mostrar | Qué decir (sugerido) |
|---|---|---|---|
| 0:00–0:20 | Diagrama de arquitectura (PDF o `docs/img/diagrama-1.png`) | Las 5 herramientas | "DataPyme es una consultora ficticia que recibe consultas comerciales. Automaticé su primera respuesta con n8n, Airtable, Claude, Slack y Gmail, con una regla: ningún email sale sin aprobación humana." |
| 0:20–0:40 | Airtable | Tabla `Leads` con su columna `Estado`, y la tabla `Servicios` | "Airtable es la memoria del sistema: 5 tablas relacionadas, generadas con Omni AI. El catálogo de servicios es lo único que la IA puede ofrecer." |
| 0:40–1:00 | Formulario → n8n | Completar y enviar el formulario. Pasar a n8n → **Executions** y abrir la ejecución en curso | "El cliente completa el formulario público. El flujo arranca solo: valida los datos, controla duplicados y crea el lead." |
| 1:00–1:30 | n8n (lienzo de la ejecución) | Recorrer con el mouse las partes A, B y C. Detenerse en los nodos 13 y 16 | "Claude Haiku clasifica la consulta en JSON y Claude Sonnet redacta la respuesta. Las dos salidas se validan; si la IA falla o responde mal, hay reintentos con tope y el error queda registrado." |
| 1:30–2:00 | Slack | Mensaje con el borrador, la versión y los botones. Tocar **Aprobar y enviar**. Mostrar la respuesta en el hilo | "Acá el flujo se detiene. Solo yo puedo aprobar, y el mensaje muestra la versión exacta del borrador. Si alguien lo cambiara en Airtable, el envío se bloquearía." |
| 2:00–2:20 | Gmail → Airtable | El email recibido y el lead en `Enviado` con `Thread_ID` | "Recién después de la aprobación sale el email, y el lead queda en Enviado con el hilo de Gmail registrado." |
| 2:20–2:45 | Airtable → tabla `Leads` | Señalar los leads `Datos incompletos`, `Error`, `Rechazado por humano` y `Expirado` | "Probé 11 ejecuciones reales: datos incompletos, duplicados, un error real de la API, una respuesta inválida de la IA, un rechazo, una aprobación vencida… Todas terminaron en el estado esperado y ningún email salió sin aprobación." |
| 2:45–3:00 | Dashboard público | KPIs y gráfico de torta con la tasa de errores | "Todo se monitorea en este dashboard público, con la tasa de errores. Gracias." |

## Después de grabar

1. Revisá el video completo y verificá que no aparezca ningún token, key ni contraseña.
2. Subilo a YouTube como **No listado** (o a Google Drive con *Cualquier persona con el enlace*).
3. Pegá el enlace en el README, en la sección **Video demostrativo**.
