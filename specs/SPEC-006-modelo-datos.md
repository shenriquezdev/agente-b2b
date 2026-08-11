# SPEC-006: Modelo de Datos y Esquema Multi-Tenant

> Spec transversal: define el patrón canónico de aislamiento (Row-Level Security) y cifrado por columna que todas las specs anteriores (SPEC-000 a SPEC-005) asumen ya disponible. Ninguna tabla de tenant se crea sin aplicar este patrón.

## Contexto y objetivo

Las specs de dominio ya definieron sus propias tablas asumiendo aislamiento por `tenant_id` y cifrado de campos sensibles. Esta spec formaliza **cómo** se garantiza eso de forma consistente en todo el sistema, para que el aislamiento no dependa de que cada desarrollador lo recuerde en cada query, sino de un mecanismo que falla de forma segura por defecto.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-006-01 | Toda tabla que contenga datos de un tenant debe tener una columna `tenant_id` no nula. |
| RF-006-02 | Toda tabla con `tenant_id` debe tener Row-Level Security habilitada y forzada (`FORCE ROW LEVEL SECURITY`), con una política que restrinja el acceso al tenant activo en el contexto de la sesión. |
| RF-006-03 | Los campos de identificación directa de personas (ej. número de teléfono) deben cifrarse a nivel de columna con una llave de Cloud KMS específica por tenant. |
| RF-006-04 | Debe existir un mecanismo de rotación de llaves de cifrado sin tiempo de inactividad. |
| RF-006-05 | Las migraciones de esquema se aplican de forma versionada y reproducible mediante una herramienta de migraciones (a definir en diseño técnico de implementación). |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-006-01 | Ninguna consulta a una tabla con `tenant_id` puede ejecutarse sin que el contexto de sesión de base de datos establezca explícitamente el tenant activo antes de la transacción. |
| RNF-006-02 | Las pruebas automatizadas de aislamiento entre tenants cubren el 100% de las tablas con `tenant_id`, como *gate* obligatorio del pipeline de CI (hereda la estrategia de SPEC-009). |

## Patrón Canónico de Aislamiento

```sql
-- Aplicado a TODA tabla que contenga tenant_id, sin excepción
ALTER TABLE <tabla> ENABLE ROW LEVEL SECURITY;
ALTER TABLE <tabla> FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON <tabla>
  USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Sin contexto de sesión establecido, current_setting() falla o retorna
-- un valor inválido → la policy deniega todo por defecto. No existe una
-- policy de "allow all" salvo un rol de administración interno, auditado
-- por separado y nunca usado por el flujo de aplicación estándar.
```

```
Wrapper obligatorio en el backend: withTenantContext(tenant_id, fn)
  - Abre una transacción.
  - Ejecuta: SET LOCAL app.current_tenant = '<tenant_id>';
  - Ejecuta fn(transaction) — toda la lógica de dominio corre aquí.
  - Cierra la transacción (commit/rollback).

  Ninguna consulta del backend a una tabla con tenant_id se ejecuta fuera
  de este wrapper. Es el único punto de entrada permitido a datos de tenant.
```

## Catálogo de Tablas por Spec (referencia consolidada)

| Tabla | Definida en | Contiene campos cifrados |
|---|---|---|
| `tenants` | SPEC-000 | No |
| `data_subjects` | SPEC-000 | Sí (`phone_encrypted`) |
| `consent_records` | SPEC-000 | No |
| `processing_activities` | SPEC-000 | No |
| `subprocessors` | SPEC-000 | No |
| `rights_requests` | SPEC-000 | No |
| `retention_jobs` | SPEC-000 | No |
| `audit_log` | SPEC-000 | No |
| `task_executions` | SPEC-001 | No |
| `rate_limit_counters` | SPEC-001 | No |
| `whatsapp_numbers` | SPEC-002 | No |
| `knowledge_documents` / `knowledge_chunks` | SPEC-003 | No |
| `conversations` / `conversation_messages` | SPEC-004 | Revisar: `content` puede incluir datos personales mencionados por el usuario; se trata como sensible a nivel de acceso aunque no se cifre campo a campo |
| `catalog_items` / `quotes` / `quote_line_items` | SPEC-005 | No |
| `dashboard_users` | SPEC-007 | No |
| `handoff_events` | SPEC-008 | No |
| `security_incidents` | SPEC-011 | No |

## Flujo

1. Cada request/tarea del backend resuelve su `tenant_id` (ya presente en el payload de la tarea, según el contrato de SPEC-001).
2. Se invoca `withTenantContext(tenant_id, fn)`, que establece el contexto RLS antes de ejecutar cualquier lógica de dominio.
3. Toda lectura/escritura dentro de `fn` queda automáticamente acotada al tenant activo por la política RLS, sin que el código de dominio tenga que repetir el filtro `WHERE tenant_id = ...` como única defensa.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Una query se ejecuta por error fuera del wrapper `withTenantContext` | Sin contexto de sesión establecido, la política RLS deniega el acceso por defecto (falla segura), en vez de exponer datos de todos los tenants. |
| Rotación de llave KMS a mitad de operación | Debe soportarse descifrado con la llave anterior durante una ventana de transición mientras se re-cifra en segundo plano, sin downtime. |
| Un job administrativo interno necesita operar sobre múltiples tenants (ej. reporte agregado) | Usa un rol explícitamente auditado y separado del flujo de aplicación estándar, nunca el mismo camino que usan las requests de usuario. |

## Criterios de Aceptación

```
Given dos tenants A y B con datos en la misma tabla
When se ejecuta una consulta con el contexto de sesión establecido en tenant A
Then la consulta no puede retornar ni afectar ninguna fila de tenant B,
  incluso si la condición WHERE de la query aplicativa fuera incorrecta

Given una consulta ejecutada sin pasar por withTenantContext
When se intenta leer una tabla con tenant_id
Then la política RLS deniega el acceso por defecto

Given una rotación de llave KMS en curso
When se solicita descifrar un campo cifrado con la llave anterior
Then el sistema puede descifrarlo correctamente durante la ventana de transición
  And no hay interrupción de servicio durante la rotación

Given el pipeline de CI ejecutando la suite de aislamiento
When se agrega una tabla nueva con tenant_id sin política RLS
Then el pipeline falla explícitamente, bloqueando el despliegue
```

## Requisitos de Cumplimiento Normativo Asociados

Este módulo es la base técnica directa de RNF-05 (Aislamiento Multi-Tenant) y RNF-07 (Cifrado) del informe de arquitectura, y sustenta el principio de privacidad desde el diseño y por defecto descrito en §7.10.

## Fuera de Alcance

- Elección específica de la herramienta de migraciones de esquema (detalle de implementación).
- Diseño de negocio del catálogo de productos más allá de lo ya definido en SPEC-005.
