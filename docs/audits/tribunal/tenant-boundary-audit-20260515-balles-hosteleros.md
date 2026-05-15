# Auditoría SaaS Admin vs Tenant Data

## Resumen ejecutivo

Balles-Hosteleros contempla una frontera tenant de forma parcial: existen `empresas`, múltiples columnas `empresa_id`, políticas RLS, roles de aplicación y un selector de empresa activa.
La implementación auditada no demuestra una separación completa entre plano SaaS/plataforma y plano tenant: no se encontró un backoffice/superadmin de plataforma con privilegios acotados, ni tablas de billing, lifecycle, soporte auditado o accesos excepcionales.
El modelo depende de `public.user_empresas` para membresía multiempresa en código, RLS y acciones, pero no se encontró una migración reproducible que cree la tabla ni sus políticas RLS.
Hay caminos de runtime con `SUPABASE_SERVICE_ROLE_KEY` en middleware, acciones de administración, acciones de empresa, cron jobs y lecturas públicas; varios de esos caminos no aplican un filtro tenant explícito antes de leer o modificar datos.
El middleware considera todo `/api/*` como ruta pública, por lo que la frontera queda delegada a cada route handler; varios cron handlers usan service role y solo exigen secreto si `CRON_SECRET` está configurado.
La rama auditada introduce `estudios_apertura` con un patrón de scoping por `empresa_id` y storage por tenant, pero ese patrón depende de la tabla de membresía no encontrada.
El veredicto tenant-boundary es: `Lo contempla parcialmente, pero faltan piezas importantes`.

## Evidencias encontradas

Confirmado por migraciones, código o documentación:

