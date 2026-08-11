# SPEC-000: Gobernanza de Datos y Consentimiento

> Módulo bloqueante — ningún otro módulo del sistema puede procesar datos personales de un titular antes de que este exista y cumpla las condiciones que define esta spec.

## Contexto y objetivo

Este módulo es la base de la que dependen RF-06 (Registro Estructurado) y RF-10 (Derechos ARCOP+P), y sustenta directamente el diseño de cumplimiento de la Ley N° 21.719 documentado en §7 del informe de arquitectura. Su objetivo es garantizar que **ningún dato personal de un lead se procese sin una base de licitud registrada, versionada y auditable**, y que los derechos ARCOP+P puedan ejercerse de forma verificable, no solo declarada.

Este módulo se construye antes que el pipeline conversacional (SPEC-002), porque este último depende de él para decidir si puede o no invocar RAG/LLM sobre un mensaje entrante.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-000-01 | El sistema debe crear un registro de titular (`data_subject`) la primera vez que un número de WhatsApp contacta a un tenant, antes de procesar cualquier consulta sustantiva. |
| RF-000-02 | El sistema debe registrar el consentimiento como evento versionado: texto exacto aceptado, finalidad específica, canal, marca de tiempo. |
| RF-000-03 | El sistema debe permitir revocar el consentimiento con el mismo nivel de fricción con que se otorgó (ej. una palabra clave por WhatsApp). |
| RF-000-04 | El sistema debe bloquear cualquier tratamiento no esencial (RAG, LLM, scoring) sobre un titular sin consentimiento vigente para la finalidad correspondiente. |
| RF-000-05 | El sistema debe registrar y exponer, por tenant, un extracto técnico de Actividades de Tratamiento (insumo para el RAT legal del cliente). |
| RF-000-06 | El sistema debe permitir crear, ejecutar y trazar solicitudes de derechos ARCOP+P (Acceso, Rectificación, Cancelación, Oposición, Portabilidad) por titular. |
| RF-000-07 | El sistema debe ejecutar de forma automatizada y programada la purga/anonimización de titulares cuyo período de retención haya vencido. |
| RF-000-08 | El sistema debe registrar, por tenant, la lista de subencargados de tratamiento y si involucran transferencia internacional de datos. |
| RF-000-09 | El sistema debe bloquear el alta operativa de un tenant nuevo si no existe un DPA firmado registrado. |
| RF-000-10 | Toda operación sobre datos personales (creación, lectura administrativa, modificación, eliminación) debe quedar en un registro de auditoría inmutable. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-000-01 | Aislamiento por tenant mediante Row-Level Security en todas las tablas de este módulo (hereda RNF-05). |
| RNF-000-02 | Una solicitud de cancelación (derecho de supresión) debe completarse en menos de 24 horas desde su creación, salvo excepción documentada (ej. obligación legal de conservación). |
| RNF-000-03 | El registro de auditoría (`audit_log`) debe ser de solo-append: ninguna operación del sistema puede modificar o eliminar un registro existente. |
| RNF-000-04 | Los campos identificadores directos (número de teléfono) se almacenan cifrados/hasheados a nivel de columna (hereda RNF-07), nunca en texto plano en tablas de consulta general. |
| RNF-000-05 | Toda operación de este módulo debe ser idempotente: reintentar la misma solicitud de derecho o el mismo evento de consentimiento no debe duplicar registros ni efectos. |

## Modelo de Datos

