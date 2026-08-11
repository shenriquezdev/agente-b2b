# SPEC-008: Intervención Humana (Handoff)

> Implementa RF-09. Vista nativa dentro del mismo dashboard (SPEC-007) — se descartó deliberadamente una herramienta externa (ej. Chatwoot) para no fragmentar dónde viven los datos personales de los leads (informe §5.3).

## Contexto y objetivo

Resuelve un vacío identificado en el análisis crítico original: sin este módulo, cualquier conversación que el bot no pueda manejar queda sin salida. Se construye sobre la misma base de datos y el mismo sistema de identidad ya definidos, para que la gobernanza de datos de SPEC-000 (retención, ARCOP+P, auditoría) cubra también los mensajes enviados por agentes humanos, sin excepción ni sistema paralelo.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-008-01 | Un agente humano autenticado puede ver la lista de conversaciones activas de su tenant, con indicador de las marcadas para intervención. |
| RF-008-02 | Un agente puede abrir el hilo completo de una conversación y enviar un mensaje manual, despachado por el mismo canal de WhatsApp. |
| RF-008-03 | Al enviar el primer mensaje manual, la conversación pasa a `pausado_handoff` y el bot deja de responder automáticamente a ese titular. |
| RF-008-04 | Un agente puede devolver la conversación al bot ("reactivar automatización"). |
| RF-008-05 | El sistema marca automáticamente una conversación para intervención ante: solicitud explícita del usuario (SPEC-004), baja confianza sostenida del RAG (SPEC-003), o clasificación de Lead Caliente de alto valor (parámetro configurable por tenant). |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-008-01 | La transición a estado pausado es inmediata y consistente: ninguna respuesta automática del bot puede despacharse después de que un agente humano tomó la conversación. |

## Modelo de Datos

```sql
handoff_events
  id                uuid PK
  tenant_id         uuid FK
  conversation_id   uuid FK
  triggered_by      enum('usuario','sistema_baja_confianza','sistema_lead_caliente','agente_manual')
  agent_user_id     uuid NULL FK -> dashboard_users
  event_type        enum('marcada','tomada','devuelta_al_bot')
  occurred_at       timestamptz
```

Reutiliza `conversations.status` (SPEC-004) como fuente de verdad del estado bot/humano.

## Contratos

```
POST /internal/v1/conversations/{id}/handoff/flag
  body: { trigger }
  → registra handoff_events(event_type: 'marcada'), no cambia conversations.status

POST /internal/v1/conversations/{id}/handoff/take
  body: { agent_user_id }
  → conversations.status = 'pausado_handoff'
  → registra handoff_events(event_type: 'tomada')

POST /internal/v1/conversations/{id}/handoff/release
  → conversations.status = 'bot_activo'
  → registra handoff_events(event_type: 'devuelta_al_bot')

POST /internal/v1/conversations/{id}/messages/manual
  body: { agent_user_id, content }
  → si conversations.status != 'pausado_handoff': ejecuta handoff/take implícitamente
  → despacha el mensaje reutilizando el canal de WhatsApp (SPEC-002)
  → persiste en conversation_messages con sender = 'agente_humano'
```

## Flujo

1. Una conversación se marca para intervención (automática o manualmente) → `handoff/flag`, visible en el dashboard sin pausar aún al bot.
2. Un agente abre la conversación y escribe → `handoff/take` implícito, `conversations.status` pasa a `pausado_handoff`.
3. El worker del pipeline conversacional (SPEC-004) verifica `conversations.status` **inmediatamente antes de despachar** cualquier respuesta generada por el bot, no solo al inicio del procesamiento — si cambió a `pausado_handoff` mientras el bot procesaba, la respuesta se descarta.
4. El agente puede devolver la conversación al bot en cualquier momento (`handoff/release`).

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Dos agentes intentan tomar la misma conversación simultáneamente | El primero en escribir gana (`handoff/take` es la primera escritura efectiva); el segundo recibe aviso de que ya fue tomada, con el nombre del agente. |
| El worker del bot ya tenía una respuesta generada en cola cuando el agente toma la conversación (condición de carrera) | El worker revalida `conversations.status` justo antes de despachar (RNF-008-01) y descarta el envío si el estado cambió. |
| Un agente cierra sesión sin devolver la conversación al bot | Queda pausada indefinidamente hasta que alguien la retome — comportamiento esperado, evita que el bot reingrese a una conversación de forma inconsistente con lo ya hablado por el agente. |

## Criterios de Aceptación

```
Given una conversación en estado "bot_activo"
When un agente humano envía un mensaje manual
Then la conversación pasa a "pausado_handoff"
  And el bot no genera ni despacha ninguna respuesta posterior para ese titular

Given una conversación que el bot está procesando en el momento exacto
  en que un agente humano la toma
When el worker del bot intenta despachar su respuesta ya generada
Then el despacho se descarta porque el estado cambió a "pausado_handoff"

Given una conversación pausada por handoff
When el agente ejecuta "devolver al bot"
Then conversations.status vuelve a "bot_activo"
  And el bot puede volver a responder automáticamente a partir del siguiente mensaje

Given dos agentes que intentan tomar la misma conversación
When ambos actúan casi simultáneamente
Then solo uno queda como responsable
  And el segundo recibe una notificación explícita de que ya fue tomada
```

## Requisitos de Cumplimiento Normativo Asociados

Los mensajes enviados manualmente por agentes quedan en `conversation_messages`, la misma tabla gobernada por SPEC-000 (retención, derechos ARCOP+P, auditoría). Esta decisión de diseño —mantener todo en un solo sistema de datos personales— es la que evita el riesgo de fragmentación que se descartó explícitamente al no adoptar una herramienta externa de bandeja compartida (informe §5.3, §8).

## Fuera de Alcance

- Funciones avanzadas de helpdesk (respuestas predefinidas, colas de asignación automática entre agentes) — extensión futura, no parte de esta spec.
