# SPEC-003: Pipeline RAG

> Implementa RF-02. Gestiona la ingesta de conocimiento corporativo por tenant y la recuperación semántica que alimenta al LLM en SPEC-004.

## Contexto y objetivo

Provee al agente conversacional el contexto corporativo específico de cada empresa cliente, usando `pgvector` dentro del PostgreSQL multi-tenant ya decidido (informe §4.3), sin motor vectorial separado. Define también la política de "no encontrado", que es la primera línea de defensa contra respuestas sin fundamento (complementaria a la garantía determinista de precios de SPEC-005).

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-003-01 | El sistema debe permitir ingerir documentos corporativos por tenant (carga y fragmentación en *chunks*). |
| RF-003-02 | El sistema debe generar embeddings para cada fragmento y almacenarlos en `pgvector`, asociados al `tenant_id`. |
| RF-003-03 | La búsqueda semántica debe estar estrictamente acotada al `tenant_id` de la conversación en curso — nunca debe poder recuperar contexto de otro tenant. |
| RF-003-04 | Una nueva versión de un documento debe reemplazar a la anterior de forma atómica, sin degradar la búsqueda mientras se procesa. |
| RF-003-05 | Si no se encuentra contexto relevante (score de similitud bajo un umbral definido), el sistema debe aplicar una respuesta de fallback explícita, nunca dejar que el LLM responda sin fundamento. |
| RF-003-06 | El sistema debe registrar qué documento/versión fundamentó la respuesta de cada turno de conversación, para trazabilidad. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-003-01 | El tiempo combinado de generación de embedding + búsqueda no debe superar ~500 ms, dado el presupuesto de latencia total de RNF-01. |
| RNF-003-02 | El aislamiento por tenant en la tabla de vectores se refuerza con Row-Level Security (RNF-05), no solo con un filtro aplicativo opcional. |
| RNF-003-03 | La re-ingesta de un documento no debe causar caída de disponibilidad de búsqueda para ese tenant durante el reprocesamiento. |

## Modelo de Datos

```sql
knowledge_documents
  id               uuid PK
  tenant_id        uuid FK
  source_filename  text
  version          integer
  status           enum('procesando','activo','reemplazado','fallido')
  uploaded_at      timestamptz
  uploaded_by      text

knowledge_chunks
  id            uuid PK
  tenant_id     uuid FK
  document_id   uuid FK
  chunk_index   integer
  content       text
  embedding     vector(1536)   -- columna pgvector
  created_at    timestamptz
```

## Contratos de API

```
POST /internal/v1/knowledge/documents
  body: { tenant_id, file, source_filename }
  → 202, encola job de fragmentación + embedding (cola dedicada, SPEC-001)
  → al completar: nuevos knowledge_chunks en status implícito "activo",
    versión anterior marcada "reemplazado" en la misma transacción

GET /internal/v1/knowledge/documents?tenant_id=
DELETE /internal/v1/knowledge/documents/{id}

Función interna: searchContext(tenant_id, query_embedding, top_k)
  → SIEMPRE filtra por tenant_id (no es un parámetro opcional en la firma)
  → retorna chunks con score de similitud
  → si el mejor score < umbral configurado: retorna "sin_contexto_suficiente"
```

## Flujo

1. Se carga un documento para un tenant → se encola su fragmentación y generación de embeddings.
2. Al completar, los nuevos `knowledge_chunks` quedan activos y la versión anterior se marca `reemplazado` en una única transacción (evita estado intermedio inconsistente).
3. En cada turno conversacional (SPEC-004), se genera el embedding del mensaje del usuario y se invoca `searchContext` acotado al tenant.
4. Si el score del mejor resultado supera el umbral, el contexto se incluye en la invocación al LLM y se registra qué `document_id`/versión lo originó.
5. Si no supera el umbral, se activa la política de "no encontrado" (respuesta de fallback definida por producto, ej. "no tengo información sobre eso, ¿quieres que te contacte un asesor?" — puede escalar a SPEC-008).

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Carga de documento falla a mitad del procesamiento (chunks parciales) | La versión nunca pasa a "activo" con chunks incompletos; queda en "fallido" y la versión anterior sigue sirviendo búsquedas. |
| Carga duplicada del mismo documento | Se trata como nueva versión (incremento de `version`), no como error — el reemplazo atómico ya cubre este caso. |
| Documento que excede el límite de procesamiento (tamaño/páginas) | Se rechaza en la carga con un error explícito, no se encola un job que fallaría de todas formas. |
| Tenant nuevo sin ningún documento cargado | `searchContext` retorna "sin_contexto_suficiente" de inmediato; el fallback debe distinguir este caso ("base de conocimiento no configurada") de un simple "no encontré la respuesta", para que el equipo del cliente sepa que falta configuración. |

## Criterios de Aceptación

```
Given un tenant con una base de conocimiento activa
When un usuario pregunta algo cubierto por esa base
Then la búsqueda retorna chunks con score sobre el umbral
  And la respuesta generada queda trazada al documento/versión que la fundamentó

Given un tenant sin documentos cargados
When un usuario hace cualquier pregunta
Then el sistema responde con el fallback de "base de conocimiento no configurada"
  And no se invoca al LLM sin contexto para generar una respuesta libre

Given un documento con una nueva versión en proceso de ingesta
When llega una consulta durante ese proceso
Then la búsqueda sigue usando la versión anterior activa
  And no hay ventana de tiempo sin resultados de búsqueda disponibles

Given una consulta cuyo mejor resultado de búsqueda está bajo el umbral de similitud
When se procesa
Then se activa la respuesta de fallback definida
  And no se envía ese contexto de baja relevancia al LLM como si fuera válido
```

## Requisitos de Cumplimiento Normativo Asociados

- La búsqueda estrictamente acotada por `tenant_id` (RF-003-03) es una aplicación directa del aislamiento multi-tenant exigido por RNF-05 y por el principio de confidencialidad de §6.1/§7.10 del informe.
- El registro de qué documento fundamentó cada respuesta (RF-003-06) sostiene la auditabilidad general del sistema, relevante ante una eventual fiscalización.

## Fuera de Alcance

- Interfaz de carga de documentos en el dashboard (SPEC-007).
- Elección específica del modelo de embeddings y el umbral numérico exacto de similitud (detalle de implementación/tuning, no bloqueante de esta spec).
