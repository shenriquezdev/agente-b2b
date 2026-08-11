# SPEC-002: Ingesta y Seguridad Conversacional

> Punto de entrada único de todo mensaje de WhatsApp al sistema. Aplica el gate de consentimiento (SPEC-000) y el contrato de colas/idempotencia (SPEC-001) antes de que cualquier mensaje llegue al pipeline conversacional (SPEC-004).

## Contexto y objetivo

Implementa RF-01 (Recepción Omnicanal) del informe de arquitectura, integrando dos garantías ya diseñadas en specs anteriores: ningún mensaje se procesa sin verificar la autenticidad del remitente (BSP), y ningún mensaje llega a RAG/LLM sin que exista consentimiento vigente para el titular. Esta spec es, en la práctica, donde el principio de licitud de la Ley N° 21.719 se hace cumplir en tiempo de ejecución, no solo se declara en un documento.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-002-01 | El sistema debe exponer un endpoint HTTP público que reciba webhooks del proveedor BSP de WhatsApp. |
| RF-002-02 | El sistema debe validar la firma criptográfica de cada webhook entrante antes de procesarlo. |
| RF-002-03 | El sistema debe rechazar y descartar sin procesar cualquier webhook cuya firma no sea válida. |
| RF-002-04 | El sistema debe resolver el tenant correspondiente a partir del número de WhatsApp de destino del mensaje. |
| RF-002-05 | El sistema debe consultar el estado de consentimiento del titular (SPEC-000) antes de encolar el mensaje para procesamiento conversacional. |
| RF-002-06 | Si no hay consentimiento vigente, el sistema debe despachar únicamente el aviso de consentimiento, sin encolar hacia RAG/LLM. |
| RF-002-07 | El sistema debe aplicar el gate de rate limiting (SPEC-001) antes de encolar cualquier mensaje. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-002-01 | El endpoint responde en menos de 500 ms desde la recepción hasta el acuse (hereda RNF-001-03). |
| RNF-002-02 | El endpoint no expone ningún detalle interno de error en la respuesta HTTP — mensajes genéricos hacia el exterior. |
| RNF-002-03 | El endpoint opera exclusivamente sobre TLS, sin puertos internos expuestos, conforme a la postura de red definida en el informe (§6.1). |

## Modelo de Datos

Reutiliza `tenants` y `data_subjects` (SPEC-000) y `task_executions` (SPEC-001). Se agrega:

```sql
whatsapp_numbers
  id                uuid PK
  tenant_id         uuid FK
  phone_number_id   text UNIQUE   -- identificador del número asignado por el BSP
  bsp_account_id    text
  active            boolean
```

Usada para resolver `tenant_id` a partir del número de destino del webhook entrante (RF-002-04).

## Contrato de API

```
POST /webhooks/whatsapp/{bsp_provider}
  headers: X-Signature
  body: payload crudo del BSP

  1. Verificar X-Signature contra el secreto configurado para ese bsp_provider.
     → inválida: 401, no se procesa ni se registra como tarea.
  2. Resolver tenant_id vía whatsapp_numbers.phone_number_id.
     → no existe o active = false: 404, se registra como evento anómalo para revisión.
  3. Extraer message_id, remitente, contenido.
  4. Invocar GET /internal/v1/consent/status (SPEC-000).
     → no vigente: encolar únicamente el aviso de consentimiento (cola de baja prioridad, no pasa por RAG/LLM).
  5. Invocar POST /internal/v1/rate-limit/check (SPEC-001).
     → excede cuota: no encolar; opcionalmente un único aviso de "límite alcanzado" por ventana.
  6. Encolar en whatsapp-inbound-messages (SPEC-001) con idempotency_key = message_id.
  7. Responder 200 de inmediato.
```

## Flujo

Ver contrato de API — la secuencia de validación (firma → tenant → consentimiento → rate limit → encolado) es el flujo completo de este módulo; no existe lógica adicional fuera de esos cinco pasos.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Webhook de un número de WhatsApp dado de baja (`active = false`) | Se rechaza con 404 y se registra como evento anómalo — podría indicar un número reciclado por el BSP. |
| Mensaje entrante no textual (imagen, audio, sticker) | Se encola igual; la decisión de cómo tratarlo corresponde al pipeline conversacional (SPEC-004), fuera del alcance de esta spec. |
| Firma válida pero payload malformado | 400; se registra el evento crudo para diagnóstico sin intentar procesarlo. |
| Ráfaga de webhooks duplicados del mismo mensaje en milisegundos | Protegida por la restricción de idempotencia de SPEC-001; solo el primero genera una tarea. |

## Criterios de Aceptación

```
Given un webhook con firma inválida
When llega al endpoint
Then se responde 401
  And no se crea ninguna tarea en la cola
  And no se consulta el estado de consentimiento del titular

Given un webhook válido de un titular sin consentimiento vigente
When se procesa
Then se despacha únicamente el aviso de consentimiento
  And no se invoca la cola conversacional (RAG/LLM)

Given un webhook válido de un titular con consentimiento vigente y dentro de su cuota
When se procesa
Then se encola en whatsapp-inbound-messages con idempotency_key = message_id
  And el endpoint responde en menos de 500 ms

Given un número de WhatsApp dado de baja que recibe un webhook
When se intenta resolver el tenant
Then se responde 404
  And se registra un evento anómalo para revisión manual
```

## Requisitos de Cumplimiento Normativo Asociados

- Aplica en tiempo de ejecución el gate de consentimiento de SPEC-000 (§7.2 del informe) — el principio de licitud se hace cumplir aquí, no solo se declara.
- La validación de firma y el rechazo de webhooks no autenticados forman parte de la postura de seguridad de red documentada en §6.1 del informe.

## Fuera de Alcance

- Lógica de qué responder o cómo clasificar el mensaje (SPEC-004).
- Gestión de credenciales/secretos del BSP (parte de la IaC/gestión de secretos de SPEC-009).
- Tratamiento de contenido multimedia más allá de su recepción y encolado.
