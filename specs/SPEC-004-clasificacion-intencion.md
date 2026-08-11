# SPEC-004: Clasificación de Intención y Orquestación Conversacional

> Implementa RF-03. Es el orquestador central del turno conversacional: decide, para cada mensaje, si responde con RAG, activa una cotización (SPEC-005), deriva a un humano (SPEC-008) o enruta una solicitud de derechos (SPEC-000).

## Contexto y objetivo

Este módulo consume los mensajes ya validados y encolados por SPEC-002, aplica el contexto recuperado por SPEC-003, y decide el siguiente paso del sistema. Es el punto donde conviven, en una sola decisión, la lógica de negocio (calificar un lead) y las garantías de cumplimiento (detectar una solicitud de derechos antes de tratarla como una consulta cualquiera).

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-004-01 | El sistema debe clasificar cada mensaje entrante en una categoría: Curioso/Spam, Consulta Informativa, Lead Caliente, Solicita Humano, o Ejercicio de Derecho. |
| RF-004-02 | El sistema debe enrutar el procesamiento según la categoría: respuesta mínima (Curioso/Spam), pipeline RAG completo (Consulta Informativa/Lead Caliente), activación de handoff (Solicita Humano), o derivación a SPEC-000 (Ejercicio de Derecho). |
| RF-004-03 | La detección de una solicitud de ejercicio de derechos debe aplicarse mediante una regla determinista de palabras clave como red de seguridad, además de la clasificación del LLM — no debe depender exclusivamente del modelo. |
| RF-004-04 | El sistema debe mantener el estado y el historial acotado de cada conversación. |
| RF-004-05 | El sistema debe aplicar el system prompt específico configurado por cada tenant. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-004-01 | La clasificación de intención se resuelve en la misma invocación al LLM que genera la respuesta, salvo la red de seguridad determinista de RF-004-03, para no duplicar costo y latencia. |
| RNF-004-02 | El historial de conversación enviado al LLM está acotado a una ventana máxima de tokens configurada, para controlar costo y latencia. |

## Modelo de Datos

```sql
conversations
  id                uuid PK
  tenant_id         uuid FK
  data_subject_id   uuid FK
  status            enum('bot_activo','pausado_handoff','cerrado')
  last_message_at   timestamptz
  created_at        timestamptz

conversation_messages
  id                uuid PK
  conversation_id   uuid FK
  tenant_id         uuid FK
  sender            enum('usuario','bot','agente_humano')
  content           text
  intent_category   enum('curioso_spam','consulta_informativa','lead_caliente','solicita_humano','ejercicio_derecho') NULL
  created_at        timestamptz
```

## Contrato de Orquestación

```
Función interna: classifyAndRespond(tenant_id, conversation_id, message)

  1. Regla determinista: ¿el mensaje contiene patrones asociados a ejercicio de
     derechos (ej. "borrar mis datos", "quiero mis datos", "no quiero que me
     contacten")?
     → sí: enrutar directo a POST /internal/v1/rights-requests (SPEC-000),
       sin pasar por RAG/LLM para la clasificación de fondo.
  2. Si no aplica la regla determinista: cargar historial acotado + system
     prompt del tenant + contexto RAG (SPEC-003).
  3. Invocar al LLM con herramientas disponibles: clasificación de intención
     + (si el catálogo del tenant está configurado) la herramienta de
     cotización de SPEC-005.
  4. Según la categoría devuelta:
     - curioso_spam: respuesta breve predefinida, sin gasto adicional de RAG.
     - consulta_informativa: respuesta generada con el contexto RAG.
     - lead_caliente: respuesta generada + evaluación de disparo de handoff
       proactivo según configuración del tenant (SPEC-008).
     - solicita_humano: se invoca el flag de handoff (SPEC-008) y se informa
       al usuario que un agente continuará la conversación.
  5. Persistir el mensaje y su intent_category en conversation_messages.
```

## Flujo

Ver contrato de orquestación — es la secuencia completa del módulo. Cada paso delega en la spec correspondiente (SPEC-000, SPEC-003, SPEC-005, SPEC-008); esta spec no duplica su lógica, solo la invoca en el orden correcto.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Mensaje ambiguo que podría ser "Lead Caliente" y "Solicita Humano" a la vez | La solicitud explícita de hablar con una persona tiene prioridad sobre continuar con el bot, sin excepción. |
| Conversación inactiva que retoma contacto tiempo después | Se continúa la misma `conversation` si está dentro de la ventana de retención vigente (SPEC-000); si expiró, se trata como conversación nueva. |
| Usuario cambia de tema abruptamente dentro de la misma conversación | No requiere lógica especial: el historial acotado (RNF-004-02) y el contexto RAG recuperado en cada turno son suficientes para que el LLM maneje el cambio de tema. |
| La regla determinista de derechos tiene un falso positivo (ej. el usuario pregunta por la "política de eliminación de pedidos", no por sus datos) | Se enruta igual a SPEC-000 como solicitud de derecho; es preferible un falso positivo ocasional (revisable) a un falso negativo que deje una solicitud real sin atender. |

## Criterios de Aceptación

```
Given un mensaje que contiene un patrón asociado a ejercicio de derechos
When se procesa
Then se enruta directamente a SPEC-000 sin depender de la clasificación del LLM
  And queda registrado con intent_category = "ejercicio_derecho"

Given un mensaje clasificado como "Curioso/Spam"
When se procesa
Then se responde con el mensaje breve predefinido
  And no se invoca la búsqueda RAG completa

Given un mensaje clasificado como "Solicita Humano"
When se procesa
Then se activa el flag de handoff de SPEC-008
  And el usuario recibe confirmación de que un agente continuará la conversación

Given una conversación cuyo historial excede la ventana máxima configurada
When se invoca al LLM
Then el historial enviado se trunca según la política definida
  And la conversación sigue siendo coherente para el usuario
```

## Requisitos de Cumplimiento Normativo Asociados

- RF-004-03 (regla determinista de derechos como red de seguridad, no dependencia exclusiva del LLM) refuerza directamente RF-000-06 de SPEC-000: el ejercicio de derechos ARCOP+P no puede depender únicamente del comportamiento probabilístico de un modelo de lenguaje.

## Fuera de Alcance

- Contenido específico de los system prompts por tenant (gestión de producto, no de esta spec).
- Lógica de generación de PDF y resolución de precios (SPEC-005).
- Interfaz de usuario de la intervención humana (SPEC-008).
