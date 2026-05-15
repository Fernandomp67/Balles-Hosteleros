---
schema_version: "1.0"
id: "security-audit-20260515-balles-hosteleros"
tipo: "security"
componente: "Balles Hosteleros SaaS"
alcance: "Auditoria TRIBUNAL de seguridad sobre worktree reproducible"
status: "audited"
created_at: "2026-05-15"
updated_at: "2026-05-15"
auditor:
  agent: "security-auditor"
  model: "GPT-5"
  worktree: "/tmp/balles-hosteleros-audit"
  branch: "upstream-feature-aperturas-fotos-ocupacion"
  head: "32cd5c3"
verdict: "FAIL"
blockers_count: 9
---

## 🔒 Security Audit Report

**Fecha**: 2026-05-15  
**Alcance**: Completo, TRIBUNAL security-auditor  
**Base auditada**: `/tmp/balles-hosteleros-audit`, branch `upstream-feature-aperturas-fotos-ocupacion`, HEAD `32cd5c3`  
**Veredicto**: ❌ FAIL — 9 bloqueantes

### Contexto y límites

- Auditoría estática de código, migraciones Supabase y configuración versionada.
- No se modificó código de producto, no se ejecutaron fixes y no se revirtieron cambios.
- Validación dinámica previa aportada por Capataz: `npm ci` OK; `npm run lint` falla por `next lint`; `tsc`/build fallan por tipos RRHH; `dev` levanta.
- No se verificó la configuración real de Supabase/Vercel en producción. Cuando una protección puede depender de infraestructura externa, queda indicada como verificación pendiente.

### Resultados

| Área | Estado | Hallazgos |
|------|--------|-----------|
| Aislamiento de datos | ❌ | RLS abierta en `accesos_apps`; política anon enumerables en `estudios_apertura`; service role usado para saltarse RLS sin controles suficientes. |
| Secrets | ❌ | Token real de Ágora POS versionado en documentación y memoria `.claude`; aparece también en historial git. |
| Input Validation | ❌ | Algunas rutas usan Zod correctamente, pero hay SSRF por URL arbitraria y varias server actions aceptan IDs/slugs sin autorización ni validación de ownership. |
| Auth | ❌ | Backdoor de autorización por email fijo; `/api` queda fuera del proxy; endpoints IA públicos sin auth/rate limit; crons fail-open si falta `CRON_SECRET`. |
| Env Vars | ⚠️ | `.env.local` está ignorado; no se encontraron service-role keys reales versionadas. Variables críticas existen en `.env.example`; `NEXT_PUBLIC_DEV_BYPASS_AUTH` debe bloquearse en producción. |
| Headers/OWASP | ❌ | Sin baseline global visible de CSP/nosniff/referrer/permissions; `/empleo` permite `frame-ancestors *`; XSS confirmado en HTML de Gmail. |
| Reglas de incidentes reales | ❌ | `redirectTo` interno razonable, pero se incumplen reglas de service role, crons, errores/logs y renderizado HTML no sanitizado. |

### Bloqueantes

| # | Problema | Severidad | Acción requerida |
|---|----------|-----------|------------------|
| 1 | Backdoor de autorización por email fijo en el proxy. | 🔴 Crítica | Eliminar bypass hardcoded y reemplazar por roles/permisos verificables. |
| 2 | Crons con `service_role` quedan públicos si `CRON_SECRET` no está configurado. | 🔴 Crítica | Fallar cerrado en producción y exigir secreto o header Vercel validado. |
| 3 | `accesos_apps` expone credenciales y permite lectura/escritura cross-tenant. | 🔴 Crítica | Rediseñar RLS por `empresa_id`/membership y cifrar/no exponer contraseñas. |
| 4 | Server actions con `service_role` no autentican ni comprueban ownership. | 🔴 Crítica | Añadir `getUser()`, autorización por rol/empresa y validación de inputs en cada action. |
| 5 | Política pública de estudios compartidos permite enumerar todos los estudios `share_active`. | 🔴 Alta | Eliminar policy anon directa o exponer solo vía route server-side con slug/token no enumerable. |
| 6 | SSRF autenticado en importador de webs. | 🔴 Alta | Validar destinos con allowlist/bloqueo de IPs internas, esquemas y redirects. |
| 7 | XSS por renderizar HTML de Gmail/firma sin sanitizar. | 🔴 Crítica | Sanitizar HTML server/client con allowlist estricta antes de `dangerouslySetInnerHTML`. |
| 8 | Endpoints IA de soporte públicos, sin auth ni rate limit. | 🔴 Alta | Requerir sesión o limitar públicamente por IP/Turnstile/cuota antes de llamar al proveedor IA. |
| 9 | Secret real de Ágora POS versionado. | 🔴 Crítica | Rotar token con el partner y purgar/redactar documentación e historial sensible. |