- `BUSINESS_LOGIC.md` declara el objetivo multiempresa, el uso de `empresa_id`, la tabla conceptual `empresa_usuarios` y el selector de empresa activa como invariant de negocio.
- `supabase/migrations/002_align_profiles_and_roles.sql:9` crea `public.empresas`; `supabase/migrations/002_align_profiles_and_roles.sql:16` habilita RLS; `supabase/migrations/002_align_profiles_and_roles.sql:18` define `"Authenticated can view empresas"` con `using (true)`, permitiendo lectura de todas las empresas por cualquier usuario autenticado.
- `supabase/migrations/002_align_profiles_and_roles.sql:31` añade `profiles.empresa_id`; `supabase/migrations/002_align_profiles_and_roles.sql:58` crea `public.user_roles`; `supabase/migrations/002_align_profiles_and_roles.sql:68` permite que cada usuario vea sus propios roles.
- `supabase/migrations/033_empresa_config.sql:6` crea `public.empresa_roles`; `supabase/migrations/033_empresa_config.sql:36` y `supabase/migrations/033_empresa_config.sql:45` aplican RLS por `profiles.empresa_id`, no por una membresía multiempresa.
- `supabase/migrations/031_ajustes_audit.sql:4` crea `public.audit_log` con `empresa_id`; `supabase/migrations/031_ajustes_audit.sql:19` restringe lectura del audit log por `profiles.empresa_id`.
- `supabase/migrations/060_accesos_apps.sql:7` documenta `accesos_apps` como tabla con `empresa_slug`, usuario y contraseña; `supabase/migrations/060_accesos_apps.sql:55` y `supabase/migrations/060_accesos_apps.sql:62` dejan RLS abierto a autenticados con `using (true)` y `with check (true)`.
- `supabase/migrations/090_fix_rls_always_true_grupo_a.sql:8` deja explícitamente aparcadas tablas pendientes de rediseño RLS, incluyendo `cronogramas_operativos`, `tareas`, `accesos_apps`, `escandallos_config_grupos`, `nueva_receta_gatekeeper`, `nueva_receta_sub_estado`, `carta_item_likes` y `empresa_logos`.
- `supabase/migrations/099_estudios_apertura.sql:21` crea `public.estudios_apertura` con `empresa_id`; `supabase/migrations/099_estudios_apertura.sql:48` y `supabase/migrations/099_estudios_apertura.sql:57` restringen lectura y gestión por `public.user_empresas` o `profiles.empresa_id`.
- `supabase/migrations/099_estudios_apertura.sql:111` crea políticas de storage para `estudios-apertura-fotos`; `supabase/migrations/099_estudios_apertura.sql:121` comprueba que el primer segmento del path coincida con una empresa permitida por `public.user_empresas` o `profiles.empresa_id`.
- `supabase/migrations/20260509081727_estudios_apertura_compartir.sql:31` crea `estudios_apertura_public_read` para `anon` con condición `share_active = true AND share_slug IS NOT NULL`; la política no limita por slug concreto dentro de RLS.
- `src/features/direccion/actions/estudios-apertura-actions.ts:75` resuelve contexto con `getEmpresaActivaForUser`; `src/features/direccion/actions/estudios-apertura-actions.ts:202` lista estudios con `.eq("empresa_id", empresaId)`; `src/features/direccion/actions/estudios-apertura-actions.ts:242` inserta `empresa_id: empresaId`.
- `src/features/direccion/services/estudio-publico-fetch.ts:1` declara lectura pública vía service role para saltar RLS; `src/features/direccion/services/estudio-publico-fetch.ts:30` crea cliente con `SUPABASE_SERVICE_ROLE_KEY`; `src/features/direccion/services/estudio-publico-fetch.ts:86` consulta por `share_slug`.
- `src/lib/supabase/proxy.ts:14` trata cualquier ruta `/api/*` como pública; `src/lib/supabase/proxy.ts:80` permite rutas públicas sin exigir sesión.
- `src/proxy.ts:66` contiene bypass hardcoded para `ricardosilva211@gmail.com`; `src/proxy.ts:71` usa `SUPABASE_SERVICE_ROLE_KEY` en el proxy; `src/proxy.ts:73` omite la verificación modular si falta service role; `src/proxy.ts:117` otorga bypass total al rol `director`.
- `src/lib/supabase/get-context.ts:25` resuelve `empresaId` desde `profiles.empresa_id`; `src/features/empresa/lib/empresa-server.ts:13` resuelve empresa activa desde cookie `bh_empresa_activa` y valida contra `user_empresas` o `profiles.empresa_id`.
- `src/features/empresa/contexts/empresa-context.tsx:150` hace fallback a todas las empresas cuando no hay filas en `user_empresas`; el comentario de `src/features/empresa/contexts/empresa-context.tsx:152` lo describe como cuenta legacy o admin global.
- `src/lib/supabase/admin.ts:4` centraliza `createAdminClient()` con `SUPABASE_SERVICE_ROLE_KEY`, cliente que bypassa RLS.
- `src/actions/admin.ts:55` define `requireAdmin` con roles `admin` o `director`; `src/actions/admin.ts:170` ejecuta `getEmployees()` con admin client y selecciona `profiles` sin filtro `empresa_id`; `src/actions/admin.ts:218` permite `resetEmployeePassword(userId, newPassword)` tras `requireAdmin` sin comprobar pertenencia tenant del `userId`.
- `src/actions/admin.ts:250` genera enlace de recuperación por `profileId`; `src/actions/admin.ts:265` selecciona el perfil objetivo por id usando admin client sin `.eq("empresa_id", empresaDelInvocador)`.
- `src/features/empresa/actions/empresas-actions.ts:113` crea empresas con admin client; `src/features/empresa/actions/empresas-actions.ts:151` borra empresas con admin client; no se ve una guarda explícita de superadmin/plataforma en esas funciones.
- `src/features/empresa/actions/user-empresas-actions.ts:38` permite sustituir empresas de un usuario y delega seguridad a RLS; `src/features/empresa/actions/user-empresas-actions.ts:79` añade/quita accesos sin guarda explícita; `src/features/empresa/actions/user-empresas-actions.ts:124` lista todas las membresías con admin client.
- `src/features/auth/actions/permisos-actions.ts:27` contiene bypass hardcoded para el email `ricardosilva211@gmail.com`; `src/features/auth/actions/permisos-actions.ts:52` usa admin client para leer `profiles`, `user_roles` y `empresa_roles`.
- `src/app/api/cron/agora-sync/route.ts:19` solo exige `CRON_SECRET` si está configurado; `src/app/api/cron/agora-sync/route.ts:33` usa service role; `src/app/api/cron/agora-sync/route.ts:40` acepta `empresa_id` por query o procesa todas las empresas.
- `src/app/api/cron/psd2-sync/route.ts:20` solo exige `CRON_SECRET` si está configurado; `src/app/api/cron/psd2-sync/route.ts:29` selecciona cuentas bancarias activas de todas las empresas con service role.
- `src/app/api/cron/firmas-expirar/route.ts:9` solo exige `CRON_SECRET` si está configurado; `src/app/api/cron/firmas-expirar/route.ts:28` usa service role para actualizar firmas globalmente.
- `src/app/api/cron/cerrar-fichajes-huerfanos/route.ts:16` solo exige `CRON_SECRET` si está configurado; `src/app/api/cron/cerrar-fichajes-huerfanos/route.ts:30` actualiza fichajes huérfanos globalmente sin filtro `empresa_id`.

