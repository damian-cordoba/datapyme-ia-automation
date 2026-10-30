# Paso 2 — El "cerebro": base de datos en Airtable

> Estado: **Configurado y verificado** (29/10/2026). La base se generó con Omni AI y se verificó campo por campo contra el checklist del §6. Evidencias: `evidence/airtable-omni-*.png` y `evidence/airtable-esquema-*.png`.

## 1. Modelo de datos

```text
Servicios 1 ──< N Analisis_IA          (servicio sugerido por la IA)
Leads     1 ──< N Analisis_IA          (un lead puede tener más de un análisis si se regenera)
Analisis_IA 1 ──< N Aprobaciones       (cada decisión humana queda registrada)
Leads     1 ──< N Ejecuciones          (cada corrida del flujo deja un log)
```

Ninguna tabla queda aislada: todo cuelga de `Leads`, y `Servicios` es la base de conocimiento que la IA consulta.

> Convención: nombres de campos **sin tildes ni espacios** para evitar errores de mapeo en n8n.

## 2. Tablas y campos

### Tabla `Leads` — una fila por consulta recibida

| Campo | Tipo | Uso |
|---|---|---|
| `Lead_ID` *(primario)* | Fórmula: `"LD-" & {Nro}` | Identificador legible |
| `Nro` | Autonumber | Base del identificador |
| `Nombre` | Single line text | Del formulario |
| `Email` | Email | Del formulario |
| `Empresa` | Single line text | Del formulario (opcional) |
| `Mensaje` | Long text | Del formulario |
| `Fecha_Recepcion` | Date (con hora) | La escribe n8n |
| `Estado` | Single select (ver §3) | Máquina de estados del proceso |
| `Clave_Dedup` | Single line text | Hash de email + mensaje → evita procesar duplicados |
| `Intentos_IA` | Number (entero) | Intentos de clasificación usados |
| `Thread_ID` | Single line text | Hilo de Gmail del envío |
| `Message_ID` | Single line text | ID del email enviado |
| `Fecha_Envio` | Date (con hora) | Momento del envío confirmado |
| `Analisis_IA` | Link → `Analisis_IA` | Relación (campo inverso creado por Airtable) |
| `Ejecuciones` | Link → `Ejecuciones` | Relación (campo inverso creado por Airtable) |

### Tabla `Servicios` — catálogo (base de conocimiento de la IA)

| Campo | Tipo | Uso |
|---|---|---|
| `Servicio` *(primario)* | Single line text | Nombre del servicio |
| `Codigo` | Single select: `sistema_excel`, `dashboard_bi`, `automatizacion_reportes`, `capacitacion` | Lo que devuelve la IA al clasificar |
| `Descripcion` | Long text | Qué es |
| `Incluye` | Long text | Alcance |
| `Plazo_Estimado` | Single line text | Orientativo |
| `Activo` | Checkbox | Solo los activos se envían a la IA |
| `Analisis_IA` | Link → `Analisis_IA` | Relación inversa (creada por Airtable) |

### Tabla `Analisis_IA` — resultado de la IA por lead

| Campo | Tipo | Uso |
|---|---|---|
| `Analisis_ID` *(primario)* | Fórmula: `"AN-" & {Nro}` | Identificador |
| `Nro` | Autonumber | — |
| `Lead` | Link → `Leads` | Relación |
| `Tipo_Servicio` | Single select: `sistema_excel`, `dashboard_bi`, `automatizacion_reportes`, `capacitacion`, `otro` | Clasificación |
| `Prioridad` | Single select: `alta`, `media`, `baja` | Clasificación |
| `Nivel_Sugerido` | Single select: `basico`, `intermedio`, `completo` | Clasificación |
| `Servicio_Sugerido` | Link → `Servicios` | Relación con el catálogo |
| `Resumen` | Single line text | Una línea |
| `Datos_Faltantes` | Long text | Lo que conviene pedirle al cliente |
| `Borrador_Asunto` | Single line text | Asunto del email |
| `Borrador_Cuerpo` | Long text | Cuerpo del email |
| `Hash_Borrador` | Single line text | Huella de la versión → la aprobación vale solo para esta versión |
| `Modelo_Clasificacion` | Single line text | Trazabilidad / costos |
| `Modelo_Redaccion` | Single line text | Trazabilidad / costos |
| `Tokens_Entrada` | Number | Costos |
| `Tokens_Salida` | Number | Costos |
| `Fecha_Analisis` | Date (con hora) | — |
| `Aprobaciones` | Link → `Aprobaciones` | Relación |