## Hallazgos Detallados

### SEC-001 — Backdoor de autorización por email fijo

**Severidad**: Critical  
**Ubicación**: `src/proxy.ts:66-69`

**Evidencia**:

```ts
// Bypass local para el usuario de dirección
if (user.email === 'ricardosilva211@gmail.com') {
  return sessionResponse
}
```

**Impacto**: cualquier sesión que controle ese email salta la autorización por módulo antes de revisar permisos de `empresa_roles`. Esto convierte una identidad concreta en superusuario implícito, fuera del modelo RBAC y sin auditoría de rol.

**Fix**: eliminar el bypass y expresar el acceso total mediante `user_roles.role = 'director'` o una política equivalente gestionada en BD.

**Mitigación**: si se necesita un break-glass account, documentarlo, limitarlo por entorno, MFA/SSO, logging explícito y caducidad.

**False positive notes**: el comentario dice "local", pero no hay guard por `NODE_ENV !== 'production'`.

### SEC-002 — Crons fail-open con `service_role`

**Severidad**: Critical  
**Ubicación**:

- `src/app/api/cron/agora-sync/route.ts:18-35`
- `src/app/api/cron/psd2-sync/route.ts:20-27`
- `src/app/api/cron/cerrar-fichajes-huerfanos/route.ts:16-25`
- `src/app/api/cron/firmas-expirar/route.ts:19-28`
- `vercel.json:2-26`

**Evidencia**:

```ts
const cronSecret = process.env.CRON_SECRET;
if (cronSecret && authHeader !== `Bearer ${cronSecret}`) {
  return NextResponse.json({ error: "No autorizado" }, { status: 401 });
}
```

El patrón anterior permite ejecución si `CRON_SECRET` está ausente. Después se crea cliente admin con `SUPABASE_SERVICE_ROLE_KEY`, por ejemplo `agora-sync` en `src/app/api/cron/agora-sync/route.ts:33-35`.

**Impacto**: un atacante anónimo podría disparar jobs administrativos si falta o se elimina el secreto en producción. Los efectos incluyen descuentos de stock por Ágora, sincronización PSD2, cierre de fichajes y expiración/limpieza de firmas.

**Fix**: `if (!cronSecret) return 500/503` en producción, o validar estrictamente `x-vercel-cron` con controles de plataforma más `CRON_SECRET`. No ejecutar service-role si falta auth.

**Mitigación**: alertar cuando `CRON_SECRET` esté ausente; registrar ejecución con IP/origen; rate limit y allowlist en edge.

**False positive notes**: los endpoints de `points/cron/*` sí implementan patrón fail-closed en producción (`src/app/api/points/cron/devengo-diario/route.ts:18-27`), por lo que hay precedente interno de implementación segura.

### SEC-003 — `accesos_apps` expone credenciales y cruza tenants

**Severidad**: Critical  
**Ubicación**:

- `supabase/migrations/060_accesos_apps.sql:15-27`
- `supabase/migrations/060_accesos_apps.sql:55-67`
- `supabase/migrations/090_fix_rls_always_true_grupo_a.sql:8-11`
- `src/features/rrhh/actions/accesos-apps-actions.ts:69-150`
- `src/features/rrhh/components/AccesosView.tsx:194-198`

**Evidencia**:

```sql
usuario              text not null default '',
contrasena           text not null default '',
...
create policy "accesos_apps_auth_read" ... using (true);
create policy "accesos_apps_auth_write" ... using (true) with check (true);
```

La migración de hardening deja esta tabla aparcada: `accesos_apps: requiere rediseño (empresa_id + rol↔apps)`.

Las actions usan `createAdminClient()` sin auth interna:

```ts
export async function listAllAccesosApps(): Promise<AccesoApp[]> {
  const supabase = createAdminClient();
  const { data, error } = await supabase
    .from("accesos_apps")
    .select("*")
```

La UI muestra contraseña con `canView={true}`.

**Impacto**: cualquier usuario autenticado con el anon key puede leer/escribir todos los accesos por RLS. Además, las server actions saltan RLS con service-role. La tabla contiene usuarios y contraseñas de apps externas, por lo que la exposición es de credenciales, no solo metadata.

**Fix**: migrar a `empresa_id uuid`, RLS por membership (`user_empresas` o `profiles`), permisos por rol, cifrado de secretos, y endpoints/actions que nunca devuelvan `contrasena` salvo flujo explícito auditado.

**Mitigación**: revocar temporalmente lectura de `contrasena`, retirar `listAllAccesosApps` del cliente y rotar cualquier credencial real ya almacenada.

**False positive notes**: el seed inicial tiene contraseñas vacías, pero el modelo y la UI están preparados para almacenar y mostrar secretos reales.

### SEC-004 — Server actions con `service_role` sin autorización interna

**Severidad**: Critical  
**Ubicación**:

- `src/features/auth/actions/avatar-actions.ts:9-35`
- `src/features/auth/components/AvatarPicker.tsx:125-137`
- `src/features/empresa/actions/logo-actions.ts:28-59`
- `src/features/empresa/actions/logo-actions.ts:87-104`
- `src/features/empresa/actions/logo-actions.ts:205-220`

**Evidencia**:

```ts
export async function uploadAvatar(userId: string, formData: FormData): Promise<string> {
  if (!userId) throw new Error("Usuario no identificado.");
  ...
  const supabase = createAdminClient();
  ...
  .from("profiles")
  .update({ avatar_url: publicUrl, avatar_obligatorio: false })
  .eq("user_id", userId);
}
```

El cliente pasa `user.id` desde el navegador:

```ts
const url = await uploadAvatar(user.id, fd);
```

Logo/branding acepta `empresaSlug` y usa admin sin `getUser()`:

```ts
async function uploadVariant(empresaSlug: string, formData: FormData, variant: LogoVariant) {
  const supabase = createAdminClient();
  ...
  .from("empresas")
  .update({ [variantColumn(variant)]: publicUrl })
  .eq("slug", empresaSlug);
}
```

**Impacto**: las Server Actions son endpoints de red. Si una action no comprueba sesión y ownership en el servidor, un cliente manipulado puede intentar actualizar otro perfil o marca de otra empresa saltando RLS con service-role.

**Fix**: en cada action, llamar a `supabase.auth.getUser()`, derivar `userId/empresaId` en servidor, validar rol y membership, ignorar IDs sensibles recibidos del cliente o validarlos contra membership.

**Mitigación**: inventariar todas las actions que importan `createAdminClient()` y bloquear despliegue hasta que cada una tenga guard server-side.

**False positive notes**: que la UI solo pase IDs legítimos no es una barrera de seguridad.

### SEC-005 — Estudios compartidos enumerables por anon

**Severidad**: High  
**Ubicación**:

- `supabase/migrations/20260509081727_estudios_apertura_compartir.sql:31-39`
- `src/features/direccion/services/estudio-publico-fetch.ts:83-98`

**Evidencia**:

```sql
CREATE POLICY "estudios_apertura_public_read" ON public.estudios_apertura FOR SELECT TO anon
  USING (share_active = true AND share_slug IS NOT NULL);
```

El comentario dice que la lectura pública se controla por slug, pero la policy no compara contra un slug de la petición. El fetch server-side sí filtra `.eq("share_slug", slug)` y comprueba `share_active`, pero la policy anon directa permite seleccionar todas las filas activas.

**Impacto**: cualquiera con `NEXT_PUBLIC_SUPABASE_URL` y `NEXT_PUBLIC_SUPABASE_ANON_KEY` puede consultar directamente Supabase y enumerar todos los estudios compartidos activos, que contienen datos de viabilidad, costes, facturación, fotos y marca.

**Fix**: eliminar la policy anon directa y servir el contenido solo por route server-side que filtre `share_slug` y `share_active`, o usar RPC `SECURITY DEFINER` que acepte slug/token y devuelva solo una fila estrictamente seleccionada.

