# SPEC-001: Núcleo del Backend y Contrato de Colas

> Define la estructura de ejecución confiable (colas, idempotencia, rate limiting) que todos los módulos de dominio (SPEC-002 en adelante) reutilizan. No contiene lógica de negocio propia de ningún dominio.

## Contexto y objetivo

Al descartar n8n como motor de orquestación (informe §4.1), el equipo asume explícitamente la responsabilidad de construir la confiabilidad de procesamiento que antes era implícita en la herramienta no-code: reintentos, idempotencia y control de tasa. Esta spec define esa capa una sola vez, como cimiento compartido, para que ningún módulo de dominio tenga que resolverlo por su cuenta.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-001-01 | El sistema debe exponer un servicio HTTP (API) desplegado en Cloud Run que reciba webhooks y solicitudes internas. |
| RF-001-02 | El sistema debe exponer un servicio *worker* separado (otro despliegue de Cloud Run) que consuma tareas desde Cloud Tasks. |
| RF-001-03 | Toda tarea encolada debe portar una clave de idempotencia única por tenant. |
| RF-001-04 | El sistema debe registrar el resultado de cada tarea procesada (éxito/fallo/reintento) para trazabilidad operativa. |
| RF-001-05 | El sistema debe aplicar rate limiting por titular y por tenant antes de encolar cualquier tarea de procesamiento conversacional. |
| RF-001-06 | El sistema debe definir colas separadas por tipo de tarea, con política de reintentos y timeout configurados de forma independiente por cola. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-001-01 | Reintentos con backoff exponencial gestionados por Cloud Tasks, configurables por cola. |
| RNF-001-02 | Cualquier tarea debe poder reprocesarse sin producir efectos de negocio duplicados. |
| RNF-001-03 | El tiempo entre la recepción de un webhook y la respuesta de acuse (ack) debe ser inferior a 500 ms. |
| RNF-001-04 | Cada tarea tiene un timeout máximo configurado; al expirar se marca como fallida y queda disponible para inspección, nunca "colgada" indefinidamente. |

## Modelo de Datos

```sql
task_executions
  id                 uuid PK
  tenant_id          uuid FK
  idempotency_key    text NOT NULL
  queue_name         text
  status             enum('recibida','procesando','completada','fallida')
  attempt_count      integer DEFAULT 0
  last_error         text NULL
  created_at         timestamptz
  completed_at       timestamptz NULL
  UNIQUE (tenant_id, idempotency_key)

rate_limit_counters
  id                 uuid PK
  tenant_id          uuid FK
  subject_key        text          -- data_subject_id, o "tenant" para límites agregados
  window_start       timestamptz
  window_type        enum('hourly')
  count              integer DEFAULT 0
  UNIQUE (tenant_id, subject_key, window_start, window_type)
```

## Contratos de API / Colas

```
Cola: whatsapp-inbound-messages
  payload: { tenant_id, data_subject_id, message_id (idempotency_key), whatsapp_payload }
  consumidor: worker de pipeline conversacional (SPEC-002 / SPEC-004)
  reintentos: 5, backoff exponencial base 10s, máx 5min
  timeout por tarea: 30s

Cola: quote-generation
  payload: { tenant_id, data_subject_id, conversation_id, catalog_items[] }
  idempotency_key: conversation_id + hash(catalog_items)
  consumidor: worker de SPEC-005
  reintentos: 3, backoff exponencial base 30s
  timeout por tarea: 60s

Cola: retention-jobs
  payload: { tenant_id, data_subject_id, trigger }
  consumidor: worker de SPEC-000
  reintentos: 3
  timeout por tarea: 120s

POST /internal/v1/rate-limit/check
  body: { tenant_id, subject_key, window_type: "hourly", limit: 15 }
  → 200 { allowed: bool, remaining, reset_at }
  → 429 si excede; el llamador debe descartar o posponer el encolado
```

## Flujo

1. El servicio API recibe un evento, calcula su `idempotency_key` (ej. `message_id` entregado por el BSP).
2. Verifica en `task_executions` si esa clave ya existe para el tenant; si existe en estado `completada` o `procesando`, responde 200 sin reencolar.
3. Invoca `rate-limit/check`; si `allowed=false`, no se encola.
4. Crea el registro `task_executions` en estado `recibida`, encola en Cloud Tasks, y responde 200 de inmediato.
5. El worker toma la tarea, la marca `procesando`, ejecuta la lógica de negocio delegada al módulo de dominio correspondiente, y marca `completada` o `fallida` con el detalle del error.
6. Si falla y quedan reintentos, Cloud Tasks reintenta automáticamente respetando el backoff de la cola.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Mismo `idempotency_key` llega dos veces casi simultáneamente (condición de carrera) | La restricción UNIQUE (tenant_id, idempotency_key) rechaza el segundo insert; ese webhook responde 200 sin reencolar. |
| Una tarea completó su efecto de negocio pero falló al marcarse `completada` por un corte de red | Queda como "tarea huérfana" (`procesando` por más de N minutos); un job de salud la marca para inspección manual, no se reintenta automáticamente sin revisión. |
| Rate limit alcanzado justo en un mensaje que es una solicitud de derechos ARCOP+P | Las solicitudes de derechos (SPEC-000) se procesan en una cola separada sin cuota — excepción explícita, nunca se bloquean por rate limit. |
| Cola de cotizaciones degradada mientras la conversacional sigue activa | El usuario puede seguir conversando con normalidad; solo el tool-call de cotización queda pendiente con reintento, sin bloquear el resto del pipeline. |

## Criterios de Aceptación

```
Given un webhook con message_id ya procesado exitosamente
When se reenvía por el BSP (reintento de red)
Then el sistema responde 200 sin crear una nueva tarea
  And no se genera una segunda respuesta al usuario

Given un usuario que ya alcanzó su límite de 15 mensajes en la última hora
When envía un mensaje número 16
Then el mensaje no se encola en la cola conversacional
  And no se invoca RAG ni LLM para ese mensaje

Given una tarea de generación de cotización que falla por un error transitorio del LLM
When Cloud Tasks reintenta según la política de la cola
Then el reintento no genera un PDF duplicado si el primer intento ya se completó parcialmente
  And solo un PDF es efectivamente despachado al usuario

Given una tarea que excede su timeout configurado
When el worker no responde dentro del plazo
Then la tarea se marca fallida y queda disponible para reintento según la política de la cola
  And no permanece en estado "procesando" de forma indefinida sin ser detectada
```

## Requisitos de Cumplimiento Normativo Asociados

Este módulo no maneja datos personales más allá de referenciar `tenant_id`/`data_subject_id` como claves foráneas. Su relevancia de cumplimiento es indirecta: sostiene RNF-000-02 (SPEC-000) garantizando que los jobs de retención y derechos ARCOP+P se ejecuten de forma confiable, y aplica RNF-04 (rate limiting) antes de cualquier tratamiento conversacional.

## Fuera de Alcance

- Lógica de negocio específica de cada dominio (RAG, clasificación de intención, cotización) — vive en su propia spec; este módulo solo define el contrato de ejecución confiable que todas comparten.
- Colas de notificación de incidentes (SPEC-011).
- Autenticación de las APIs internas — se asume la red privada/IAM de GCP definida en SPEC-009.
