# SPEC-011: Respuesta a Incidentes y Brechas

> Implementación operativa directa de §7.7 del informe (notificación de brechas en 72 horas) y de RNF-08. Es, en gran parte, un runbook — no solo una funcionalidad de software.

## Contexto y objetivo

Garantiza que, ante un incidente de seguridad que involucre datos personales, la organización pueda cumplir el plazo legal de notificación de 72 horas a la Agencia de Protección de Datos Personales y a los titulares afectados. Se apoya directamente en la capa de observabilidad de SPEC-010 para la detección temprana.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-011-01 | Debe existir un procedimiento documentado y accionable para clasificar un evento detectado como incidente de seguridad o brecha de datos personales. |
| RF-011-02 | El sistema debe registrar, para cada incidente confirmado, el momento de detección, el alcance (tenants/titulares afectados) y las acciones tomadas. |
| RF-011-03 | El procedimiento debe garantizar que la notificación formal a la Agencia y a los titulares afectados sea posible dentro de las 72 horas desde la detección. |
| RF-011-04 | El Delegado de Protección de Datos (DPO, §7.6 del informe) debe ser notificado automáticamente ante cualquier alerta de observabilidad (SPEC-010) clasificada como potencial incidente de seguridad. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-011-01 | El plazo interno de escalamiento —desde la alerta técnica hasta la clasificación humana del incidente— no supera unas pocas horas, dejando margen suficiente dentro de la ventana de 72 horas para investigar y notificar formalmente. |

## Modelo de Datos

```sql
security_incidents
  id                              uuid PK
  detected_at                     timestamptz     -- momento que cuenta para el plazo legal
  classified_at                   timestamptz NULL
  severity                        enum('baja','media','alta','critica')
  affected_tenants                uuid[]
  affected_data_subjects_count    integer NULL
  status                          enum('detectado','en_investigacion','contenido','notificado','cerrado')
  notified_agency_at              timestamptz NULL
  notified_subjects_at            timestamptz NULL
  summary                         text
  dpo_notified_at                 timestamptz NULL
```

## Contrato

```
POST /internal/v1/incidents
  body: { source: 'alerta_automatica' | 'reporte_manual', detected_at, summary, severity_inicial }
  → crea security_incidents(status: 'detectado', detected_at)
  → notifica automáticamente al DPO (RF-011-04)
  → invocado por: alertas de SPEC-010 clasificadas como potencial incidente
    de seguridad, o reporte manual desde el dashboard interno del equipo
```

## Flujo (Runbook)

```
1. Detección
   — Alerta automatizada (SPEC-010) o reporte manual.
   — Se crea el registro en security_incidents con detected_at = ahora.
   — Notificación automática e inmediata al DPO.

2. Clasificación
   — El DPO/equipo técnico determina severity y alcance preliminar
     (affected_tenants, affected_data_subjects_count si es posible estimarlo).
   — classified_at se registra; el reloj de 72 horas sigue corriendo desde
     detected_at, no desde este paso.

3. Contención técnica
   — Acciones inmediatas: revocar credenciales comprometidas, aislar el
     componente afectado, bloquear el vector de ataque identificado.
   — status → 'contenido' cuando la causa activa deja de operar.

4. Evaluación de notificación
   — Si se confirma que involucra datos personales: se prepara la
     notificación formal a la Agencia y a los titulares afectados.
   — Debe completarse dentro de 72 horas desde detected_at.

5. Cierre
   — status → 'cerrado', con documentación de causa raíz y medidas
     correctivas aplicadas (incluye, si corresponde, ajustes a las
     alertas de SPEC-010 para detectar antes un caso similar).
```

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Incidente detectado fuera de horario hábil (noche, fin de semana) | El plazo de 72 horas corre igual; el procedimiento de escalamiento (RNF-011-01) incluye contacto fuera de horario hábil para el DPO/on-call — no puede depender de que alguien lo note al llegar el lunes. |
| Un incidente que en primera instancia parece afectar a un solo tenant, pero la investigación revela que afectó a varios (ej. por un bug de aislamiento) | `affected_tenants` se actualiza durante la investigación sin perder `detected_at`, que es el que cuenta para el plazo legal. |
| Falso positivo: una alerta automática que tras revisión no constituye un incidente real | Se cierra igual con `status = cerrado` y `severity = baja`, manteniendo el registro para trazabilidad y para calibrar las alertas de SPEC-010 a futuro. |
| Incidente que involucra a un subencargado (ej. brecha reportada por el propio BSP o por Anthropic) | Se registra igual en `security_incidents`; la coordinación de la notificación se hace en conjunto con el subencargado, pero el plazo de 72 horas hacia la Agencia sigue siendo responsabilidad de la plataforma como Encargado del Tratamiento. |

## Criterios de Aceptación

```
Given una alerta de observabilidad clasificada como potencial incidente de seguridad
When se dispara
Then se crea un registro en security_incidents con detected_at correcto
  And el DPO recibe notificación automática

Given un incidente confirmado que involucra datos personales
When se completa la investigación
Then la notificación a la Agencia y a los titulares afectados se realiza
  dentro de las 72 horas desde detected_at

Given un incidente cuyo alcance se amplía durante la investigación
When se actualiza affected_tenants
Then detected_at no se modifica
  And el plazo de 72 horas sigue calculándose desde la detección original

Given una alerta que resulta ser un falso positivo
When se revisa
Then el incidente se cierra con severity "baja"
  And queda disponible para análisis histórico de calibración de alertas
```

## Requisitos de Cumplimiento Normativo Asociados

Esta spec es la implementación operativa directa de §7.7 del informe (notificación de brechas de seguridad en 72 horas) y de RNF-08 del documento de requisitos.

## Fuera de Alcance

- La investigación forense técnica detallada de un incidente específico — es un proceso operativo ad-hoc, no especificable de antemano.
- El contenido legal exacto de la notificación a la Agencia — responsabilidad del DPO/asesoría legal, no de esta spec técnica.
