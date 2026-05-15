---
schema_version: "1.0"
id: "tribunal-20260515-balles-hosteleros"
tipo: "architecture"
componente: "Balles Hosteleros SaaS"
alcance: "Auditoria completa del estado real del SaaS externo en worktree reproducible"
max_severity: "critical"
status: "reviewed"
tags:
  - "saas-analysis"
  - "tenant-boundary"
  - "security"
  - "qa"
created_at: "2026-05-15"
updated_at: "2026-05-15"
auditor:
  model: "GPT-5"
  session_id: "codex"
  timestamp: "2026-05-15"
  agent: "capataz -> saas-analyst + tenant-boundary-auditor + security-auditor"
  findings_count: 18
  confidence: "high"
executor:
  model: "GPT-5"
  session_id: "codex"
  timestamp: "2026-05-15"
  agent: "N/A - auditoria sin cambios de codigo"
  commit_ref: "32cd5c3"
  accepted_count: 0
  rejected_count: 0
  partial_count: 0
  emergent_count: 0
  waves_count: 0
  validation:
    required: true
    tools_used:
      - "npm ci"
      - "npm run lint"
      - "npx tsc --noEmit"
      - "npm run build"
      - "npm run dev"
    result: "FAIL"
    notes: "npm ci OK; lint, typecheck y build bloqueados; dev levanta parcialmente."
judge:
  model: "GPT-5"
  session_id: "codex"
  timestamp: "2026-05-15"
  agent: "final-auditor"
  review_batch: "balles-hosteleros-20260515"
  auditor_score: 8
  executor_score: null
  verdict: "NO-GO"
  drift_detected: false
  new_audit_recommended: true
  notes: "Producto real y continuable, pero bloqueado para produccion por QA, seguridad y tenant-boundary."
---

# TRIBUNAL Audit `tribunal-20260515-balles-hosteleros`

> Componente: `Balles Hosteleros SaaS`
> Alcance: `Auditoria completa del estado real del SaaS externo en worktree reproducible`
> Tipo: `architecture`
> Estado: `reviewed`

## Decision de Capataz

**Agente principal**: `saas-analyst`

**Por que**

- La pregunta central es reconstruir que hace realmente el SaaS, separar producto real de fachada y decidir si merece seguir construyendose encima.
- El repo es amplio, tiene datos reales en Supabase y mezcla modulos operativos con zonas parciales.
- La validacion dinamica ya confirmo que el estado reproducible correcto es el branch upstream `feature/aperturas-fotos-ocupacion`, no el `main` del fork.

**Apoyo adicional**

- `tenant-boundary-auditor`: necesario por multiempresa, roles, service role, aislamiento de datos y privacidad operador-tenant.
- `security-auditor`: necesario por APIs, auth, crons/webhooks, secrets, service role y validacion de inputs.
- `qa-gate`: necesario para registrar el estado real de instalacion, lint, tipos, build y smoke.
- `final-auditor`: necesario para cerrar el dictamen consolidado.

**No elegiria**

- `ejecutor`: no hay autorizacion para implementar fixes en esta fase.
- `maker`: queda marcado como no aplicable salvo que despues se aprueben cambios de codigo.

## Base Auditada

- Ruta: `/tmp/balles-hosteleros-audit`
- Repo original: `https://github.com/balleshosteleros/Balles-Hosteleros/`
- Branch local: `upstream-feature-aperturas-fotos-ocupacion`
- HEAD: `32cd5c3`
- Commit de reproducibilidad incluido: `3ec4f4b`
- Node requerido: `20.20.2`
- Instalacion reproducible: `npm ci`

## Validacion Dinamica Preliminar

| Gate | Resultado | Evidencia |
|---|---|---|
| `npm ci` | OK | Instala dependencias desde `package-lock.json` con `.npmrc` `legacy-peer-deps=true`. |
| `npm ls --depth=0` | OK | Dependencias resueltas desde lockfile; no se reproduce el fallo previo de `lucide-react`. |
| `npm run lint` | FAIL | `next lint` no es compatible con este uso en Next 16: interpreta `lint` como directorio. |
| `npx tsc --noEmit` | FAIL | Errores en `src/features/rrhh/actions/empleados-actions.ts:60` y `:405`. |
| `npm run build` | FAIL | Compila, pero cae en typecheck por los mismos errores de RRHH. |
| `npm run dev -- --port 3016` | Parcial | Servidor levanta; `/login` responde; `/` reescribe a `__site` y devuelve 404 sin host de sitio valido. |

## Artefactos Esperados

- `/tmp/balles-hosteleros-audit/docs/legacy/ANALYSIS.md`
- `/tmp/balles-hosteleros-audit/docs/audits/tribunal/tenant-boundary-audit-20260515-balles-hosteleros.md`
- `/tmp/balles-hosteleros-audit/docs/audits/tribunal/security-audit-20260515-balles-hosteleros.md`
- `/tmp/balles-hosteleros-audit/docs/audits/tribunal/qa-gate-20260515-balles-hosteleros.md`
- `/tmp/balles-hosteleros-audit/docs/audits/tribunal/final-audit-20260515-balles-hosteleros.md`

## Estado

- `capataz`: completado
- `saas-analyst`: completado
- `tenant-boundary-auditor`: completado
- `security-auditor`: completado
- `qa-gate`: completado
- `final-auditor`: completado

## Cierre Final Auditor

- Decision: `NO-GO`
- Informe: `docs/audits/tribunal/final-audit-20260515-balles-hosteleros.md`
- Motivo: `qa-gate` falla, `security-auditor` reporta 9 bloqueantes y `tenant-boundary-auditor` concluye que la frontera SaaS/tenant solo esta parcialmente resuelta.
- Dictamen base: `saas-analyst` clasifica el repo como `APTO CON REPARACIONES`: producto real y continuable para Balles Hosteleros, no listo para produccion ni base generica sin remediacion.

## Artefactos Generados

- `docs/legacy/ANALYSIS.md`
- `docs/audits/tribunal/security-audit-20260515-balles-hosteleros.md`
- `docs/audits/tribunal/tenant-boundary-audit-20260515-balles-hosteleros.md`
- `docs/audits/tribunal/qa-gate-20260515-balles-hosteleros.md`
- `docs/audits/tribunal/final-audit-20260515-balles-hosteleros.md`