**Mitigación**: revisar todos los estudios con `share_active=true` y regenerar slugs si ya fueron expuestos.

**False positive notes**: no se verificó la BD real; la evidencia es la migración versionada que define el estado esperado.

### SEC-006 — SSRF autenticado en importador de URL

**Severidad**: High  
**Ubicación**: `src/app/api/pagina-web/importar-url/route.ts:16-55`

**Evidencia**:

```ts
const bodySchema = z.object({
  paginaId: z.string().uuid(),
  url: z.string().url(),
});
...
const htmlRes = await fetch(parsed.data.url, {
  headers: { "User-Agent": "Mozilla/5.0 (compatible; BallesHosteleros-Importer/1.0; +https://balleshosteleros.com)" },
  signal: AbortSignal.timeout(20_000),
});
```

**Impacto**: cualquier usuario autenticado con empresa puede hacer que el servidor solicite URLs arbitrarias. Esto permite probing de red interna, metadata services, servicios locales o endpoints cloud no pensados para el usuario.

**Fix**: permitir solo `https`, resolver DNS y bloquear rangos privados/link-local/loopback, limitar redirects, tamaño y content-type, y considerar allowlist por dominio.

**Mitigación**: ejecutar el fetch en un worker aislado sin acceso a red interna y con egress policy.

**False positive notes**: el endpoint valida que la página pertenece a la empresa antes de fetch, pero eso no restringe el destino de red.

### SEC-007 — XSS por HTML de Gmail sin sanitizar

**Severidad**: Critical  
**Ubicación**:

- `src/app/api/google/gmail/message/route.ts:88-103`
- `src/app/api/google/gmail/message/route.ts:142-147`
- `src/features/google-workspace/components/GmailDrawer.tsx:1527-1538`
- `src/features/google-workspace/components/GmailDrawer.tsx:1611-1622`
- `src/features/google-workspace/components/GmailDrawer.tsx:962-977`

**Evidencia**:

```ts
const html = findPart(msg.payload, "text/html");
...
cuerpoHtml: html,
```

```tsx
<div
  ...
  dangerouslySetInnerHTML={{ __html: mensaje.cuerpoHtml }}
/>
```

También se renderiza `firmaHtml` sin sanitizar.

**Impacto**: un email entrante o una firma HTML controlada por terceros puede inyectar HTML en el origen de la app. Un XSS en este contexto puede invocar acciones/API same-origin, leer datos visibles en la página y operar con la sesión del usuario.

**Fix**: sanitizar HTML con DOMPurify/isomorphic-dompurify en allowlist estricta antes de devolverlo o justo antes de renderizar. Eliminar scripts, event handlers, `javascript:` URLs, iframes no permitidos y estilos peligrosos.

**Mitigación**: CSP estricta con `script-src` nonce/hash, `object-src 'none'`, `base-uri 'none'`, y Trusted Types si es viable.

**False positive notes**: Gmail puede aplicar saneamiento propio en su UI, pero aquí se renderiza el payload recibido vía API dentro de otra aplicación; no hay saneamiento visible en el código auditado.

### SEC-008 — Endpoints IA de soporte públicos sin auth ni rate limit

**Severidad**: High  
**Ubicación**:

- `src/lib/supabase/proxy.ts:14-18`
- `src/app/api/soporte/chat/route.ts:23-65`
- `src/app/api/soporte/ayuda-rapida/route.ts:8-45`

**Evidencia**:

```ts
function isPublicPath(pathname: string) {
  ...
  if (pathname.startsWith('/api/')) return true
```

```ts
export async function POST(request: Request) {
  const { mensajes } = (await request.json().catch(() => ({}))) ...
  ...
  const aiRaw = await openrouterChat(chat);
```

No hay `getUser()`, captcha, IP quota ni rate limit antes de llamar a `openrouterChat`.

**Impacto**: cualquier visitante puede consumir cuota/coste del proveedor IA, provocar abuso de tokens y extraer o enumerar la base de conocimiento interna embebida en prompt.

**Fix**: requerir sesión para soporte interno o implementar endpoint público con Turnstile, rate limit por IP, tamaño máximo, cuota diaria y abuso monitoring.

