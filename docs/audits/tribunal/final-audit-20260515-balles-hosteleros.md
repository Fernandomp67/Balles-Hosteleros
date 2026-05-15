---
schema_version: "1.0"
id: "final-audit-20260515-balles-hosteleros"
tipo: "final-auditor"
componente: "Balles Hosteleros SaaS"
alcance: "Cierre TRIBUNAL consolidado sobre auditoria funcional, tenant-boundary, seguridad y QA"
status: "reviewed"
created_at: "2026-05-15"
updated_at: "2026-05-15"
agent: "final-auditor"
worktree: "/tmp/balles-hosteleros-audit"
branch: "upstream-feature-aperturas-fotos-ocupacion"
base_head: "32cd5c3"
audit_artifacts_commit: "5558078"
decision: "NO-GO"
---

## Reporte Final Auditor

**Decision**: NO-GO

### Resumen ejecutivo

- Se valida que Balles Hosteleros no es una fachada: `saas-analyst` reconstruyo un SaaS real, amplio y con implementacion sustancial en RRHH, logistica, sala/POS, cocina, direccion, marketing, contabilidad y superficies publicas.
- El repo queda clasificado como `APTO CON REPARACIONES`: sirve para continuar como producto Balles Hosteleros, pero no para produccion inmediata ni como base generica de otro SaaS.
- El cierre de release es `NO-GO` porque `qa-gate` falla, `security-auditor` reporta 9 bloqueantes, y `tenant-boundary-auditor` concluye que el aislamiento SaaS/tenant solo esta parcialmente resuelto.
- El riesgo principal no es falta de producto, sino deuda critica de seguridad, boundaries multiempresa, service role, crons, secretos versionados y gates de build/lint/typecheck rotos.
- No se han implementado fixes en esta fase; este cierre es auditoria y consolidacion TRIBUNAL.

### Resultado por bloque

- QA: Bloqueante. `npx tsc --noEmit` falla en `src/features/rrhh/actions/empleados-actions.ts:60` y `:405`; `npm run lint` falla por `next lint` en Next 16; `npm run build` compila y cae en typecheck. Evidencia: `docs/audits/tribunal/qa-gate-20260515-balles-hosteleros.md`.
- UX: Riesgo. No hubo auditoria UX especifica en esta ronda; `saas-analyst` detecta navegacion funcional amplia, pero tambien raices de departamento tipo placeholder y zonas parciales/mock. No se puede emitir OK de experiencia completa.
- Seguridad: Bloqueante. `security-auditor` reporta `FAIL` con 9 bloqueantes: bypass por email, crons fail-open, RLS abierta en `accesos_apps`, service role sin ownership suficiente, estudios publicos enumerables, SSRF, XSS, IA publica sin rate limit y token real de Agora versionado.
- Performance: Riesgo. No hubo auditoria performance dedicada. El build no llega a cerrar por typecheck y no se ejecutaron pruebas de carga ni medicion de rutas criticas.
- Legal: Riesgo. No hubo auditoria legal/compliance dedicada. Los hallazgos de secretos, credenciales en `accesos_apps`, XSS y boundaries tenant impiden considerar cubiertos privacidad, minimizacion y gobernanza.
- Revision tecnica: Bloqueante. `tenant-boundary-auditor` concluye `Lo contempla parcialmente, pero faltan piezas importantes`; falta `user_empresas` reproducible, hay service role amplio, `/api/*` publico a nivel middleware, RLS abiertas/aparcadas y ausencia de plano plataforma/superadmin auditado.
- Operacion: Bloqueante. El entorno ya es reproducible con Node `20.20.2` y `npm ci`, pero los gates fallan y existen crons que dependen de configuracion critica (`CRON_SECRET`) con patron fail-open.

### Checklist consolidado