Inferido a partir de las evidencias anteriores:

- La intención de producto es multiempresa real, no single tenant, porque hay selector de empresa activa, memberships, `empresa_roles`, `empresa_id` y storage segmentado por empresa.
- `director` y `admin` funcionan como roles operativos de tenant o negocio, no como superadmin SaaS con límites de plataforma definidos.
- El uso recurrente de service role parece compensar RLS insuficiente para casos multiempresa, en vez de existir una capa privilegiada con autorización tenant explícita y auditada.
- La ruta pública de estudios compartidos intenta exponer un recurso por slug, pero la política anon permite leer cualquier fila activa compartida si se accede por cliente Supabase con permisos anon.

No encontrado en la auditoría:

- No se encontró una migración `CREATE TABLE public.user_empresas` ni políticas RLS reproducibles para `public.user_empresas`, aunque el código y varias políticas dependen de ella.
- No se encontró la tabla conceptual `empresa_usuarios` mencionada en `BUSINESS_LOGIC.md`.
- No se encontró un rol o tabla dedicada `superadmin`, `super_admin`, `platform_admin`, `internal_admin`, `saas_admin` o equivalente.
- No se encontró un backoffice SaaS separado del plano tenant para lifecycle de tenants, billing, planes, estado contractual o soporte.
- No se encontraron tablas específicas de `subscriptions`, `plans`, `billing_accounts`, `support_access_grants`, `support_cases`, `tenant_status` o auditoría de acceso excepcional de soporte.
- No se encontró una política general que prohíba a superadmin/plataforma leer datos tenant por defecto y fuerce break-glass con ticket, motivo, alcance y expiración.

## Qué sí está resuelto

- Existe una entidad `public.empresas` que actúa como eje principal de tenant.
- Muchas tablas operativas tienen columna `empresa_id` y FKs hacia `public.empresas`.
- Hay RLS habilitado en tablas importantes como `profiles`, `empresas`, `user_roles`, `empresa_roles`, `audit_log` y `estudios_apertura`.
- La funcionalidad nueva de `estudios_apertura` filtra por `empresa_id` en server actions y define políticas de tabla y storage que intentan validar membresía tenant.
- Existe una abstracción `getEmpresaActivaForUser` que valida la empresa activa contra `user_empresas` o `profiles.empresa_id`.
- Existe un modelo de roles de aplicación en `user_roles` y un modelo de permisos de empresa en `empresa_roles`.
- Existe `audit_log` tenant-scoped para ciertos cambios de ajustes.
- Algunas rutas cron del módulo points aplican un patrón más seguro al exigir secreto en producción o cabecera de Vercel Cron, lo que sirve como referencia interna de endurecimiento.

## Qué no está resuelto o está incompleto

