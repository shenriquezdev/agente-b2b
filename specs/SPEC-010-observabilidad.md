# SPEC-010: Observabilidad

> Implementa la estrategia del informe §8: stack nativo de GCP para telemetría de infraestructura + Sentry para errores de aplicación, con sanitización obligatoria de datos personales.

## Contexto y objetivo

Resuelve una debilidad explícita del documento original ("sin observabilidad/alerting"). Con la plataforma multi-tenant compartida ya decidida, un fallo no detectado afecta a todos los clientes a la vez — esta spec define cómo se detecta antes de que los clientes reclamen, y sostiene además la ventana de detección temprana que exige la notificación de brechas en 72 horas (SPEC-011).

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-010-01 | Todos los servicios (API, workers) emiten logs estructurados (JSON) hacia Cloud Logging, incluyendo `tenant_id` como campo indexado. |
| RF-010-02 | Se definen métricas de negocio en Cloud Monitoring: conversaciones/hora, tasa de error de cotización, latencia p50/p95/p99. |
| RF-010-03 | Se configuran *uptime checks* sobre los endpoints públicos (webhook, dashboard), con alerta ante caída. |
| RF-010-04 | Se configuran alertas ante incumplimiento de RNF-01 (latencia) y RNF-02 (disponibilidad). |
| RF-010-05 | Los errores de aplicación se capturan en Sentry, con el payload sanitizado para excluir contenido de conversaciones o datos personales identificables. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-010-01 | Ningún log ni evento de error puede contener el número de teléfono en texto plano ni el contenido textual de una conversación — regla verificada mediante un test automatizado en CI que inspecciona el formato de los logs de ejemplo. |
| RNF-010-02 | La retención de logs es coherente con las políticas de retención de datos ya definidas en SPEC-000 — la telemetría no se conserva indefinidamente si contiene identificadores, aunque sean opacos. |

## Contratos

```
Middleware de logging compartido (usado por todos los servicios):
  logger.info(event, { tenant_id, ...campos_no_sensibles })
  — la firma de la función no acepta un campo libre de "mensaje de usuario";
    quien quiera loguear contenido de conversación debe hacerlo explícito
    y pasa automáticamente por el filtro de sanitización.

Hook de Sentry (beforeSend):
  — elimina cualquier campo fuera de una lista blanca explícita
    (endpoint, tipo de error, tenant_id, código de status).
  — nunca reenvía el body de la request ni el contenido de mensajes.

Alertas configuradas (Cloud Monitoring):
  - latencia p95 > 4.0s sostenida por N minutos → RNF-01
  - uptime check fallido → RNF-02
  - tasa de error de la cola quote-generation > umbral → posible
    incidente de cotizaciones (deriva a SPEC-011 si se confirma)
```

## Flujo

1. Cada request/tarea genera logs estructurados, que Cloud Logging indexa por `tenant_id` y tipo de evento.
2. Cloud Monitoring agrega métricas de negocio e infraestructura sobre esos logs/eventos.
3. Las alertas configuradas notifican al canal definido (a especificar en implementación: email/Slack interno) ante umbral excedido.
4. Cualquier excepción no controlada se envía a Sentry pasando primero por el filtro de sanitización.
5. Una alerta clasificada como potencial incidente de seguridad dispara el flujo de SPEC-011.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Un desarrollador agrega por error un log que incluye contenido de un mensaje de usuario | El test de sanitización en CI (RNF-010-01) lo detecta y bloquea el merge — no depende únicamente de revisión manual. |
| Pico de errores que satura la cuota del plan de Sentry contratado | Se alerta sobre el propio volumen de errores como señal temprana de incidente, en vez de dejar que el exceso se trunque silenciosamente. |
| Una alerta de latencia se dispara por una causa externa (ej. degradación de la API de Anthropic, no de nuestro sistema) | Igual se registra y notifica; la clasificación de causa raíz es parte de la investigación, no una condición para no alertar. |

## Criterios de Aceptación

```
Given un log generado por cualquier servicio del backend
When se inspecciona su contenido
Then no contiene el número de teléfono en texto plano ni el contenido
  textual de una conversación

Given la latencia p95 supera 4.0 segundos durante el período configurado
When se evalúa la alerta
Then se notifica al canal definido
  And el evento queda disponible para vincularse a un registro de
  incidente en SPEC-011 si corresponde

Given una excepción no controlada en el backend
When se envía a Sentry
Then el payload recibido por Sentry no incluye datos personales
  identificables ni contenido de conversaciones

Given un pull request que introduce un log con contenido sensible
When se ejecuta el pipeline de CI
Then el test de sanitización falla y bloquea el merge
```

## Requisitos de Cumplimiento Normativo Asociados

La sanitización de datos personales en logs y errores (RNF-010-01) extiende el principio de minimización de datos (§7.10 del informe) hacia las herramientas de ingeniería. Esta capa es, además, la que sostiene la detección temprana necesaria para cumplir el plazo de notificación de brechas en 72 horas, desarrollado operativamente en SPEC-011.

## Fuera de Alcance

- El procedimiento de respuesta a incidentes en sí (SPEC-011).
- Dashboards de negocio para el equipo comercial — son contenido de producto sobre esta infraestructura, ya cubiertos como parte de SPEC-007.