### Tabla `Aprobaciones` — decisiones humanas (HITL)

| Campo | Tipo | Uso |
|---|---|---|
| `Aprobacion_ID` *(primario)* | Fórmula: `"AP-" & {Nro}` | Identificador |
| `Nro` | Autonumber | — |
| `Analisis` | Link → `Analisis_IA` | Qué versión se aprobó |
| `Decision` | Single select: `aprobado`, `rechazado`, `expirado` | Resultado |
| `Aprobador` | Single line text | Quién decidió (usuario de Slack) |
| `Hash_Aprobado` | Single line text | Debe coincidir con `Hash_Borrador` para poder enviar |
| `Fecha_Decision` | Date (con hora) | — |
| `Comentario` | Long text | Motivo del rechazo (opcional) |

### Tabla `Ejecuciones` — registro de cada corrida (alimenta el dashboard)

| Campo | Tipo | Uso |
|---|---|---|
| `Execution_ID` *(primario)* | Single line text | ID de ejecución de n8n |
| `Lead` | Link → `Leads` | Relación (vacío si el lead no llegó a crearse) |
| `Workflow` | Single select: `principal`, `errores` | Origen del registro |
| `Etapa` | Single line text | Nodo/etapa donde terminó |
| `Resultado` | Single select: `ok`, `error`, `bloqueado`, `duplicado`, `incompleto` | Para KPIs |
| `Mensaje_Error` | Long text | Detalle técnico |
| `Fecha` | Date (con hora) | — |

## 3. Estados del lead (`Leads.Estado`)

```text
Pendiente → En procesamiento → Procesado por IA → Esperando aprobacion
                     │                                   ├→ Aprobado por humano → Enviado
                     │                                   ├→ Rechazado por humano   (terminal)
                     │                                   └→ Expirado               (terminal)
                     ├→ Datos incompletos   (terminal)
                     └→ Error               (terminal, tras agotar reintentos)
```

| Estado | Significado | ¿Terminal? |
|---|---|---|
| `Pendiente` | Recibido, sin procesar | No |
| `En procesamiento` | El flujo lo tomó (bloquea una segunda toma) | No |
| `Datos incompletos` | Faltan campos obligatorios o el email es inválido | Sí |
| `Procesado por IA` | Clasificación y borrador guardados | No |
| `Esperando aprobacion` | Enviado a Slack, el flujo está en pausa | No |
| `Aprobado por humano` | Hay una aprobación válida para el hash actual | No |
| `Rechazado por humano` | No se envía nada al cliente | Sí |
| `Expirado` | Nadie respondió a tiempo | Sí |
| `Enviado` | Gmail confirmó el envío | Sí |
| `Error` | Falló de forma definitiva; detalle en `Ejecuciones` | Sí |

**Regla anti-bucle y anti-duplicado:** el flujo solo avanza si el estado actual es el esperado para ese paso. Los estados terminales nunca se reprocesan. Además, Airtable **no** dispara el flujo (no hay trigger de Airtable): el disparador es el formulario de n8n, así que las escrituras del propio flujo en la base no pueden volver a activarlo.

## 4. Prompt para Omni AI

Copiar y pegar tal cual en Omni (ver §5):