- La tabla crítica `public.user_empresas` no está definida de forma reproducible en migraciones, aunque es dependencia directa de RLS, selector de empresa activa, acciones de permisos y creación de empleados.
- Hay dos resolutores de contexto tenant incompatibles: `src/lib/supabase/get-context.ts` usa `profiles.empresa_id` y `src/features/empresa/lib/empresa-server.ts` usa empresa activa validada.
- La UI de empresa activa puede caer a listar todas las empresas si no hay filas de `user_empresas`, lo que debilita la semántica de membership.
- `src/actions/admin.ts` usa service role para leer perfiles globalmente y operar sobre usuarios por id sin filtro tenant explícito en todas las acciones sensibles.
- `src/features/empresa/actions/empresas-actions.ts` permite crear y borrar empresas con service role sin una guarda visible de superadmin/plataforma.
- `src/features/empresa/actions/user-empresas-actions.ts` gestiona memberships con guardas implícitas o ausentes, y además ofrece lectura global vía admin client.
- Varios cron handlers son públicos a nivel middleware, usan service role y solo validan secreto si `CRON_SECRET` existe.
- `public.empresas` es legible por cualquier autenticado mediante `using (true)`, lo que puede exponer el directorio de tenants.
- `public.accesos_apps` conserva RLS abierta a autenticados y contiene credenciales por `empresa_slug`.
- No hay separación explícita entre datos de plataforma SaaS y datos operativos tenant.
- No hay superadmin/backoffice con capacidades acotadas, ni soporte excepcional auditado.
- La política anon de estudios compartidos permite leer cualquier estudio con `share_active = true` y `share_slug IS NOT NULL`, en vez de limitar la exposición al slug solicitado mediante una función o endpoint controlado.

## Riesgos de diseño detectados

- Crítico: un usuario con rol `admin` o `director` puede activar acciones que usan service role y leen o modifican datos fuera de su tenant si la acción no aplica filtro tenant propio.
- Crítico: el modelo multiempresa depende de `user_empresas`, pero la tabla no es parte del esquema reproducible; esto puede romper RLS, despliegues nuevos, tests y límites reales de tenant.
- Alto: cron endpoints con service role y secreto opcional pueden ejecutar operaciones globales cross-tenant si `CRON_SECRET` está ausente o mal configurado.
- Alto: rutas `/api/*` son públicas en middleware, por lo que cualquier omisión de auth en un handler se convierte en superficie directa.
- Alto: `accesos_apps` almacena credenciales y mantiene RLS abierta a todos los autenticados, con aislamiento por aplicación declarado pero no por política.
- Alto: `empresas` se usa a la vez como catálogo tenant y dato operativo, y es visible globalmente para autenticados.
- Medio: el fallback de la UI a todas las empresas cuando no hay memberships puede ocultar errores de datos y convertir cuentas legacy en acceso global de facto.
- Medio: la lectura pública por slug de estudios está minimizada en servicio, pero la política anon de tabla es más amplia que el contrato de URL pública.
- Medio: los bypasses hardcoded por email reducen auditabilidad y no expresan un límite formal de superadmin.
- Medio: no hay bitácora específica para accesos privilegiados de soporte o service role a datos tenant.

## Veredicto final

`Lo contempla parcialmente, pero faltan piezas importantes`

La aplicación tiene una base tenant real: `empresas`, `empresa_id`, RLS, roles, selector de empresa activa y acciones que filtran por empresa en varias áreas. Sin embargo, no alcanza una frontera SaaS madura porque la membresía multiempresa no está definida de forma reproducible, hay uso amplio de service role en runtime, existen acciones administrativas sin filtros tenant consistentes, varias rutas cron operan globalmente, y no hay plano de plataforma/superadmin separado con privilegios mínimos y auditoría de soporte.

## Arquitectura correcta recomendada

Plano plataforma SaaS:

- Crear un plano de plataforma explícito para lifecycle de tenants, billing, planes, estado contractual, límites, feature flags y soporte.
- Separar metadatos de plataforma de datos operativos tenant; `empresas` puede seguir siendo tenant, pero debe distinguirse de tablas de plataforma como `platform_tenants`, `platform_admins`, `tenant_status`, `billing_accounts` y `subscriptions`.
- Definir roles de plataforma separados de roles tenant; `director` y `admin` no deben equivaler a superadmin SaaS.
- Registrar toda acción privilegiada en un `platform_audit_log` inmutable con actor, tenant, recurso, motivo, ticket, origen, timestamp y resultado.

Plano tenant:

- Hacer de `user_empresas` o una tabla equivalente la fuente canónica de membresía tenant, con migración, constraints, RLS, backfill y tests.
- Usar funciones SQL estables como `is_member_of_empresa(empresa_id)` y `current_active_empresa_id()` para políticas RLS y queries.
- Mantener `profiles.empresa_id` solo como empresa principal o compatibilidad legacy, no como fuente única de autorización.
- Exigir `empresa_id` en toda tabla operativa tenant, storage path y operación de escritura sensible.
- Evitar `using (true)` en tablas tenant salvo que el dato sea deliberadamente global y no sensible.

Límites del superadmin:

- El superadmin SaaS debe poder administrar tenants, planes, estado contractual, límites, integraciones de plataforma y usuarios de plataforma.
- El superadmin SaaS no debe leer datos operativos tenant por defecto.
- La creación, baja, reset de credenciales o reasignación de usuarios tenant debe comprobar explícitamente el tenant afectado y registrar la acción.
- Las acciones de soporte deben estar separadas de acciones tenant normales y no depender de emails hardcoded.

Soporte excepcional auditado:

- Implementar break-glass con concesiones temporales por tenant, alcance, motivo, ticket externo, aprobador y expiración.
- Limitar el acceso excepcional a tablas, recursos o acciones concretas, no al service role global.
- Exigir logging inmutable de cada lectura o modificación hecha por soporte.
- Revocar automáticamente concesiones expiradas y alertar sobre accesos fuera de horario, alcance o motivo.

## Plan de refactor por fases

Fase 1: estabilizar la base de membresía tenant.

- Objetivo: hacer reproducible el modelo multiempresa.
- Cambios: crear migración de `user_empresas`, constraints, índices, RLS, backfill desde `profiles.empresa_id` y tests de dos tenants.
- Impacto: bajo en UI si se mantiene compatibilidad temporal con `profiles.empresa_id`.
- Riesgo de no hacerlo: RLS y selector multiempresa dependerán de una tabla inexistente en despliegues limpios.

Fase 2: unificar resolución de tenant activo.

- Objetivo: eliminar divergencias entre `getAppContext`, `getEmpresaActivaForUser`, server actions y UI.
- Cambios: centralizar helper server-only, validar cookie contra membership, eliminar fallback a todas las empresas salvo superadmin formal.
- Impacto: medio porque obliga a revisar acciones existentes.
- Riesgo de no hacerlo: accesos inconsistentes según módulo y regresiones cross-tenant difíciles de detectar.

Fase 3: cerrar RLS abierta y políticas legacy.

- Objetivo: sustituir `using (true)` y políticas basadas solo en `profiles.empresa_id`.
- Cambios: migrar `accesos_apps`, tablas aparcadas en `090_fix_rls_always_true_grupo_a.sql`, `empresa_roles`, storage y tablas operativas a funciones de membership.
- Impacto: medio-alto por riesgo de romper flujos legacy.
- Riesgo de no hacerlo: datos tenant y credenciales pueden ser visibles o modificables por usuarios autenticados de otros tenants.

Fase 4: contener service role.

- Objetivo: que service role no sea un bypass funcional de autorización tenant.
- Cambios: envolver admin client en funciones server-only con `actor_user_id`, `empresa_id`, razón y comprobaciones; añadir filtros tenant obligatorios; eliminar lecturas globales no justificadas.
- Impacto: alto en acciones administrativas y cron.
- Riesgo de no hacerlo: cualquier acción mal expuesta puede convertirse en fuga o mutación cross-tenant.