**Mitigación**: desactivar `OPENROUTER_API_KEY` en producción hasta cerrar auth/rate limit si este endpoint no debe ser público.

**False positive notes**: el contenido del soporte puede ser intencionalmente público, pero el consumo de proveedor IA sin control sigue siendo un riesgo de disponibilidad/coste.

### SEC-009 — Secret real de Ágora POS versionado

**Severidad**: Critical  
**Ubicación**:

- `BUSINESS_LOGIC.md:165-168`
- `.claude/PRPs/PRP-ARCH-001-logistica.md:91-94`
- `.claude/memory/project/logistica_spec_completa.md:18-27`
- Historial: `git log --all --oneline -G '[token-redactado]|api-token' ...` devuelve commit `c5db4cc`.

**Evidencia**:

Los archivos versionados contienen la URL base del servidor Ágora y el header `api-token` con un valor real. El valor exacto se ha redactado en este informe por política de secretos.

**Impacto**: cualquier persona con acceso al repo o a su historial puede llamar al servicio Ágora del partner si la red lo permite. Además, el secreto ya está en historial git, por lo que borrar el texto actual no basta.

**Fix**: rotar el token con el partner, moverlo a `AGORA_API_TOKEN`, sustituir documentación por placeholders y purgar o restringir el historial sensible si el repositorio se distribuye.

**Mitigación**: limitar por IP en el servidor Ágora, monitorizar accesos y revisar si se ha usado el token fuera de la infraestructura prevista.

**False positive notes**: el token aparece en documentación, no en código ejecutable, pero sigue siendo un secreto operativo real.

## Recomendaciones No Bloqueantes

1. Añadir baseline global de headers en `next.config.ts` o edge: CSP, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, `frame-ancestors` por defecto.
2. Revisar `/empleo/:path*`: `X-Frame-Options: ALLOWALL` no es un valor estándar efectivo y `frame-ancestors *` permite embedding universal. Si el portal debe ser embebible, usar allowlist por dominios de cliente.
3. Cambiar `src/actions/auth.ts` para no devolver `error.message` crudo de Supabase en login/signup/reset/update; usar mensajes genéricos para no filtrar detalles de auth.
4. Alinear `package.json`: `npm run lint` usa `next lint`, incompatible con el estado actual de Next 16 según la validación dinámica previa. Aunque no es vulnerabilidad directa, bloquea el gate preventivo.
5. Revisar todos los `console.error` que devuelven `err.message` al cliente en endpoints; algunos mensajes exponen detalles internos o de proveedor.
6. Inventariar `createAdminClient()` en server actions y APIs con una regla de CI: service-role solo permitido detrás de guard explícito de auth, rol y tenant.
7. Revisar storage buckets y políticas con datos privados: `cvs-candidatos`, `firmas`, `estudios-apertura-fotos`, `recordings`, `empresa-logos`, `carta-fotos`.
8. Validar en Supabase real el estado final de RLS con consulta a `pg_tables`/`pg_policies`; las migraciones contienen hardenings parciales y políticas históricas abiertas.

## Evidencia de Cobertura

- Rutas API revisadas: 38 route handlers en `src/app/api`.
- Proxy/middleware revisado: `src/lib/supabase/proxy.ts`, `src/proxy.ts`.
- Auth/server actions revisadas: `src/actions/auth.ts`, `src/actions/admin.ts`, actions de `auth`, `empresa`, `rrhh`, `direccion`, `marketing`.
- Service role revisado: `src/lib/supabase/admin.ts` y usos directos de `SUPABASE_SERVICE_ROLE_KEY`.
- Supabase/RLS revisado: `supabase/migrations/*.sql`, con foco en `060_accesos_apps`, `090_fix_rls_always_true_grupo_a`, `20260509081727_estudios_apertura_compartir`, crons, firmas y PSD2.
- Headers revisados: `next.config.ts`.
- Secrets/env revisados: `.gitignore`, `.env.example`, `.env.local.example`, `git grep` de patrones sensibles y `git log -G` para Ágora.

## Veredicto

**FAIL**. No es seguro promover este worktree a producción ni construir encima sin una fase de remediación. Los bloqueantes principales son aislamiento cross-tenant/credenciales, service-role sin controles, crons fail-open, XSS, SSRF y secretos versionados.