```text
Creá una base llamada "DataPyme - Consultas Comerciales" para gestionar consultas comerciales de una consultora de sistemas Excel y Power BI para pymes. Los nombres de tablas y campos deben ir exactamente como los escribo, sin tildes. No agregues tablas ni campos extra, y no cargues datos de ejemplo.

Tabla "Leads":
- Nro: autonumber
- Lead_ID: campo primario, fórmula "LD-" & {Nro}
- Nombre: texto de una línea
- Email: email
- Empresa: texto de una línea
- Mensaje: texto largo
- Fecha_Recepcion: fecha con hora
- Estado: selección única con estas opciones exactas: Pendiente, En procesamiento, Datos incompletos, Procesado por IA, Esperando aprobacion, Aprobado por humano, Rechazado por humano, Expirado, Enviado, Error
- Clave_Dedup: texto de una línea
- Intentos_IA: número entero
- Thread_ID: texto de una línea
- Message_ID: texto de una línea
- Fecha_Envio: fecha con hora

Tabla "Servicios":
- Servicio: campo primario, texto de una línea
- Codigo: selección única con opciones: sistema_excel, dashboard_bi, automatizacion_reportes, capacitacion
- Descripcion: texto largo
- Incluye: texto largo
- Plazo_Estimado: texto de una línea
- Activo: casilla de verificación

Tabla "Analisis_IA":
- Nro: autonumber
- Analisis_ID: campo primario, fórmula "AN-" & {Nro}
- Lead: vínculo a la tabla Leads
- Tipo_Servicio: selección única con opciones: sistema_excel, dashboard_bi, automatizacion_reportes, capacitacion, otro
- Prioridad: selección única con opciones: alta, media, baja
- Nivel_Sugerido: selección única con opciones: basico, intermedio, completo
- Servicio_Sugerido: vínculo a la tabla Servicios
- Resumen: texto de una línea
- Datos_Faltantes: texto largo
- Borrador_Asunto: texto de una línea
- Borrador_Cuerpo: texto largo
- Hash_Borrador: texto de una línea
- Modelo_Clasificacion: texto de una línea
- Modelo_Redaccion: texto de una línea
- Tokens_Entrada: número entero
- Tokens_Salida: número entero
- Fecha_Analisis: fecha con hora

Tabla "Aprobaciones":
- Nro: autonumber
- Aprobacion_ID: campo primario, fórmula "AP-" & {Nro}
- Analisis: vínculo a la tabla Analisis_IA
- Decision: selección única con opciones: aprobado, rechazado, expirado
- Aprobador: texto de una línea
- Hash_Aprobado: texto de una línea
- Fecha_Decision: fecha con hora
- Comentario: texto largo

Tabla "Ejecuciones":
- Execution_ID: campo primario, texto de una línea
- Lead: vínculo a la tabla Leads
- Workflow: selección única con opciones: principal, errores
- Etapa: texto de una línea
- Resultado: selección única con opciones: ok, error, bloqueado, duplicado, incompleto
- Mensaje_Error: texto largo
- Fecha: fecha con hora
```

> Los vínculos crean automáticamente el campo inverso en la otra tabla (por ejemplo, `Leads` recibe un campo `Analisis_IA`). Es lo esperado; solo renombralo si hace falta para que coincida con §2.

## 5. Paso a paso en Airtable

1. **Crear la cuenta** en airtable.com (plan Free) y entrar al home.
2. **Crear la base con Omni:** *+ Create* → elegí tu workspace → opción de construir con Omni (describir lo que querés) → pegá el prompt de §4 → confirmá el plan que propone Omni.
   - Si Omni muestra un plan con tablas o campos distintos a los pedidos, pedile que lo corrija **antes** de confirmar.
3. **Revisar lo que generó** contra el checklist de §6. Omni puede elegir mal un tipo de campo; corregilo a mano (clic en la flecha del encabezado del campo → *Edit field*).
4. **Importar el catálogo:** abrí la tabla `Servicios` → botón de agregar/importar → *CSV file* → subí `data/servicios.csv` → elegí **agregar a la tabla existente** y mapeá cada columna a su campo.
5. **Captura para el repositorio:** una imagen de cada tabla mostrando sus campos (encabezados visibles) y la tabla `Servicios` con los 5 registros. Guardalas como `evidence/airtable-<tabla>.png`.
6. **Mandame las capturas** (o una sola de cada tabla) y verifico el esquema antes de pasar al paso 3.

> Todavía **no** hace falta crear el token de API ni compartir la base: eso va en el paso 3 y en la entrega.

## 6. Checklist de verificación

- [ ] Existen exactamente 5 tablas: `Leads`, `Servicios`, `Analisis_IA`, `Aprobaciones`, `Ejecuciones`
- [ ] Los primarios son: fórmula en `Leads`, `Analisis_IA` y `Aprobaciones`; texto en `Servicios` y `Ejecuciones`
- [ ] `Email` es tipo **Email** (no texto)
- [ ] `Intentos_IA`, `Tokens_Entrada`, `Tokens_Salida` son **Number** (no texto) → permite comparar número contra número en los filtros
- [ ] Las fechas son **Date con hora**
- [ ] `Estado` tiene las 10 opciones escritas exactamente como en §3
- [ ] Los 4 vínculos existen: Analisis_IA→Leads, Analisis_IA→Servicios, Aprobaciones→Analisis_IA, Ejecuciones→Leads
- [ ] `Servicios` tiene 5 registros activos
- [ ] Omni **no** agregó registros de ejemplo en las demás tablas (si lo hizo, borrarlos)