```sql
tenants
  id                    uuid PK
  legal_name            text
  dpa_signed_at         timestamptz          -- NULL bloquea operación (RF-000-09)
  retention_period_days integer NOT NULL DEFAULT 730
  timezone              text

data_subjects
  id                    uuid PK
  tenant_id             uuid FK -> tenants
  phone_hash            text NOT NULL        -- hash del número, no texto plano
  phone_encrypted       bytea                -- cifrado por tenant (Cloud KMS), solo desencriptable para despacho real
  display_name          text
  status                enum('active','opposed','deleted')
  created_at            timestamptz
  UNIQUE (tenant_id, phone_hash)

consent_records
  id                    uuid PK
  tenant_id             uuid FK
  data_subject_id       uuid FK
  purpose               enum('respuesta_consultas','perfilamiento_comercial','comunicaciones_futuras')
  consent_text_version  text NOT NULL
  consent_text_snapshot text NOT NULL        -- copia literal del texto aceptado
  channel               text
  granted_at            timestamptz
  revoked_at            timestamptz NULL
  revocation_method     text NULL

processing_activities   -- insumo técnico del RAT
  id                    uuid PK
  tenant_id             uuid FK
  activity_name         text
  purpose               text
  legal_basis           text
  data_categories        text[]
  retention_period_days integer
  recipients            text[]               -- referencia a subprocessors.id
  international_transfer boolean
  updated_at             timestamptz

subprocessors
  id                    uuid PK
  name                  text                 -- ej. 'Anthropic', 'BSP WhatsApp', 'GCP'
  role                  text
  dpa_signed_at         timestamptz
  data_categories       text[]
  country               text

rights_requests
  id                    uuid PK
  tenant_id             uuid FK
  data_subject_id       uuid FK
  right_type            enum('acceso','rectificacion','cancelacion','oposicion','portabilidad')
  status                enum('recibida','en_proceso','completada','rechazada')
  requested_at          timestamptz
  requested_via         text                 -- 'whatsapp_intent' | 'dashboard' | 'email_soporte'
  completed_at          timestamptz NULL
  resolution_notes      text NULL

retention_jobs
  id                    uuid PK
  tenant_id             uuid FK
  data_subject_id       uuid FK
  executed_at            timestamptz
  trigger                enum('retention_expired','rights_request','opposition')
  scope                  enum('anonymized','deleted')
  affected_tables        text[]

audit_log
  id                    uuid PK
  tenant_id             uuid FK
  actor_type            enum('system','human_agent','data_subject')
  actor_id              text NULL
  action                 text
  target_table            text
  target_id               uuid
  occurred_at             timestamptz
```

## Contratos de API (internos)

```
POST /internal/v1/consent
  body: { tenant_id, phone, purpose, consent_text_version, channel }
  → crea data_subject si no existe (idempotente por phone_hash) + consent_record
  → 201 { data_subject_id, consent_record_id }

POST /internal/v1/consent/revoke
  body: { tenant_id, data_subject_id, purpose | "all" }
  → 200, dispara evento data_subject.consent_revoked

POST /internal/v1/rights-requests
  body: { tenant_id, data_subject_id, right_type, requested_via }
  → 201 { request_id, status: "recibida" }
  → si right_type == "cancelacion" | "oposicion": encola ejecución automática inmediata

GET /internal/v1/rights-requests/{id}
  → 200 { status, completed_at, resolution_notes }

GET /internal/v1/rat/export?tenant_id={id}
  → 200 (JSON/CSV) extracto de processing_activities + subprocessors para ese tenant

GET /internal/v1/consent/status?tenant_id={id}&data_subject_id={id}&purpose={p}
  → 200 { granted: bool, granted_at, revoked_at }
  → Consultado obligatoriamente por SPEC-002 antes de invocar RAG/LLM
```

## Flujo

1. Llega el primer mensaje de un número no registrado para ese tenant (verificado por `phone_hash`).
2. El pipeline conversacional (SPEC-002) invoca `GET /consent/status`; si no existe, despacha el aviso de consentimiento vigente y **no** invoca RAG/LLM.
3. El usuario responde afirmativamente → `POST /consent` crea `data_subject` + `consent_record` (finalidad `respuesta_consultas`) → el pipeline puede continuar desde el siguiente mensaje.
4. Si en cualquier momento el clasificador de intención (RF-03, SPEC-004) detecta una solicitud de derecho, invoca `POST /rights-requests`.
   - `cancelacion` u `oposicion` → ejecución automática inmediata (job idempotente que anonimiza/purga y registra en `retention_jobs`).
   - `acceso` o `portabilidad` → se compila el export y se marca `completada` al entregarse.
   - `rectificacion` → se enruta a intervención humana (SPEC-008), dado que requiere validar el dato correcto con el titular.
