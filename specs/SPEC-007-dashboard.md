# SPEC-007: Dashboard B2B

> Implementa RF-07 y RF-08. Interfaz de marca blanca para el personal de cada empresa cliente, construida en Next.js con autenticación GCP Identity Platform.

## Contexto y objetivo

Expone al equipo de ventas de cada tenant sus métricas, leads, cotizaciones, catálogo y base de conocimiento, resolviendo el tema visual en el servidor antes del renderizado (informe §4.7) y aplicando control de acceso basado en roles sobre datos explícitamente sensibles ("métricas financieras y contactos").

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-007-01 | Autenticación de usuarios del dashboard vía GCP Identity Platform, con aislamiento nativo por tenant. |
| RF-007-02 | Control de acceso basado en roles (`administrador`, `ventas`, `solo_lectura`) mediante *custom claims* del token. |
| RF-007-03 | Resolución en el servidor del tema/marca (logo, colores) del tenant, antes del renderizado de cualquier página. |
| RF-007-04 | Panel de métricas: volumen de conversaciones, leads por categoría de intención, cotizaciones generadas/despachadas, tasa de conversión. |
| RF-007-05 | Listado de leads con filtro por categoría de intención y estado. |
| RF-007-06 | Exportación de reportes en PDF/CSV. |
| RF-007-07 | Gestión del catálogo de productos/precios que alimenta SPEC-005. |
| RF-007-08 | Carga y administración de documentos de la base de conocimiento que alimenta SPEC-003. |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-007-01 | Cada vista consulta exclusivamente datos del tenant del usuario autenticado, reforzado por el mismo mecanismo de RLS de SPEC-006 — nunca solo por lógica de UI. |
| RNF-007-02 | Toda ruta de API que modifica datos (catálogo, base de conocimiento) valida el rol del usuario en el servidor, no solo oculta la opción en la interfaz. |

## Modelo de Datos

```sql
dashboard_users
  id                       uuid PK
  tenant_id                uuid FK
  identity_platform_uid    text UNIQUE
  role                     enum('administrador','ventas','solo_lectura')
  created_at               timestamptz
```

Las vistas de este módulo consultan, sin duplicarlas, las tablas ya definidas en specs anteriores: `conversations`, `conversation_messages`, `quotes`, `catalog_items`, `knowledge_documents`.

## Contratos

```
Autenticación: cada request al backend del dashboard porta un JWT de
Identity Platform con custom claims { tenant_id, role }.

Middleware de autorización (aplicado a toda ruta):
  1. Verifica el JWT.
  2. Extrae tenant_id y role de los custom claims.
  3. Invoca withTenantContext(tenant_id, ...) (SPEC-006) antes de cualquier
     consulta — el tenant del dashboard nunca se toma de un parámetro de
     la URL o del body, siempre del token verificado.
  4. Para rutas de escritura (catálogo, base de conocimiento), valida que
     role esté en el conjunto permitido para esa acción; "solo_lectura"
     nunca pasa esta validación en rutas de escritura.

GET  /api/dashboard/metrics?range=
GET  /api/dashboard/leads?category=&status=
GET  /api/dashboard/reports/export?format=pdf|csv
POST /api/dashboard/catalog-items          (rol: administrador, ventas)
POST /api/dashboard/knowledge-documents    (rol: administrador)
```

## Flujo

1. El usuario inicia sesión → Identity Platform valida credenciales y emite un JWT con `tenant_id`/`role`.
2. Cada página del dashboard (Next.js, renderizado en servidor) valida el token, resuelve el tema visual del tenant, y solicita datos exclusivamente a través del middleware de autorización.
3. Las rutas de escritura validan el rol antes de tocar cualquier tabla.

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Usuario con rol `solo_lectura` intenta modificar el catálogo mediante una llamada directa a la API (sin pasar por la UI) | Rechazado en el servidor (RNF-007-02) — la restricción de UI no es la única defensa. |
| Usuario de un tenant intenta acceder, por URL directa, a un recurso identificado con el ID de otro tenant | Bloqueado por Row-Level Security (SPEC-006), incluso si existiera un bug de UI que expusiera el enlace. |
| Token expirado a mitad de sesión | La sesión se invalida y se solicita reautenticación; ninguna request con token vencido llega a ejecutar una consulta. |

## Criterios de Aceptación

```
Given un usuario autenticado con rol "solo_lectura"
When intenta crear un catalog_item mediante la API
Then la operación es rechazada en el servidor con un error de autorización

Given un usuario autenticado del tenant A
When solicita, por URL directa, un recurso perteneciente al tenant B
Then la respuesta no expone ningún dato de tenant B

Given un usuario autenticado del tenant A
When carga cualquier página del dashboard
Then el tema visual (logo, colores) corresponde exclusivamente a la
  configuración del tenant A, resuelto en el servidor
```

## Requisitos de Cumplimiento Normativo Asociados

Esta es la superficie donde se exponen "métricas financieras y contactos" — datos explícitamente sensibles según el informe (§6.1). El control de acceso basado en roles y la verificación de tenant en cada request son la aplicación directa de esa postura de seguridad a nivel de interfaz.

## Fuera de Alcance

- Vista de intervención humana en conversaciones en vivo — vive en el mismo dashboard pero se especifica aparte por su complejidad propia (SPEC-008).
