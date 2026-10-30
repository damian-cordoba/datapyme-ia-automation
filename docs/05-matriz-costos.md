# Estrategia de optimización de costos y recursos

> Estado: **Calculado con tokens medidos en una ejecución real** (registro **AN-1**, prueba T04).
> Precios por millón de tokens según la tabla de precios de Anthropic, consultada en octubre de 2026 a través de [eesel: Haiku 5.5](https://www.eesel.ai/blog/claude-haiku-5-5-pricing) y [eesel: Sonnet 5.5](https://www.eesel.ai/blog/claude-sonnet-5-5-pricing) (ambas citan platform.claude.com/docs/en/about-claude/pricing; verificar ahí antes de usar estas cifras en producción).

## 1. Precios usados

| Modelo | Entrada (USD / 1M) | Salida (USD / 1M) | Batch entrada / salida | Lectura de caché |
|---|---|---|---|---|
| Claude Haiku 5.5 (prompts de hasta 100K tokens) | 0,10 | 0,50 | 0,05 / 0,25 | 0,01 |
| Claude Sonnet 5.5 | 2,00 | 10,00 | 1,00 / 5,00 | 0,20 |

En Haiku y Sonnet 5.5 la salida incluye los tokens de razonamiento (*thinking*). El modelo los usa aunque no se pidan, por eso se subió `maxTokens` (1.000 para clasificar y 1.500 para redactar).

## 2. Matriz de decisión: qué modelo hace cada tarea

| Tarea | Tipo de trabajo | Modelo elegido | Por qué | Alternativa descartada |
|---|---|---|---|---|
| Validar campos, detectar duplicados, calcular la huella del borrador | Mecánica, determinística | **Sin IA** (nodos Code e IF) | Costo cero, resultado exacto y repetible | Pedírselo a un LLM: más caro y no determinístico |
| Clasificar la consulta (tipo, prioridad, nivel, datos faltantes) | Extracción estructurada sobre un texto corto, con valores cerrados | **Claude Haiku 5.5** | 20 veces más barato que Sonnet; la salida se valida contra listas cerradas y el catálogo, y si falla se reintenta | Sonnet: mayor calidad que esta tarea no necesita |
| Redactar el email al cliente | Texto que lee un tercero, con tono y precisión | **Claude Sonnet 5.5** | Mejor redacción; un error de tono tiene costo de reputación | Haiku: más barato pero con más riesgo de tono o de inventar datos |
| Revisión final antes del envío | Juicio humano | **Persona en Slack (HITL)** | Única acción irreversible del sistema | Envío automático sin revisión |
| Reprocesar en bloque (por ejemplo, reclasificar todo el historial si cambia el catálogo) | Masiva, sin urgencia | **Message Batches API** (Haiku) | 50 % de descuento; la respuesta puede esperar horas | Llamadas en tiempo real |
| Prefijo fijo de la clasificación (instrucciones + catálogo) | Repetido en cada consulta | **Prompt Caching** cuando el prefijo supere el mínimo cacheable | Lectura de caché = 10 % del precio de entrada | Hoy el prefijo es corto, así que el ahorro sería marginal (ver §5) |

## 3. Costo medido por consulta

Tokens registrados en **AN-1** (suma de las dos llamadas): **3.262 de entrada** y **1.268 de salida**. Airtable guarda el total y no el desglose por modelo, por eso se calcula un rango y una estimación con un supuesto explícito.

| Escenario | Cálculo | Costo por consulta |
|---|---|---|
| Todo con Haiku (piso teórico) | 3.262 × 0,10 + 1.268 × 0,50 (por millón) | **USD 0,00096** |
| **Diseño implementado: Haiku + Sonnet** (supuesto: entrada repartida 50/50; salida 25 % clasificación y 75 % redacción) | Haiku: 1.631 × 0,10 + 317 × 0,50 · Sonnet: 1.631 × 2 + 951 × 10 | **≈ USD 0,0131** |
| Todo con Sonnet (techo) | 3.262 × 2 + 1.268 × 10 | **USD 0,0192** |

**Ahorro del diseño implementado frente a usar Sonnet para todo: ≈ 32 %.** El ahorro es moderado porque la redacción, que es la parte más cara, se mantiene en Sonnet a propósito: es el texto que lee el cliente.

Además, **las consultas que no pasan la validación no llegan a la IA** (datos incompletos y duplicados): en las pruebas, 2 de las 11 ejecuciones se resolvieron sin gastar tokens.

## 4. Proyección mensual

| Volumen | Haiku + Sonnet (implementado) | Todo Sonnet | Haiku + Sonnet con reproceso mensual en Batch |
|---|---|---|---|
| 100 consultas / mes | ≈ USD 1,31 | USD 1,92 | + ≈ USD 0,016 por reclasificar las 100 en batch |
| 300 consultas / mes | ≈ USD 3,93 | USD 5,76 | + ≈ USD 0,047 |
| 1.000 consultas / mes | ≈ USD 13,10 | USD 19,20 | + ≈ USD 0,155 |

Reproceso en batch: solo la clasificación con Haiku, unos 1.600 tokens de entrada y 300 de salida por consulta, al 50 % de descuento: 1.600 × 0,05 + 300 × 0,25 = USD 0,000155 por consulta.

**Costo en el entorno académico: USD 0.** La IA se pagó con los Gateway credits incluidos en la prueba de n8n Cloud (saldo inicial USD 2,00), sin API key propia. Airtable, Slack y Gmail se usaron en sus planes gratuitos o de prueba.

## 5. Otras decisiones de costo

| Recurso | Decisión | Efecto |
|---|---|---|
| Trigger | Formulario (webhook), no consultas periódicas | Cero ejecuciones en vacío |
| `maxTokens` | Tope de 1.000 (clasificación) y 1.500 (redacción) | Acota el costo máximo por llamada |
| Reintentos | 3 por llamada y 1 de validación, con tope | Un fallo no genera gasto ilimitado |
| Longitud del borrador | 80 a 220 palabras pedidas; máximo validado de 1.800 caracteres | Controla los tokens de salida, que son los más caros |
| Prompt Caching | No activado: el prefijo de la clasificación (instrucciones + catálogo de 5 servicios) es corto, y el ahorro sobre un modelo que cuesta USD 0,10 por millón sería de centavos | Se recomienda activarlo si el catálogo crece o si se pasa la clasificación a un modelo más caro |
