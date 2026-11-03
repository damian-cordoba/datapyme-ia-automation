# Video demostrativo

> **Video:** https://youtu.be/U75VL4reE_o (YouTube, no listado) · Grabado el 03/11/2026 con Xbox Game Bar (Windows), **sin voz**.
> Para que se entienda sin narración, entre cada paso aparece un cartel explicativo: [`video/carteles-video.html`](../video/carteles-video.html).

## Qué muestra

| Paso | Pantalla | Qué se ve |
|---|---|---|
| 1 | Cartel | Título del proyecto y herramientas |
| 2 | Diagrama de arquitectura | Las 5 herramientas y cómo se conectan |
| 3 | Airtable | Tabla `Leads` con la columna `Estado` y el catálogo `Servicios` |
| 4 | Formulario público (**trigger**) | Se carga una consulta de prueba (Carolina Vega, Librería Vega) y se envía |
| 5 | n8n (**orquestador**) | La ejecución en curso: validación, clasificación con Claude Haiku y redacción con Claude Sonnet |
| 6 | Slack (**HITL**) | El mensaje de aprobación con el borrador y su versión; se toca *Aprobar y enviar* |
| 7 | Gmail (**resultado final**) | El email recibido; después, el aviso en el hilo de Slack y el lead en `Enviado` |
| 8 | Airtable | Los leads del camino infeliz: `Datos incompletos`, `Error`, `Rechazado por humano`, `Expirado` |
| 9 | Dashboard público | KPIs y tasa de errores |
| 10 | Cartel | Cierre con el enlace al repositorio |

## Seguridad en la grabación

- No se abre ninguna credencial de n8n ni la configuración de la app de Slack.
- El email del cliente de prueba es una cuenta propia; ningún mensaje se envía a clientes reales.