Fase 5: crear plano plataforma y soporte auditado.

- Objetivo: separar administración SaaS de administración tenant.
- Cambios: introducir roles/tablas de plataforma, backoffice acotado, soporte break-glass, lifecycle tenant, billing/planes y audit log de plataforma.
- Impacto: alto pero incremental si se crea paralelo al plano tenant actual.
- Riesgo de no hacerlo: se seguirá usando `director/admin` y service role como sustitutos de superadmin, sin límites verificables.

Fase 6: endurecer APIs públicas, cron y enlaces compartidos.

- Objetivo: reducir superficie pública y exposición anon.
- Cambios: exigir auth por handler o middleware segmentado, hacer `CRON_SECRET` obligatorio en producción, eliminar `empresa_id` arbitrario por query, mover share público a RPC/endpoint controlado.
- Impacto: medio.
- Riesgo de no hacerlo: endpoints públicos con service role seguirán siendo vectores de operación global.

Fase 7: validar con pruebas y auditoría continua.

- Objetivo: convertir tenant-boundary en garantía automática.
- Cambios: matriz de tests con tenant A/B, usuario multiempresa, tenant admin, platform admin, anon, cron y service role; checks de migraciones para `using (true)` y tablas sin `empresa_id`.
- Impacto: medio.
- Riesgo de no hacerlo: las regresiones de aislamiento volverán a aparecer en nuevas features.

## Checklist accionable para implementación

TAREA-01

- Objetivo: crear fuente canónica reproducible de membresía tenant.
- Tipo: migración y RLS.
- Archivos probables: `supabase/migrations/*_create_user_empresas.sql`, tipos generados y seeds.
- Cambios concretos: crear `public.user_empresas(user_id, empresa_id, rol, activo, created_at)`, FKs, índice único, RLS por `auth.uid()`, backfill desde `profiles.empresa_id`.
- Dependencias: revisión de datos legacy y decisión de si `profiles.empresa_id` queda como empresa principal.
- Criterio de validación: despliegue limpio crea tabla; usuario A no puede leer membresías de usuario B salvo política explícita; tests A/B pasan.
- Prioridad: bloqueante.

TAREA-02

- Objetivo: unificar funciones de autorización tenant.
- Tipo: migración SQL y refactor backend.
- Archivos probables: `supabase/migrations/*_tenant_auth_helpers.sql`, `src/features/empresa/lib/empresa-server.ts`, `src/lib/supabase/get-context.ts`.
- Cambios concretos: crear funciones `is_member_of_empresa(uuid)`, `can_manage_empresa(uuid)` y helper server-only único para empresa activa.
- Dependencias: TAREA-01.
- Criterio de validación: ningún módulo nuevo usa `profiles.empresa_id` como única comprobación de autorización.
- Prioridad: bloqueante.

TAREA-03

- Objetivo: cerrar políticas RLS abiertas o legacy.
- Tipo: migración de seguridad.
- Archivos probables: `supabase/migrations/*_harden_tenant_rls.sql`, `supabase/migrations/060_accesos_apps.sql`, `supabase/migrations/090_fix_rls_always_true_grupo_a.sql`.
- Cambios concretos: sustituir `using (true)` en tablas tenant por funciones de membership; rediseñar `accesos_apps` para usar `empresa_id` y no `empresa_slug` como frontera.
- Dependencias: TAREA-01 y TAREA-02.
- Criterio de validación: búsqueda de `using (true)` no devuelve tablas tenant sensibles sin excepción documentada.
- Prioridad: bloqueante.

TAREA-04

