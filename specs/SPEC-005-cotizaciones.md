# SPEC-005: Tool Calling y Motor Determinista de Cotizaciones

> Implementa RF-04 y RF-05. Resuelve directamente el riesgo R-03 de la matriz de riesgos del informe (alucinación de precios): el LLM nunca tiene, en ningún punto del flujo, la capacidad técnica de escribir un precio en el documento final.

## Contexto y objetivo

El documento original del proyecto dependía de que el LLM "no alucinara" un precio, mitigado solo por una instrucción de prompt. Esta spec reemplaza esa dependencia por una garantía de arquitectura: el modelo únicamente identifica *qué* se cotiza; el precio se resuelve siempre mediante código determinista contra la base de datos del cliente.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-005-01 | El LLM debe poder invocar una herramienta `solicitar_cotizacion` que devuelve exclusivamente identificadores de catálogo (SKU) y cantidades — el esquema de la herramienta no incluye ningún campo de precio. |
| RF-005-02 | El backend debe resolver cada SKU contra la tabla de precios vigente del tenant; ningún precio puede originarse en la salida del LLM. |
| RF-005-03 | El sistema debe generar un PDF con la identidad visual del tenant a partir de los ítems resueltos. |
| RF-005-04 | El sistema debe despachar el PDF por WhatsApp y marcar el lead como "Cotizado". |
| RF-005-05 | Si un SKU mencionado por el LLM no existe o está inactivo en el catálogo del tenant, esa línea se rechaza explícitamente y se informa al usuario — nunca se completa la cotización con un ítem inventado. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-005-01 | El precio de cualquier línea de una cotización debe ser trazable, en todo momento, a un registro real de la tabla de precios vigente al momento de su generación. |
| RNF-005-02 | La generación y despacho del PDF es idempotente (hereda SPEC-001): un reintento no produce una segunda cotización ni un segundo envío. |

## Modelo de Datos

```sql
catalog_items
  id            uuid PK
  tenant_id     uuid FK
  sku           text
  name          text
  unit_price    numeric NOT NULL
  currency      text
  active        boolean
  updated_at    timestamptz

quotes
  id                uuid PK
  tenant_id         uuid FK
  data_subject_id   uuid FK
  conversation_id   uuid FK
  status            enum('generada','despachada','fallida')
  pdf_url           text
  total_amount      numeric
  created_at        timestamptz

quote_line_items
  id                    uuid PK
  quote_id              uuid FK
  catalog_item_id       uuid FK
  quantity              integer
  unit_price_at_time    numeric   -- snapshot del precio real usado; trazable aunque el precio cambie después
```

## Contratos

```
Herramienta expuesta al LLM: solicitar_cotizacion(items: [{ sku, quantity }])
  — el esquema NUNCA incluye un campo de precio ni de descuento.
  — precondición: el tenant debe tener catalog_items activos; si no los
    tiene, esta herramienta no se habilita en la invocación al LLM.

POST /internal/v1/quotes
  body: { tenant_id, data_subject_id, conversation_id, items: [{ sku, quantity }] }
  1. Valida cada sku contra catalog_items activos del tenant.
     → sku inexistente/inactivo: se excluye de la cotización, se registra
       para informar al usuario cuáles ítems no se pudieron cotizar.
  2. Calcula el total con los precios reales vigentes en este momento.
  3. Crea quote + quote_line_items, con unit_price_at_time congelado.
  4. Encola generación de PDF (cola quote-generation, SPEC-001), con
     idempotency_key = conversation_id + hash(items).
  → 201 { quote_id, status: "generada" }

Worker de la cola quote-generation:
  5. Renderiza la plantilla con la marca del tenant.
  6. Genera el PDF, lo almacena, actualiza quotes.pdf_url.
  7. Despacha por WhatsApp (reutiliza el canal de despacho de SPEC-002).
  8. Marca quotes.status = "despachada" y el lead como "Cotizado".
```

## Flujo

1. Dentro del pipeline conversacional (SPEC-004), el LLM detecta intención de cierre e invoca `solicitar_cotizacion` con SKUs y cantidades.
2. El backend resuelve precios reales (nunca los provistos por el modelo, porque nunca existen) y crea la cotización.
3. El PDF se genera y despacha de forma asíncrona (SPEC-001), con el precio de cada línea ya congelado en el momento de creación de la cotización, no en el momento de generación del PDF.
4. El lead queda marcado como "Cotizado" con trazabilidad completa del origen de cada precio.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| El LLM menciona un SKU o nombre de producto que no existe en el catálogo del tenant (alucinación de producto) | Esa línea se rechaza; se informa al usuario qué ítems sí se pudieron cotizar y cuáles no — nunca se inventa un precio para completar la cotización. |
| El precio de un ítem cambia entre la creación de la cotización y la generación asíncrona del PDF | Se usa el precio congelado en `unit_price_at_time` al momento de creación (paso 2), no el vigente al momento de generar el PDF — evita inconsistencia entre lo que el sistema calculó y lo que el documento muestra. |
| Tenant sin catálogo configurado | La herramienta `solicitar_cotizacion` nunca se habilita para ese tenant; se trata como precondición de configuración, no como error en tiempo de conversación. |
| Cantidad solicitada igual a cero o negativa | Se rechaza la línea en la validación del paso 1, no se propaga a la generación del PDF. |

## Criterios de Aceptación

```
Given un tenant con catálogo activo
When el LLM invoca solicitar_cotizacion con SKUs válidos
Then el precio de cada línea proviene exclusivamente de catalog_items
  And el LLM nunca tuvo la posibilidad de especificar un precio en su respuesta

Given una solicitud de cotización que incluye un SKU inexistente
When se procesa
Then esa línea se excluye de la cotización
  And el usuario recibe información explícita de qué ítem no se pudo cotizar

Given una cotización ya generada
When el precio del catálogo cambia antes de que el PDF se despache
Then el PDF despachado refleja el precio congelado al momento de creación de la cotización

Given una tarea de generación de PDF que se reintenta por un fallo transitorio
When el reintento se ejecuta
Then no se genera un segundo PDF ni un segundo envío al usuario
```

## Requisitos de Cumplimiento Normativo Asociados

Este módulo no trata datos personales adicionales más allá de las referencias ya gobernadas por SPEC-000 (`data_subject_id`). Su relevancia normativa es comercial/de consumidor: sostiene que la única oferta válida enviada al cliente sea un documento cuyo contenido es enteramente verificable y trazable, mitigando el riesgo regulatorio (ej. SERNAC) señalado en la matriz de riesgos del informe.

## Fuera de Alcance

- Diseño visual de la plantilla del PDF (producto/UX).
- Lógica de descuentos dinámicos o negociación de precios — mencionada como extensión futura en el informe, no parte del alcance actual; de implementarse, debe modelarse como reglas explícitas en el motor de precios, nunca como libertad del LLM.
