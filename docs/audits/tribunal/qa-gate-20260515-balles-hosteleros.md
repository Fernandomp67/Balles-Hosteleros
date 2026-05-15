---
schema_version: "1.0"
id: "qa-gate-20260515-balles-hosteleros"
tipo: "qa-gate"
componente: "Balles Hosteleros SaaS"
alcance: "Validacion de gates reproducibles en worktree auditado"
status: "blocked"
created_at: "2026-05-15"
updated_at: "2026-05-15"
agent: "qa-gate"
worktree: "/tmp/balles-hosteleros-audit"
branch: "upstream-feature-aperturas-fotos-ocupacion"
head: "32cd5c3"
node: "20.20.2"
result: "FAIL"
---

# QA Gate - FAIL

**Fecha**: 2026-05-15  
**Proyecto**: Balles Hosteleros SaaS  
**Worktree auditado**: `/tmp/balles-hosteleros-audit`  
**Branch**: `upstream-feature-aperturas-fotos-ocupacion`  
**HEAD**: `32cd5c3`  
**Node requerido usado**: `v20.20.2` via `source ~/.nvm/nvm.sh && nvm use 20.20.2`

## Veredicto

**FAIL**. El gate queda bloqueado por errores de TypeScript y por script de lint no ejecutable en este stack.

No se aprueba merge, deploy ni cierre TRIBUNAL validado.

## Contexto Leido

- `package.json`
- `docs/audits/tribunal/tribunal-audit-20260515-balles-hosteleros.md`
- `docs/002_finalizacion_logistica_y_entrega.md`
- Rutas reales bajo `src/app/`

No se encontraron `PROJECT_BRIEF.md`, `PRODUCTO_TECH_STACK_BLUEPRINT.md`, `EXECUTION_PLAN.md` ni `tasks/*.md` en el worktree auditado. Por tanto, el gate se limita a los scripts reales de `package.json`, la auditoria TRIBUNAL existente y la estructura real de la app.

## Scripts Detectados

```json
{
  "dev": "next dev --turbopack",
  "build": "cross-env NODE_OPTIONS=--max-old-space-size=8192 next build",
  "start": "next start",
  "lint": "next lint"
}
```

No hay script dedicado de typecheck, por lo que el gate usa `npx tsc --noEmit`.

## Resultado de Gates

| Gate | Comando | Resultado |
|---|---|---|
| Typecheck | `npx tsc --noEmit` | FAIL - 2 errores TS2352 |
| Lint | `npm run lint` | FAIL - `next lint` interpreta `lint` como directorio |
| Build | `npm run build` | FAIL - build compila, pero cae en typecheck |
| Smoke dev | No ejecutado | No procede como gate aprobatorio porque typecheck/build fallan |

## Comandos Ejecutados

```bash
source ~/.nvm/nvm.sh && nvm use 20.20.2 && node -v && npm -v
source ~/.nvm/nvm.sh && nvm use 20.20.2 >/dev/null && npx tsc --noEmit
source ~/.nvm/nvm.sh && nvm use 20.20.2 >/dev/null && npm run lint
source ~/.nvm/nvm.sh && nvm use 20.20.2 >/dev/null && npm run build
```

Nota: el primer `npm run build` dentro del sandbox fallo antes de typecheck por descarga bloqueada de Google Fonts. Se repitio fuera del sandbox para distinguir bloqueo de red del resultado real del proyecto. El resultado autoritativo del build es FAIL por typecheck.

## Evidencia

### Typecheck

```text
src/features/rrhh/actions/empleados-actions.ts(60,23): error TS2352: Conversion of type '{ id: any; nombre: any; }[]' to type '{ id: string; nombre: string; }' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.
  Type '{ id: any; nombre: any; }[]' is missing the following properties from type '{ id: string; nombre: string; }': id, nombre
src/features/rrhh/actions/empleados-actions.ts(405,21): error TS2352: Conversion of type '{ id: any; nombre: any; }[]' to type '{ id: string; nombre: string; }' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.
  Type '{ id: any; nombre: any; }[]' is missing the following properties from type '{ id: string; nombre: string; }': id, nombre
```

### Lint

```text
> saas-factory-app@0.1.0 lint
> next lint

Invalid project directory provided, no such directory: /tmp/balles-hosteleros-audit/lint
```

### Build

Resultado relevante del rerun con red disponible:

```text
✓ Compiled successfully in 46s
  Running TypeScript ...
Failed to type check.

./src/features/rrhh/actions/empleados-actions.ts:60:23
Type error: Conversion of type '{ id: any; nombre: any; }[]' to type '{ id: string; nombre: string; }' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.
  Type '{ id: any; nombre: any; }[]' is missing the following properties from type '{ id: string; nombre: string; }': id, nombre

Next.js build worker exited with code: 1 and signal: null
```

Fallo inicial descartado como bloqueo ambiental de red:

```text
next/font: error:
Failed to fetch `Cormorant Garamond` from Google Fonts.

next/font: error:
Failed to fetch `Inter` from Google Fonts.
```

## Accion Requerida

1. Corregir los casteos de `r.empresas` en `src/features/rrhh/actions/empleados-actions.ts:60` y `src/features/rrhh/actions/empleados-actions.ts:405`, porque Supabase esta tipando la relacion como array (`{ id; nombre }[]`) y el codigo la fuerza a objeto (`{ id; nombre }`).
2. Sustituir o reparar el script `lint`, porque `next lint` no es un gate valido en este proyecto Next 16 tal como esta definido.
3. Reejecutar, en este orden, `npx tsc --noEmit`, `npm run lint`, `npm run build`.
4. Ejecutar smoke dev solo despues de que typecheck, lint y build pasen.

**Asignar a**: `maker` / `ejecutor` para correccion.  
**qa-gate** debe repetirse tras los fixes.