- Objetivo: eliminar lecturas globales no autorizadas en acciones admin.
- Tipo: refactor server actions.
- Archivos probables: `src/actions/admin.ts`, `src/features/empresa/actions/empresas-actions.ts`, `src/features/empresa/actions/user-empresas-actions.ts`.
- Cambios concretos: exigir `actor`, `empresa_id` y permisos; filtrar `profiles`, `user_roles` y `user_empresas` por tenant; comprobar pertenencia del usuario objetivo antes de reset, baja o actualización.
- Dependencias: TAREA-02.
- Criterio de validación: un admin del tenant A no puede listar, resetear ni modificar usuarios del tenant B.
- Prioridad: bloqueante.

TAREA-05

- Objetivo: contener `SUPABASE_SERVICE_ROLE_KEY`.
- Tipo: hardening runtime.
- Archivos probables: `src/lib/supabase/admin.ts`, `src/proxy.ts`, acciones bajo `src/features/**/actions`, rutas bajo `src/app/api/**/route.ts`.
- Cambios concretos: prohibir admin client directo salvo wrappers privilegiados; registrar motivo y alcance; añadir lint/check de importaciones de `createAdminClient`.
- Dependencias: TAREA-04.
- Criterio de validación: todo uso de service role tiene autorización previa explícita y test de aislamiento tenant.
- Prioridad: alta.

TAREA-06

- Objetivo: separar plano plataforma SaaS.
- Tipo: diseño de datos y backoffice.
- Archivos probables: nuevas migraciones `platform_*`, rutas `src/app/platform/**`, servicios de billing/lifecycle.
- Cambios concretos: crear roles/tables de plataforma, estado tenant, planes, suscripciones, límites y auditoría de plataforma.
- Dependencias: decisión de producto sobre billing y soporte.
- Criterio de validación: un platform admin puede gestionar lifecycle sin leer datos operativos tenant por defecto.
- Prioridad: alta.

TAREA-07

- Objetivo: formalizar soporte excepcional auditado.
- Tipo: seguridad y compliance.
- Archivos probables: migraciones de `support_access_grants`, `platform_audit_log`, wrappers de acceso privilegiado.
- Cambios concretos: implementar break-glass con tenant, alcance, ticket, motivo, aprobador, expiración y logs inmutables.
- Dependencias: TAREA-06.
- Criterio de validación: no existe acceso de soporte a datos tenant sin concesión vigente y evento de auditoría.
- Prioridad: alta.

TAREA-08

- Objetivo: endurecer cron y APIs públicas.
- Tipo: seguridad de rutas.
- Archivos probables: `src/lib/supabase/proxy.ts`, `src/app/api/cron/*/route.ts`, `vercel.json`.
- Cambios concretos: exigir secreto en producción, validar `x-vercel-cron` cuando aplique, rechazar `empresa_id` arbitrario por query y mover lógica cross-tenant a jobs internos auditados.
- Dependencias: inventario de cron jobs productivos.
- Criterio de validación: cron sin secreto en producción devuelve 401 y no ejecuta service role.
- Prioridad: alta.

TAREA-09

- Objetivo: limitar exposición de estudios compartidos.
- Tipo: RLS/API pública.
- Archivos probables: `supabase/migrations/20260509081727_estudios_apertura_compartir.sql`, `src/features/direccion/services/estudio-publico-fetch.ts`.
- Cambios concretos: eliminar lectura anon directa de tabla o reemplazarla por RPC que exige slug; mantener endpoint server-side con campos mínimos y URLs firmadas.
- Dependencias: decisión sobre compatibilidad de enlaces públicos.
- Criterio de validación: cliente anon no puede enumerar todos los estudios compartidos activos.
- Prioridad: media-alta.

TAREA-10

- Objetivo: crear suite de regresión tenant-boundary.
- Tipo: tests automatizados.
- Archivos probables: `tests/tenant-boundary/*.spec.ts`, scripts de fixtures Supabase, CI.
- Cambios concretos: probar tenant A/B, usuario multiempresa, admin tenant, platform admin, anon, cron, storage y acciones con service role.
- Dependencias: TAREA-01 a TAREA-09.
- Criterio de validación: CI falla ante lectura o mutación cross-tenant no autorizada.
- Prioridad: alta.
