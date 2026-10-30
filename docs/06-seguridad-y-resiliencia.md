# Seguridad, resiliencia y control humano

> Estado: **Implementado y probado** (pruebas T02, T03, T05a, T05b, T06, T07, T08 y T09 en [`tests/matriz-pruebas.md`](../tests/matriz-pruebas.md)).

## 1. Minimización y protección de datos

| Medida | Implementación |
|---|---|
| **Minimización** | El formulario pide solo 4 datos: nombre, email, empresa (opcional) y mensaje. No se piden teléfono, documento ni datos de facturación. |
| **Datos inválidos no se guardan** | Si el email tiene un formato inválido, se guarda vacío (el valor recibido queda solo en el detalle del error). |
| **La IA ve lo mínimo** | Claude recibe nombre, empresa, mensaje y catálogo. **No recibe el email** del cliente. |
| **Prompt injection** | Los prompts delimitan la consulta con etiquetas (`<consulta_cliente>`) y ordenan tratarla solo como datos, nunca como instrucciones. Además, la salida se valida contra valores cerrados, así que una respuesta manipulada no pasa la validación. |
| **Sin precios inventados** | La IA no cotiza; el nodo 17 rechaza borradores con precios o monedas. |
| **Datos ficticios** | Negocio y clientes ficticios. Las pruebas se enviaron solo a una casilla propia. |
| **Transparencia (IA Act, riesgo limitado)** | El sistema redacta borradores que una persona revisa antes de enviar. No decide sobre derechos de personas. |

## 2. Credenciales y mínimo privilegio

| Servicio | Tipo de credencial | Alcance |
|---|---|---|
| Airtable | Token personal | 3 scopes (`data.records:read`, `data.records:write`, `schema.bases:read`), **1 sola base**, dominio restringido a `api.airtable.com` |
| Slack | Token de bot + *signing secret* | 5 scopes (`chat:write`, `channels:read`, `groups:read`, `users:read`, `users:read.email`); n8n verifica la firma de cada clic; dominio restringido a `slack.com` |
| Gmail | OAuth2 gestionado por n8n | Revocable desde la cuenta de Google |
| Claude | Gateway credits de n8n Cloud | Sin API key propia que pueda filtrarse |

- Todas las credenciales viven **solo en el gestor de credenciales de n8n**. El JSON exportado contiene únicamente el nombre y el ID interno de cada credencial: se verificó que no incluye tokens, keys ni secretos.
- El repositorio incluye [`.env.example`](../.env.example) con los nombres de las variables, sin valores.
- Las capturas de evidencia muestran los secretos ocultos.

## 3. Rutas de error (resiliencia)

| Falla | Detección | Acción | Estado final del lead | Prueba |
|---|---|---|---|---|
| Datos incompletos o email inválido | Nodo 02 | Se registra el detalle; no se llama a la IA | `Datos incompletos` | T03 |
| Envío duplicado del formulario | Nodos 04–05 (`Clave_Dedup`) | No se crea un lead nuevo; se registra `duplicado` | (sin lead nuevo) | T02 |
| Catálogo de servicios vacío | Nodo 12 | Error explícito | `Error` | — |
| Fallo de la API de Claude (caída, *rate limit*, parámetros) | `retryOnFail` (3 intentos, 3 s) + salida de error del nodo | Se registra el mensaje del proveedor | `Error` | **T05a** (fallo real) |
| Respuesta de la IA inválida o inconsistente | Nodo 14 | 1 reintento; si vuelve a fallar, se descarta | `Error` | **T05b** |
| Borrador con precios, marcadores o longitud fuera de rango | Nodo 17 | Se descarta | `Error` | Lógica probada con datos simulados |
| Nadie aprueba a tiempo | Límite de espera de 45 min | No se envía nada | `Expirado` | **T08** |
| Clic tardío sobre una aprobación vencida | n8n rechaza el callback (409) | No se envía nada | `Expirado` | **T09** |
| Borrador editado después de pedir la aprobación | Nodo 28 (`Hash_Borrador` recalculado) | Envío bloqueado | `Error` | Lógica probada con datos simulados |
| Envío ya realizado o lead fuera de estado | Nodo 28 (`Message_ID`, `Estado`) | Envío bloqueado | `Error` | Lógica probada con datos simulados |
| Fallo de Gmail | `retryOnFail` (2 intentos, 5 s) + salida de error | Se registra el error y se avisa en Slack | `Error` | — |

Cada camino termina en un **estado final explícito** y en un registro en `Ejecuciones` con etapa, resultado y detalle. Así la tasa de errores del dashboard se calcula con datos reales.

## 4. Human-in-the-loop (HITL)

**Acción crítica:** enviar un email a un tercero.

Controles implementados:

1. **La IA no puede autorizar nada.** El envío depende solo de la decisión registrada por Slack (`data.approved`).
2. **Aprobador autorizado.** Solo el usuario de Slack configurado puede responder; a cualquier otro, Slack le muestra el aviso de no autorizado y el flujo sigue esperando. *(Configurado; no se probó con un segundo usuario porque el workspace tiene uno solo.)*
3. **Verificación del origen.** n8n valida la firma (*signing secret*) de cada clic, así que nadie puede falsificar una aprobación llamando a la URL.
4. **Versión aprobada = versión enviada.** El mensaje de Slack muestra `Hash_Borrador`; antes de enviar se relee el borrador desde Airtable y se recalcula la huella. Si alguien cambió el texto, el envío se bloquea.
5. **Sin doble envío.** Antes de enviar se verifica que el lead siga en `Esperando aprobacion` y no tenga `Message_ID`. Slack, además, quita los botones después de la primera decisión.
6. **Vencimiento.** Las solicitudes expiran a los 45 minutos, y una aprobación tardía es rechazada (T09).
7. **Trazabilidad.** Cada decisión queda en `Aprobaciones` con decisión, aprobador, fecha y versión aprobada, vinculada al análisis y al lead.

Resultados observados: aprobación → email enviado (T06); rechazo → ningún email (T07); sin respuesta → ningún email (T08); aprobación vencida → rechazada (T09).

## 5. Monitoreo

- **Dashboard público** (interfaz de Airtable): total de ejecuciones, ejecuciones con error, emails enviados, rechazos, ejecuciones por resultado y tasa de errores en porcentaje. Enlace en [`enlaces.md`](enlaces.md).
- **Historial de n8n** (*Executions*): detalle nodo por nodo de cada corrida.
- **Slack:** cada solicitud de aprobación recibe en su hilo el resultado final.