5. Un job programado diario recorre `data_subjects` por tenant y aplica `retention_period_days`; los vencidos e inactivos se anonimizan automáticamente.
6. Cada paso anterior escribe en `audit_log` sin excepción.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Revocación de consentimiento a mitad de un proceso de cotización ya iniciado | El proceso en curso puede completarse (base de licitud distinta: ejecución de una gestión ya solicitada por el propio titular); no se inicia ningún tratamiento nuevo tras la revocación. |
| Mismo número de WhatsApp reutilizado por una persona distinta (teléfono reciclado) | Limitación conocida y documentada: el sistema no puede detectar el cambio de titular; no se trata como error del sistema. |
| Solicitud de portabilidad sobre un titular con datos parcialmente ya purgados por retención | Se entrega lo disponible junto con una nota explícita de qué fue purgado y cuándo — nunca se falla en silencio. |
| Doble solicitud de cancelación (usuario escribe "BORRAR" dos veces) | Idempotente: la segunda solicitud referencia el mismo `retention_job` ya ejecutado o en curso, sin generar un segundo job. |
| Alta de un tenant nuevo sin `dpa_signed_at` | Bloqueada a nivel de base de datos y de API — no es un chequeo opcional de UI. |

## Criterios de Aceptación

```
Given un número de WhatsApp que nunca contactó a un tenant
When envía su primer mensaje
Then el sistema responde únicamente con el aviso de consentimiento
  And no se invoca RAG ni LLM
  And no se crea consent_record hasta la respuesta afirmativa del usuario

Given un titular con consent_record vigente para "respuesta_consultas"
When revoca su consentimiento
Then consent_records.revoked_at queda registrado con el método de revocación
  And una nueva consulta del mismo titular vuelve a mostrar el aviso de consentimiento antes de continuar

Given un titular solicita "quiero que borren mis datos"
When el clasificador de intención lo detecta
Then se crea un rights_request tipo "cancelacion"
  And se ejecuta el job de purga en menos de 24 horas
  And queda un registro en retention_jobs y en audit_log

Given un tenant sin dpa_signed_at
When se intenta procesar cualquier mensaje entrante para ese tenant
Then la operación es rechazada antes de tocar cualquier tabla de datos personales
```

## Requisitos de Cumplimiento Normativo Asociados (Ley N° 21.719)

- Consentimiento específico, informado y revocable con la misma facilidad (§7.2 del informe) → RF-000-02, RF-000-03.
- Derechos ARCOP+P ejecutables, no solo declarados (§7.3) → RF-000-06.
- Registro de Actividades de Tratamiento (§7.5) → RF-000-05, tabla `processing_activities`.
- Retención definida y automatizada (§7.4) → RF-000-07.
- Trazabilidad de subencargados y transferencias internacionales (§7.1, §7.8) → RF-000-08, tabla `subprocessors`.
- Auditoría como soporte de notificación de brechas en 72h (§7.7) → RF-000-10, tabla `audit_log`.

## Fuera de Alcance

- Interfaz del dashboard para que un agente humano gestione manualmente una solicitud (SPEC-007/SPEC-008).
- La lógica de clasificación de intención que detecta que un mensaje es una solicitud de derecho (SPEC-004).
- El texto legal específico del aviso de consentimiento (responsabilidad legal/producto; esta spec solo garantiza que cualquier texto vigente quede versionado y trazado).
- Mecanismo de cifrado por columna vía Cloud KMS (definido en SPEC-006; esta spec asume esa capa disponible y la consume).
