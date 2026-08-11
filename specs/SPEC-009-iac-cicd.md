# SPEC-009: Aprovisionamiento e Infraestructura como Código

> Implementa la estrategia de despliegue definida en el informe §9: Terraform + GitHub Actions con Workload Identity Federation, despliegue progresivo sobre Cloud Run.

## Contexto y objetivo

Define cómo se crea, versiona y despliega toda la infraestructura de la plataforma, y cómo se aprovisiona un tenant nuevo de forma reproducible — reemplazando el "script Bash/Ansible" vago del documento original por un pipeline concreto y auditable.

## Requisitos Funcionales

| Cód. | Descripción |
|---|---|
| RF-009-01 | Toda la infraestructura (Cloud Run, Cloud SQL, Identity Platform, redes, IAM, Cloud Tasks, KMS) debe estar declarada en Terraform versionado en control de código fuente. |
| RF-009-02 | El alta de un tenant nuevo debe ejecutarse mediante un módulo Terraform parametrizado, sin pasos manuales en la consola de GCP. |
| RF-009-03 | El pipeline de CI/CD debe ejecutar pruebas —incluidas las de aislamiento multi-tenant de SPEC-006— como *gate* obligatorio antes de cualquier despliegue a producción. |
| RF-009-04 | El despliegue a producción debe usar división de tráfico progresiva entre revisiones de Cloud Run. |
| RF-009-05 | Los secretos (credenciales del BSP, llaves de API de Anthropic, etc.) se gestionan mediante Secret Manager de GCP, nunca en texto plano en el repositorio. |
| RF-009-06 | El alta de un tenant nuevo debe quedar bloqueada si no existe `dpa_signed_at` registrado (hereda RF-000-09 de SPEC-000). |

## Requisitos No Funcionales

| Cód. | Criterio de aceptación |
|---|---|
| RNF-009-01 | GitHub Actions se autentica hacia GCP mediante Workload Identity Federation — sin llaves de service account estáticas almacenadas como secreto. |
| RNF-009-02 | Todo cambio de infraestructura pasa por revisión (pull request) antes de aplicarse; ningún `terraform apply` se ejecuta directo sobre producción sin ese paso. |

## Estructura de Módulos

```
infra/
  modules/
    cloud-run-service/
    cloud-sql-postgres/
    identity-platform/
    networking-vpc/
    kms-tenant-keys/
    tenant-provisioning/      -- orquesta los módulos anteriores por tenant
  environments/
    dev/
    staging/
    production/
```

## Contrato del Pipeline

```
.github/workflows/deploy.yml

  on: pull_request → 
    - lint + tests unitarios/integración
    - suite de aislamiento multi-tenant (SPEC-006) — bloqueante
    - terraform plan (comentado automáticamente en el PR)

  on: merge a main →
    - build de imagen de contenedor
    - terraform apply sobre "staging"
    - suite de humo (smoke tests) sobre staging

  promoción a producción (manual o automática según política del equipo) →
    - terraform apply sobre "production"
    - despliegue de la nueva revisión de Cloud Run con tráfico inicial 5%
    - monitoreo de métricas de error/latencia (SPEC-010) durante ventana definida
    - progresión automática 5% → 25% → 100% si no hay alertas activas
    - rollback inmediato a la revisión anterior si se dispara una alerta
```

## Flujo de Alta de Tenant Nuevo

```
1. Se registra el tenant en SPEC-000 (tenants.dpa_signed_at obligatorio).
2. Se ejecuta el módulo tenant-provisioning (vía script wrapper sobre Terraform):
   - Crea la llave KMS dedicada del tenant.
   - Registra el número de WhatsApp en whatsapp_numbers (pendiente de
     activación real en el BSP, paso externo).
   - Crea el usuario inicial en Identity Platform (dashboard_users).
3. Si tenants.dpa_signed_at es NULL, el proceso se bloquea antes de crear
   cualquier recurso — no es una validación opcional de UI.
```

## Casos Borde

| Caso | Comportamiento esperado |
|---|---|
| Fallo a mitad de un `terraform apply` (recurso parcialmente creado) | El estado de Terraform permite reintentar de forma idempotente sin duplicar recursos. |
| Secreto rotado en Secret Manager sin redeploy | Los servicios en Cloud Run recogen el nuevo valor sin downtime (montaje dinámico o reinicio controlado, según el mecanismo de la plataforma). |
| Un PR modifica infraestructura de un tenant específico | El plan de Terraform muestra un diff acotado a ese tenant; un módulo mal parametrizado que afectara a todos los tenants debe ser detectable en la revisión del plan antes de aplicar. |
| Alerta de error/latencia durante la progresión de tráfico (5%→25%→100%) | Rollback automático a la revisión anterior, sin intervención manual requerida para contener el impacto. |

## Criterios de Aceptación

```
Given un pull request que modifica infraestructura
When se abre
Then el pipeline ejecuta terraform plan y lo publica como comentario
  And no aplica ningún cambio hasta que el PR sea aprobado y fusionado

Given un intento de alta de tenant sin DPA firmado
When se ejecuta el módulo de aprovisionamiento
Then el proceso se detiene antes de crear cualquier recurso de infraestructura

Given una nueva revisión desplegada a producción con tráfico al 5%
When se detecta un incremento de errores por encima del umbral configurado
Then el tráfico revierte automáticamente a la revisión anterior
  And no se continúa la progresión hacia 25%/100%

Given un cambio de secreto en Secret Manager
When los servicios en Cloud Run lo requieren
Then lo obtienen sin necesidad de un despliegue completo del servicio
```

## Requisitos de Cumplimiento Normativo Asociados

RF-009-06 aplica a nivel de infraestructura el bloqueo de alta sin DPA definido en SPEC-000 (RF-000-09), y las pruebas de aislamiento de este pipeline son la garantía operativa concreta detrás de RNF-05 del informe.

## Fuera de Alcance

- Contenido específico de los tests de aislamiento (se definen junto a SPEC-006).
- Políticas de presupuesto/alertas de costo en GCP (gestión operativa, no arquitectura).