- [ ] Bloqueante - Corregir typecheck en `src/features/rrhh/actions/empleados-actions.ts:60` y `:405`. Responsable: maker/ejecutor.
- [ ] Bloqueante - Sustituir `npm run lint` por un gate compatible con Next 16/ESLint 9. Responsable: maker/ejecutor.
- [ ] Bloqueante - Repetir `npx tsc --noEmit`, `npm run lint`, `npm run build` y smoke local tras fixes. Responsable: `qa-gate`.
- [ ] Bloqueante - Eliminar bypass hardcoded por email en `src/proxy.ts` y `src/features/auth/actions/permisos-actions.ts`. Responsable: seguridad/backend.
- [ ] Bloqueante - Hacer que todos los crons con service role fallen cerrado si falta `CRON_SECRET` en produccion. Responsable: seguridad/backend.
- [ ] Bloqueante - Redisenar `accesos_apps`: `empresa_id`, RLS por membership, cifrado/no exposicion de contrasenas y acciones server-side autorizadas. Responsable: seguridad/backend.
- [ ] Bloqueante - Inventariar y blindar todos los usos de `SUPABASE_SERVICE_ROLE_KEY` con auth, rol, tenant y logging. Responsable: seguridad/backend.
- [ ] Bloqueante - Corregir SSRF en `/api/pagina-web/importar-url`. Responsable: seguridad/backend.
- [ ] Bloqueante - Sanitizar HTML de Gmail/firma antes de `dangerouslySetInnerHTML`. Responsable: seguridad/frontend.
- [ ] Bloqueante - Rotar el token real de Agora POS y retirar/redactar secretos versionados. Responsable: operacion/seguridad.
- [ ] Bloqueante - Crear migracion reproducible y RLS para `public.user_empresas` o tabla equivalente. Responsable: backend/datos.
- [ ] Bloqueante - Separar plano plataforma SaaS, roles de plataforma y soporte excepcional auditado. Responsable: arquitectura/backend.
- [ ] Riesgo - Ejecutar auditoria UX/accesibilidad sobre flujos core una vez que la app construya. Responsable: UX/QA.
- [ ] Riesgo - Ejecutar auditoria performance y revisar rutas criticas/bundle una vez que build pase. Responsable: performance/QA.
- [ ] Riesgo - Ejecutar revision legal/compliance de privacidad, cookies, terminos, tratamiento de credenciales y datos tenant. Responsable: legal/producto.

### Bloqueantes

- QA no pasa: typecheck, lint y build estan en `FAIL`.
- Seguridad no pasa: 9 bloqueantes confirmados.
- Aislamiento tenant no pasa como frontera madura: esta parcialmente implementado, pero con huecos importantes.
- Operacion no puede aprobar produccion: secretos/versionado, crons fail-open y service role sin controles suficientes.

### Plan inmediato (24-72h)

1. Ejecutar una fase maker centrada solo en bloqueantes de `qa-gate`: corregir tipos RRHH, reparar lint y repetir gates hasta que `qa-gate` pueda pasar o documentar bloqueos restantes.
2. Ejecutar una fase maker de seguridad priorizada: rotar Agora, quitar bypass por email, cerrar crons, corregir XSS/SSRF y bloquear endpoints IA publicos sin control.
3. Ejecutar una fase maker de tenant-boundary: crear `user_empresas` reproducible, cerrar `accesos_apps`, unificar resolucion de tenant activo y contener `SUPABASE_SERVICE_ROLE_KEY`.
4. Repetir `qa-gate`, `security-auditor` y `tenant-boundary-auditor` tras los fixes antes de considerar cualquier deploy.
5. Mantener `saas-analyst` como guia funcional base, pero no usarlo como aprobacion de release.

## Veredicto TRIBUNAL

`NO-GO`.

El repo es una base real y continuable para Balles Hosteleros, pero queda bloqueado para produccion. El siguiente trabajo no debe ampliar funcionalidad; debe cerrar primero gates de calidad, seguridad y tenant-boundary.
