# Guia del SaaS

Auditoria TRIBUNAL como `saas-analyst` de La Forja de SaaS.

Fecha de analisis: 2026-05-15  
Repositorio auditado: `https://github.com/balleshosteleros/Balles-Hosteleros/`  
Worktree local auditado: `/tmp/balles-hosteleros-audit`  
Branch auditada: `upstream-feature-aperturas-fotos-ocupacion`  
HEAD auditado: `32cd5c3`  
Factory usada: `/home/fernandomp/dev/la-forja-de-saas`  
Agente: `/home/fernandomp/dev/la-forja-de-saas/agents/saas-analyst.md`  
Skills ejecutadas en orden: `saas-analysis-codebase-scanner`, `saas-analysis-feature-mapper`, `saas-analysis-logic-inference-engine`, `saas-analysis-menu-reconstructor`, `saas-analysis-doc-generator`, `saas-analysis-saas-evaluator`

## Leyenda de evidencia

| Etiqueta | Significado |
| --- | --- |
| `confirmado por codigo` | Hay rutas, acciones, componentes, migraciones o configuracion que prueban el hallazgo en el repositorio. |
| `inferido` | La conclusion se deriva de nombres, wiring, convenciones o composicion, pero no se valido end-to-end. |
| `parcial` | Existe implementacion real, pero incompleta, con mocks, localStorage, dependencias externas no verificadas o cobertura desigual. |
| `no confirmado` | No hay evidencia suficiente en codigo o no se pudo validar dinamicamente en esta pasada. |

## Resumen ejecutivo

Balles Hosteleros es un SaaS operacional para grupos de hosteleria con foco en multiempresa, departamentos internos, RRHH, logistica, POS/sala, cocina, aperturas, marketing, contabilidad, gestoria, juridico, comunicacion y portal publico.

El producto no es una simple maqueta. Hay una base grande de Next.js App Router, server actions, integraciones Supabase, migraciones SQL, modulos con CRUD real, flujos publicos y crons. Tambien hay zonas de demo, mocks y localStorage que conviven con modulos ya persistidos.

Veredicto operativo: el repo es recuperable y sirve para continuar el producto Balles Hosteleros, pero no esta listo para produccion ni para usarse como base generica de otro SaaS sin reparaciones. La razon principal es que el typecheck/build esta roto, hay riesgos de seguridad/multi-tenant confirmados por codigo y existen inconsistencias entre documentacion, wiring real y limites de entorno.

## App detectada

| Aspecto | Estado | Evidencia |
| --- | --- | --- |
| Tipo de app | `confirmado por codigo` | SaaS B2B interno/publico para hosteleria, con App Router en `src/app`, features en `src/features` y backend Supabase. |
| Dominio funcional | `confirmado por codigo` | `BUSINESS_LOGIC.md`, menu lateral, rutas `(main)` y features cubren operaciones de restaurantes/grupos. |
| Multiempresa | `parcial` | Existen `empresas`, `user_empresas`, `empresa_roles`, selector de empresa y cookie `bh_empresa_activa`, pero muchos modulos siguen usando `profiles.empresa_id`. |
| Publicacion externa | `confirmado por codigo` | Public site host-based en `__site`, carta digital en `/carta/[slug]`, empleo en `/empleo`, firmas en `/firmar/[token]`, estudios publicos en `/p/[slug]`. |
| Estado tecnico | `parcial` | `npm ci` funciona y `npm run dev` levanta, pero `npx tsc --noEmit` y `npm run build` fallan por tipos en RRHH. |

## Stack confirmado

| Capa | Estado | Evidencia |
| --- | --- | --- |
| Framework | `confirmado por codigo` | `package.json` usa Next `^16.0.0`; scripts usan `next dev --turbopack`, `next build`, `next start`. |
| Runtime UI | `confirmado por codigo` | React `^19.0.0`, componentes Radix/shadcn, Tailwind. |
| Lenguaje | `confirmado por codigo` | TypeScript con `strict: true` en `tsconfig.json`; `allowJs: true`. |
| Backend | `confirmado por codigo` | Supabase SSR/browser/admin clients, migraciones en `supabase/migrations`, RLS, storage buckets y RPCs. |
| Integraciones | `parcial` | Agora, GoCardless/PSD2, Resend, WhatsApp Cloud, Meta Ads, Google OAuth/Workspace, R2 y OpenRouter aparecen en codigo/env. No se validaron credenciales vivas. |
| Testing | `parcial` | Hay Playwright/config y scripts, pero no se ejecuto suite funcional completa; el build falla antes de un cierre QA estable. |

## Validacion dinamica recibida

| Prueba | Estado | Resultado |
| --- | --- | --- |
| Node | `confirmado por codigo` | `.nvmrc` define `20.20.2`. |
| Instalacion | `confirmado por codigo` | `.npmrc` contiene `legacy-peer-deps=true`, `package-lock.json` existe, `npm ci` funciona segun contexto validado. |
| Lint | `confirmado por codigo` | `npm run lint` falla porque el script invoca `next lint`, retirado/incompatible con Next 16. |
| Typecheck | `confirmado por codigo` | `npx tsc --noEmit` falla en `src/features/rrhh/actions/empleados-actions.ts:60` y `:405`. |
| Build | `confirmado por codigo` | `npm run build` compila hasta caer por los mismos errores de tipos. |
| Dev server | `confirmado por codigo` | `npm run dev -- --port 3016` levanta; `/login` responde; `/` reescribe a `__site` y devuelve 404 si el host no resuelve a pagina publicada. |

## Arquitectura de alto nivel

| Area | Estado | Evidencia |
| --- | --- | --- |
| App Router | `confirmado por codigo` | Rutas en `src/app/(auth)`, `src/app/(main)`, `src/app/api`, `src/app/__site`, `src/app/carta`, `src/app/empleo`, `src/app/firmar`, `src/app/p`. |
| Feature-first | `confirmado por codigo` | `src/features/<modulo>` agrupa actions, components, contexts, services y stores por dominio. |
| Providers globales | `confirmado por codigo` | `src/app/layout.tsx` monta `AuthProvider`, `EmpresaProvider`, `AyudaProvider`, `MarketingProvider`, `ViewModeProvider` y React Query. |
| Proxy/auth | `confirmado por codigo` | `src/proxy.ts` y `src/lib/supabase/proxy.ts` gestionan sesion, host rewrite y enforcement de modulos. |
| Datos | `confirmado por codigo` | Mas de 100 migraciones, tablas multiempresa, policies RLS, funciones SQL, buckets y vistas. |
| Cron jobs | `confirmado por codigo` | `vercel.json` agenda sincronizacion Agora, fichajes huerfanos, Toques, PSD2 y firmas. |

## Modelo de navegacion reconstruido

La navegacion principal se reconstruye desde `src/features/layout/components/app-sidebar.tsx`, rutas de `src/app/(main)` y permisos de `src/features/auth`.

| Seccion | Estado | Rutas principales | Lectura funcional |
| --- | --- | --- | --- |
| Mi Panel | `confirmado por codigo` | `/mi-panel`, `/mi-panel/perfil`, `/mi-panel/points`, `/mi-panel/fichajes`, `/mi-panel/solicitudes`, `/mi-panel/comunicados`, `/mi-panel/documentos` | Panel personal del empleado con datos, fichajes, solicitudes, formacion, comunicados y puntos. |
| Mis Departamentos | `confirmado por codigo` | `/mis-departamentos` | Hub filtrado por permisos/roles que muestra departamentos accesibles. |
| Direccion | `confirmado por codigo` | `/direccion/estructura`, `/direccion/cronogramas`, `/direccion/documentacion`, `/direccion/aperturas`, `/direccion/presentaciones` | Gestion directiva, documentos, estudios de apertura y presentaciones. |
| Sala | `confirmado por codigo` | `/sala/pos`, `/sala/ventas`, `/sala/reservas`, `/sala/clientes` | POS, tickets, ventas, reservas y clientes. |
| Cocina | `confirmado por codigo` | `/cocina/comandas`, `/cocina/nuevas-recetas`, `/cocina/escandallos`, `/cocina/elaboraciones`, `/cocina/partidas`, `/cocina/temperaturas` | Comandas, receta oficial, escandallos, partidas y control operativo. |
| Logistica | `confirmado por codigo` | `/logistica`, `/logistica/proveedores`, `/logistica/productos`, `/logistica/pedidos`, `/logistica/stock`, `/logistica/inventarios`, `/logistica/tarifas` | Compras, proveedores, productos, stock, inventarios y sincronizacion Agora. |
| Gerencia | `confirmado por codigo` | `/gerencia/mantenimiento`, `/gerencia/vencimientos`, `/gerencia/cierres`, `/gerencia/descuentos`, `/gerencia/informes`, `/gerencia/ratios`, `/gerencia/comunicados`, `/gerencia/encuestas` | Seguimiento de operacion, mantenimiento, cierres, descuentos y comunicacion. |
| RRHH | `confirmado por codigo` | `/rrhh/empleados`, `/rrhh/fichajes`, `/rrhh/solicitudes`, `/rrhh/firmas`, `/rrhh/calendarios`, `/rrhh/horarios`, `/rrhh/reclutamiento`, `/rrhh/boarding`, `/rrhh/bonus`, `/rrhh/points`, `/rrhh/pagos`, `/rrhh/formacion`, `/rrhh/encuestas`, `/rrhh/cuestionarios` | Gestion de empleados, fichajes, documentos, firmas, seleccion y onboarding. |
| Marketing | `parcial` | `/marketing/calendario`, `/marketing/contenido`, `/marketing/campanas`, `/marketing/carta-digital`, `/marketing/pagina-web`, `/marketing/fidelizacion`, `/marketing/captacion` | Campanas, carta digital, web publica y leads; otras areas tienen mocks/localStorage. |
| Contabilidad | `confirmado por codigo` | `/contabilidad/contactos`, `/contabilidad/facturas`, `/contabilidad/transacciones`, `/contabilidad/conciliacion`, `/contabilidad/bancos`, `/contabilidad/reglas` | Facturas, transacciones, contactos y PSD2/conciliacion. |
| Gestoria | `confirmado por codigo` | `/gestoria/modelos`, `/gestoria/presentaciones`, `/gestoria/contrataciones` | AEAT/modelos, presentaciones y contrataciones. |
| Juridico | `confirmado por codigo` | `/juridico/procesos` | Procesos y documentos juridicos. |
| Calidad | `parcial` | `/calidad`, `/calidad/auditorias`, `/calidad/clientes`, `/calidad/empleados`, `/calidad/inspecciones` | Rutas y UI existen, pero no se detectaron actions/tablas propias en el escaneo. |
| Agenda/Reuniones/Comunicacion | `confirmado por codigo` | `/agenda`, `/reuniones`, `/comunicacion` | Contactos, reuniones, canales y mensajes. |
| Ajustes | `confirmado por codigo` | `/ajustes` | Empresas, locales, departamentos, roles, email config y reglas de submodulos. |

Las paginas raiz de algunos departamentos (`/direccion`, `/sala`, `/cocina`, `/rrhh`, `/marketing`, `/contabilidad`, `/gestoria`, `/juridico`) son mayoritariamente placeholders o contenedores, mientras que muchas subrutas si tienen implementacion real. Esto debe tenerse en cuenta para demos: entrar por submodulos, no por raiz de departamento.

## Auth, permisos y multi-tenant

| Hallazgo | Estado | Evidencia |
| --- | --- | --- |
| Login principal | `confirmado por codigo` | `src/app/(auth)/page.tsx` y `LoginForm` soportan email/password y Google. `/login` redirige a `/`. |
| Redireccion por rol | `confirmado por codigo` | `role-redirect.ts` redirige roles hacia `/mis-departamentos`. |
| Permisos cliente | `confirmado por codigo` | `AuthProvider` carga `profiles`, `user_roles` y `empresa_roles`; `puedeVer` y `puedeEditar` aplican permisos y bypass para admin/director. |
| Cache local de permisos | `confirmado por codigo` | `AuthProvider` usa claves `bh_auth_cache_*` en localStorage. |
| Enforcement server/proxy | `parcial` | `src/proxy.ts` valida modulos con service role; si falta `SUPABASE_SERVICE_ROLE_KEY`, el enforcement se salta con warning. |
| Bypass hardcoded | `confirmado por codigo` | Email `ricardosilva211@gmail.com` tiene bypass en `src/proxy.ts` y `src/features/auth/actions/permisos-actions.ts`. |
| Multiempresa seleccionable | `parcial` | `EmpresaProvider`, `empresa-activa-actions.ts` y cookie `bh_empresa_activa` existen, pero varios dominios usan solo `profiles.empresa_id`. |
| RLS | `parcial` | Hay migraciones de hardening, pero `090_fix_rls_always_true_grupo_a.sql` deja tablas aparcadas o pendientes. |

Lectura del riesgo multi-tenant: el modelo existe y esta avanzado, pero no es uniforme. Algunos modulos usan `getEmpresaActivaForUser`, otros `getAppContext` basado en `profiles.empresa_id`, y otros actions usan admin/service role. Antes de produccion hay que revisar tenant boundary modulo por modulo.

## Modulos funcionales

### Direccion y aperturas

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Estudios de apertura | `confirmado por codigo` | `src/features/direccion/actions/estudios-apertura-actions.ts` gestiona `estudios_apertura`, fotos, signed URLs y estado compartido. |
| Ocupacion | `confirmado por codigo` | `OcupacionTab.tsx` modela escenarios, franjas, personas, ticket medio y KPIs. |
| Local y fotos | `confirmado por codigo` | `LocalTab.tsx` gestiona datos del local, geolocalizacion y fotos por categorias. |
| Gastronomia | `confirmado por codigo` | `GastronomiaTab.tsx` cubre concepto, servicio, precio, categorias, platos y fotos. |
| Link publico | `confirmado por codigo` | `/p/[slug]`, `fetchEstudioPorSlug` y `EstudioPublicoView` exponen estudio publico con `noindex`. |
| Presentaciones | `confirmado por codigo` | `presentaciones`, `presentacion_slides` y branding de empresa aparecen en actions y rutas. |

### Logistica

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Dashboard logistico | `confirmado por codigo` | `/logistica/page.tsx` carga pedidos, proveedores, stock, productos e inventarios. |
| Proveedores/productos | `confirmado por codigo` | Actions y tablas para proveedores, productos, categorias, ingredientes y tarifas. |
| Pedidos/albaranes | `confirmado por codigo` | `pedidos-actions.ts` crea pedidos y lineas; `PedidosView.tsx` confirma albaranes. |
| Stock | `confirmado por codigo` | `stock-actions.ts` lista, actualiza y suma stock desde albaran. |
| Inventarios | `confirmado por codigo` | Actions y tablas `inventarios` y `lineas_inventario`. |
| Agora | `confirmado por codigo` | `agora-actions.ts`, `agora-ventas-sync.ts` y cron `/api/cron/agora-sync`. |
| Escandallos | `confirmado por codigo` | `escandallos-actions.ts`, `EscandalloEditor.tsx` y RPC `coste_escandallo`. |
| Necesidad de compra | `parcial` | RPC `calcular_necesidad_compra` y action `getNecesidadCompra` existen, pero no se confirmo UI principal que la use. |

Contradiccion documental: `BUSINESS_LOGIC.md` indica que `handleConfirmarAlbaran` no suma stock y que `coste_escandallo` no se muestra en UI. El codigo actual contradice parcialmente esa memoria: `PedidosView.tsx` si llama `sumarStockDesdeAlbaran` y `EscandalloEditor.tsx` si llama `getCosteEscandallo`.

### Sala y POS

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| POS | `confirmado por codigo` | `tickets-actions.ts` crea tickets, lineas, pagos, cierres y anulaciones. |
| Mesas/zonas | `confirmado por codigo` | Tablas y actions de mesas, zonas y sesiones de caja. |
| Descuento de stock | `confirmado por codigo` | `descontar-stock-por-ventas.ts` descuenta ingredientes por escandallo o producto compra si aplica. |
| Comandas hacia cocina | `confirmado por codigo` | `enviarACocina` y estados de lineas/tickets conectan POS con cocina. |
| Reservas/clientes | `confirmado por codigo` | Actions/tablas de `reservas` y `clientes_sala`. |

### Cocina

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Comandas | `confirmado por codigo` | `comandas-actions.ts` valida transiciones y estados de lineas enviadas a cocina. |
| Nuevas recetas | `confirmado por codigo` | Pipeline con fases, ingredientes, historial, tareas y publicacion oficial. |
| Escandallos | `confirmado por codigo` | Composiciones, ingredientes, coste y productos de venta/compra. |
| Elaboraciones/partidas/temperaturas | `confirmado por codigo` | Features, tablas y UI operativas detectadas. |
| Inconsistencia de tipos producto | `parcial` | Se observo uso de variantes como `Compra`/`compra`; requiere confirmar enum/normalizacion antes de tocar logica. |

### RRHH

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Empleados | `confirmado por codigo` | `empleados-actions.ts` crea auth user, profile, rol, relacion empresa y registro empleado. |
| Fichajes | `confirmado por codigo` | `fichajes-actions.ts` soporta entrada/salida, geolocalizacion, fichaje manual y cierre de huerfanos. |
| Firmas | `confirmado por codigo` | `firmas-actions.ts` gestiona documentos, tokens, OTP, hashes, PDF, auditoria y email. |
| Reclutamiento publico | `confirmado por codigo` | `/empleo` y `/api/empleo/candidatura` validan candidaturas, PDF, rate limit y Turnstile opcional. |
| Boarding/offboarding | `confirmado por codigo` | Plantillas, procesos y documentos en `primer-acceso` y `boarding-actions.ts`. |
| Accesos apps | `confirmado por codigo` | `accesos-apps-actions.ts` gestiona credenciales de apps externas. |
| Type safety | `confirmado por codigo` | `empleados-actions.ts` rompe typecheck en lineas 60 y 405. |

Riesgo critico: `accesos-apps-actions.ts` usa service role y no muestra guardia explicita de rol dentro de la action para listar/crear/actualizar/borrar accesos. Dado que el dominio puede incluir usuario y contrasena de apps externas, esto exige revision inmediata de exposicion y llamada desde cliente.

### Marketing, carta digital y pagina publica

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Campanas | `confirmado por codigo` | CRUD de `campanas_marketing` e integraciones Resend, WhatsApp Cloud y Meta Ads. |
| Carta digital | `confirmado por codigo` | Admin de categorias/items/fotos, publicacion por slug y likes publicos con rate limit local. |
| Pagina web | `confirmado por codigo` | `paginas_web`, dominios, leads, host resolver y render en `__site`. |
| Leads | `confirmado por codigo` | `/api/pagina-web/leads` valida payload y aplica rate limit por IP. |
| Fidelizacion/captacion/contenido | `parcial` | Hay rutas y UI, pero el modulo mantiene mocks/localStorage en varias areas. |
| Host default | `confirmado por codigo` | Sin host valido, `/` reescribe a `__site` y puede devolver 404. Esto coincide con validacion dinamica. |

### Contabilidad

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Contactos/facturas/transacciones | `confirmado por codigo` | `contabilidad-actions.ts` cubre CRUD basico. |
| Bancos PSD2 | `confirmado por codigo` | `psd2-actions.ts`, GoCardless provider y token crypto AES-GCM. |
| Sync bancario | `confirmado por codigo` | `/api/cron/psd2-sync` sincroniza cuentas activas. |
| Reglas/conciliacion | `parcial` | Rutas y UI existen, pero no se valido motor completo de conciliacion. |

### Gerencia

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Mantenimiento/vencimientos/cierres | `confirmado por codigo` | Actions y tablas de mantenimiento, vencimientos, cierres y config. |
| Comunicados/encuestas/informes | `confirmado por codigo` | Actions y tablas dedicadas en feature `gerencia`. |
| Ratios | `parcial` | Rutas y UI detectadas; no se valido formula completa ni datos reales. |

### Gestoria y juridico

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Gestoria modelos | `confirmado por codigo` | `modelos_aeat`, reglas de categorizacion, asignaciones y facturas. |
| Gestoria contrataciones | `confirmado por codigo` | Contrataciones, puestos y empleados. |
| Juridico procesos | `confirmado por codigo` | `procesos_juridicos` y `documentos_juridicos`. |
| Automatizacion legal/fiscal completa | `no confirmado` | No se valido validez legal/fiscal ni integracion externa oficial. |

### Calidad, formacion, Google Workspace y soporte

| Capacidad | Estado | Evidencia |
| --- | --- | --- |
| Calidad | `parcial` | Rutas/componentes existen; el escaneo no encontro actions/tablas propias. |
| Formacion | `parcial` | UI y stores existen, con localStorage y seeds; no hay actions principales detectadas. |
| Google Workspace | `parcial` | Botones/drawers y rutas API existen, pero hay mocks/localStorage y no se valido OAuth vivo. |
| Soporte | `parcial` | Chat de soporte con OpenRouter y FAQs por rol; no se valido calidad de respuestas ni permisos finos. |
| Toques/points | `confirmado por codigo` | Actions, reglas, recompensas, balances, canjes y crons de devengo/snapshot. |

## Rutas publicas y APIs relevantes

| Ruta | Estado | Funcion |
| --- | --- | --- |
| `/` | `confirmado por codigo` | Login o rewrite host-based hacia `__site` segun dominio/sesion. |
| `/carta/[slug]` | `confirmado por codigo` | Carta digital publica. |
| `/p/[slug]` | `confirmado por codigo` | Estudio de apertura publico. |
| `/empleo/[slug]` y `/empleo/[slug]/[oferta]` | `confirmado por codigo` | Portal publico de empleo y ofertas. |
| `/firmar/[token]` | `confirmado por codigo` | Firma publica de documentos. |
| `/__site/[[...slug]]` | `confirmado por codigo` | Render de pagina publica por host. |
| `/api/cron/*` | `confirmado por codigo` | Tareas programadas de Agora, fichajes, firmas, PSD2 y Toques. |
| `/api/empleo/candidatura` | `confirmado por codigo` | Entrada publica de candidaturas. |
| `/api/pagina-web/leads` | `confirmado por codigo` | Captura publica de leads. |

## Persistencia y migraciones

| Area | Estado | Evidencia |
| --- | --- | --- |
| Migraciones | `confirmado por codigo` | Carpeta `supabase/migrations` con modelo amplio y hardening progresivo. |
| RLS hardening | `parcial` | Migraciones `090`, `091`, `092`, `094` corrigen policies, storage y funciones, pero no cierran todas las tablas. |
| Storage | `confirmado por codigo` | Buckets para documentos, carta, estudios, empleados, logos y grabaciones. |
| RPCs | `confirmado por codigo` | `coste_escandallo`, `calcular_necesidad_compra`, POS ticket numbering y funciones de cierre. |
| Tipos Supabase | `parcial` | Hay tipos locales, pero el typecheck actual revela al menos dos inferencias incompatibles. |

---

# Evaluacion del SaaS

## Estado por dimensiones

| Dimension | Estado | Evidencia | Riesgo |
| --- | --- | --- | --- |
| Producto | `parcial` | Producto amplio y coherente para hosteleria; muchas capacidades reales. | Alcance excesivo, zonas demo y documentacion parcialmente desactualizada. |
| UI | `parcial` | Sidebar y submodulos extensos con layouts reales. | Raices de departamentos vacias, UX desigual y flows no validados end-to-end. |
| Persistencia | `confirmado por codigo` | Supabase con tablas, RPCs, storage y migraciones. | RLS/tenant boundary no uniforme; uso de service role en dominios sensibles. |
| Auth | `parcial` | Login, Supabase Auth, roles, permisos y proxy. | Bypass hardcoded, enforcement saltable si falta service key y cache local de permisos. |
| Seguridad | `parcial` | Hay hardening de policies, storage y funciones. | Crons permiten acceso si falta `CRON_SECRET`, APIs publicas dependen de guards locales y hay secretos en documentacion. |
| Multi-tenant | `parcial` | Modelo multiempresa, selector, roles por empresa y cookie activa. | Uso mixto de `profiles.empresa_id`, `bh_empresa_activa`, `user_empresas` y admin client. |
| Pagos | `no confirmado` | No se detecto billing SaaS productivo; si hay PSD2 bancario. | No hay evidencia de Stripe/subscripciones como monetizacion SaaS. |
| Codigo | `parcial` | Arquitectura feature-first grande y mantenible en varias zonas. | Typecheck/build roto, script lint obsoleto por Next 16 y duplicidades de contexto. |
| Testing | `parcial` | Config/herramientas presentes. | No se confirmo suite verde; build falla antes de cierre estable. |
| Evolucion futura | `parcial` | ROADMAP y BUSINESS_LOGIC muestran direccion de producto. | Antes de crecer hay que estabilizar build, seguridad, tenant boundary y separar mocks de produccion. |

## Contradicciones detectadas

| Contradiccion | Estado | Impacto |
| --- | --- | --- |
| `BUSINESS_LOGIC.md` dice que confirmar albaran no suma stock, pero `PedidosView.tsx` llama `sumarStockDesdeAlbaran`. | `confirmado por codigo` | La memoria funcional esta desactualizada y puede inducir fixes erroneos. |
| `BUSINESS_LOGIC.md` dice que `coste_escandallo` no se muestra en UI, pero `EscandalloEditor.tsx` llama `getCosteEscandallo`. | `confirmado por codigo` | La evaluacion de deuda de escandallos debe actualizarse. |
| La app se presenta como multiempresa, pero parte del backend usa empresa activa y otra parte `profiles.empresa_id`. | `confirmado por codigo` | Riesgo de datos cruzados o comportamiento distinto segun modulo. |
| Hay hardening de seguridad, pero crons quedan abiertos si `CRON_SECRET` no esta definido. | `confirmado por codigo` | El entorno puede pasar de seguro a inseguro por variable ausente. |
| `/api/` se trata como publico en proxy, mientras muchas APIs/actions dependen de guards internos. | `confirmado por codigo` | La seguridad real queda distribuida y dificil de auditar. |
| README mantiene tono de plantilla SaaS Factory, mientras el producto real ya es Balles Hosteleros. | `confirmado por codigo` | Riesgo de onboarding tecnico confuso. |

## Riesgos principales

🔴 CRITICO - esfuerzo: medio - impacto: alto - accion: reparar `src/features/rrhh/actions/empleados-actions.ts` y dejar `npx tsc --noEmit` y `npm run build` verdes antes de cualquier despliegue.

🔴 CRITICO - esfuerzo: bajo - impacto: alto - accion: hacer `CRON_SECRET` obligatorio en produccion y fallar cerrado en `/api/cron/agora-sync`, `/api/cron/cerrar-fichajes-huerfanos`, `/api/cron/firmas-expirar` y `/api/cron/psd2-sync`.

🔴 CRITICO - esfuerzo: medio - impacto: alto - accion: auditar server actions que usan service role, empezando por `src/features/rrhh/actions/accesos-apps-actions.ts`, y exigir auth/rol/empresa dentro de cada action sensible.

🔴 CRITICO - esfuerzo: bajo - impacto: alto - accion: eliminar bypass hardcoded de `ricardosilva211@gmail.com` o moverlo a mecanismo temporal controlado por entorno no productivo.

🟠 ALTO - esfuerzo: medio - impacto: alto - accion: normalizar tenant context para que todos los modulos usen una unica fuente validada de empresa activa con comprobacion de pertenencia.

🟠 ALTO - esfuerzo: bajo - impacto: medio - accion: corregir `npm run lint` para Next 16 y definir una puerta QA reproducible.

🟠 ALTO - esfuerzo: bajo - impacto: medio - accion: retirar secretos o tokens de documentacion versionada, empezando por datos de Agora en `BUSINESS_LOGIC.md`.

🟠 ALTO - esfuerzo: medio - impacto: medio - accion: separar explicitamente modulos productivos de modulos demo/mock/localStorage antes de presentaciones comerciales.

🟡 MEDIO - esfuerzo: medio - impacto: medio - accion: revisar CSP/frame policy de `/empleo` porque `frame-ancestors *` y `X-Frame-Options: ALLOWALL` abren el portal a embedding amplio.

🟡 MEDIO - esfuerzo: bajo - impacto: medio - accion: actualizar `BUSINESS_LOGIC.md` con el estado real de stock, escandallos y funciones SQL para evitar decisiones basadas en deuda ya resuelta.

## Decision sobre uso del repo

| Uso | Decision | Condicion |
| --- | --- | --- |
| Seguir desarrollando | Si | Solo despues de reparar typecheck/build y cerrar riesgos criticos de seguridad/tenant. |
| Ensenar a cliente | Con restricciones | Demo guiada de submodulos maduros; no entrar por raices vacias ni prometer produccion inmediata. |
| Usar como demo | Si | Entorno controlado, datos ficticios, credenciales rotadas y rutas publicas preparadas. |
| Llevar a produccion | No | El build falla, hay crons fail-open, bypass hardcoded y boundaries multiempresa no uniformes. |
| Usar como base de otro SaaS | No | El codigo esta altamente acoplado a hosteleria y mezcla producto real con demo/mocks. |

## Veredicto final del repo

APTO CON REPARACIONES

El repo debe conservarse como base de continuidad de Balles Hosteleros porque tiene mucha implementacion real y conocimiento de dominio codificado. No debe considerarse listo para produccion ni como plantilla generica. La ruta correcta es estabilizar el core tecnico, cerrar seguridad, normalizar multiempresa, actualizar memoria funcional y despues priorizar modulos maduros.

## Orden recomendado de saneamiento

1. Reparar typecheck/build y lint.
2. Cerrar seguridad fail-open: crons, bypass, service-role actions, secretos documentados.
3. Auditar tenant boundary por modulo.
4. Marcar cada modulo como productivo, parcial o demo.
5. Actualizar README/BUSINESS_LOGIC/ROADMAP con el estado real.
6. Crear smoke tests para login, permisos, empresa activa, POS, fichajes, aperturas, carta y pagina publica.

---

# Evidencia y limites del analisis

## Confirmado por codigo

| Evidencia | Rutas |
| --- | --- |
| App Next 16/React 19/TypeScript/Supabase con App Router. | `package.json`, `src/app`, `src/features`, `supabase/migrations` |
| Menu lateral reconstruible con departamentos y submodulos. | `src/features/layout/components/app-sidebar.tsx` |
| Login, roles, permisos y proxy de modulos. | `src/app/(auth)`, `src/features/auth`, `src/proxy.ts`, `src/lib/supabase/proxy.ts` |
| Multiempresa con selector/cookie y relaciones usuario-empresa. | `src/features/empresa`, `src/lib/app-context.ts`, `src/lib/empresa-server.ts` |
| Logistica, pedidos, stock, escandallos y Agora tienen implementacion real. | `src/features/logistica`, `src/app/api/cron/agora-sync` |
| POS/Sala descuenta stock y se conecta con cocina/comandas. | `src/features/sala/pos`, `src/features/cocina` |
| RRHH incluye empleados, fichajes, firmas, empleo publico y boarding. | `src/features/rrhh`, `src/app/empleo`, `src/app/firmar` |
| Direccion/aperturas incluye estudios, fotos, ocupacion y link publico. | `src/features/direccion`, `src/app/p/[slug]` |
| Marketing incluye campanas, carta digital, pagina publica y leads. | `src/features/marketing`, `src/app/carta`, `src/app/__site`, `src/app/api/pagina-web/leads` |
| Contabilidad incluye CRUD financiero y PSD2/GoCardless. | `src/features/contabilidad`, `src/app/api/cron/psd2-sync` |
| Build/typecheck fallan por RRHH. | `src/features/rrhh/actions/empleados-actions.ts:60`, `src/features/rrhh/actions/empleados-actions.ts:405` |
| Lint falla por `next lint` en Next 16. | `package.json` |
| Crons dependen de `CRON_SECRET` opcional en varias rutas. | `src/app/api/cron/*` |
| Bypass hardcoded por email. | `src/proxy.ts`, `src/features/auth/actions/permisos-actions.ts` |
| Portal empleo permite embedding amplio. | `next.config.ts` |

## Inferido con confianza media

| Inferencia | Motivo |
| --- | --- |
| La app esta pensada para operar grupos de restauracion con varias empresas/locales. | Naming, modelos, menu, `BUSINESS_LOGIC.md`, features y tablas convergen en ese dominio. |
| La parte mas madura esta en RRHH, logistica, POS/cocina, direccion/aperturas, carta y contabilidad. | Hay actions, tablas, UI y flujos complejos frente a otros modulos con stores/mocks. |
| La ruta de produccion prevista incluye Vercel + Supabase + servicios externos. | `vercel.json`, env examples, Supabase SSR y clients de integraciones. |
| La deuda principal no es falta de producto, sino estabilizacion y seguridad. | El producto tiene mucha superficie funcional, pero falla build y hay boundaries inconsistentes. |

## Parcial

| Area | Motivo |
| --- | --- |
| Multi-tenant | El modelo existe, pero no se aplica igual en todos los modulos. |
| Seguridad | Hay hardening real, pero tambien fail-open y service role sin guardia visible en actions sensibles. |
| Formacion/Calidad/Google Workspace | Existen UI/rutas, pero con mocks/localStorage o sin actions detectadas. |
| Marketing completo | Campanas/carta/web son reales; fidelizacion/captacion/contenido parecen menos maduros. |
| Testing | Hay herramientas, pero no hay prueba de suite verde y el build no pasa. |

## No confirmado o ambiguo

| Punto | Motivo |
| --- | --- |
| Validez funcional con base Supabase real. | No se usaron credenciales vivas ni se hicieron flujos completos contra datos reales. |
| Seguridad explotable end-to-end de cada action. | Se detectaron patrones de riesgo por codigo, pero no se hizo explotacion. |
| Correctitud fiscal/legal de gestoria/juridico. | El analisis fue tecnico, no legal ni fiscal. |
| Integraciones externas en produccion. | No se validaron Agora, GoCardless, Resend, WhatsApp, Meta, Google, R2 ni OpenRouter con credenciales reales. |
| Estado del repo original remoto. | El worktree local tiene origin `https://github.com/Fernandomp67/Balles-Hosteleros.git`; el contexto declara original `https://github.com/balleshosteleros/Balles-Hosteleros/`. |
| Informe previo dinamico. | `/tmp/Balles-Hosteleros-auditoria-dinamica-2026-05-14.md` no existia en esta maquina durante la auditoria. |

## Limites operativos

No se modifico codigo. No se ejecutaron fixes. No se revirtieron cambios ajenos. El analisis se baso en lectura estatica, evidencias ya validadas dinamicamente por el contexto y comprobaciones puntuales de busqueda. El directorio tenia cambios no relacionados (`docs/audits/`) antes de este entregable, que no fueron revertidos ni absorbidos.

---

# Handoffs entre fases

## Fase 1 - Codebase scanner

## HANDOFF PARA SIGUIENTE FASE

### App detectada

Balles Hosteleros, SaaS operacional de hosteleria multiempresa con app interna y superficies publicas.

### Tipo de repo y stack confirmado

Next.js 16, React 19, TypeScript, Supabase, Tailwind, App Router, feature-first modules, Vercel crons.

### Evidencias fuertes

`package.json`, `src/app`, `src/features`, `supabase/migrations`, `vercel.json`, `.env.example`, `BUSINESS_LOGIC.md`.

### Hallazgos confirmados de la fase

Repo con implementacion real amplia, package lock presente, `.nvmrc` 20.20.2, `.npmrc` con `legacy-peer-deps=true`, build/typecheck roto por RRHH.

### Hallazgos parciales o dudosos

Varios modulos mezclan persistencia real con mocks/localStorage. Testing y entorno externo no quedan confirmados.

### Contradicciones detectadas

README conserva rasgos de plantilla generica mientras el producto real ya esta especializado en Balles Hosteleros.

### Riesgos detectados

Build roto, lint obsoleto por Next 16, secretos en documentacion, superficie publica/API amplia.

### Archivos o rutas criticas

`src/features/rrhh/actions/empleados-actions.ts`, `src/proxy.ts`, `src/lib/supabase/proxy.ts`, `supabase/migrations`, `BUSINESS_LOGIC.md`.

### Incognitas para la siguiente fase

Mapa exacto de features maduras frente a parciales y demo.

## Fase 2 - Feature mapper

## HANDOFF PARA SIGUIENTE FASE

### App detectada

SaaS modular por departamentos: Mi Panel, Direccion, Sala, Cocina, Logistica, Gerencia, RRHH, Marketing, Contabilidad, Gestoria, Juridico, Calidad y Ajustes.

### Tipo de repo y stack confirmado

Feature-first con server actions, components, services, contexts y Supabase por dominio.

### Evidencias fuertes

`src/features/*`, rutas en `src/app/(main)`, `app-sidebar.tsx`, migraciones por dominio.

### Hallazgos confirmados de la fase

RRHH, Logistica, Sala/POS, Cocina, Direccion/Aperturas, Marketing/Carta/Web, Contabilidad/PSD2 y Toques tienen codigo funcional sustancial.

### Hallazgos parciales o dudosos

Calidad, Formacion, Google Workspace, Fidelizacion/Captacion y algunos submodulos de Marketing mantienen evidencia parcial o mock.

### Contradicciones detectadas

Algunas deudas descritas en `BUSINESS_LOGIC.md` ya no coinciden con el codigo actual.

### Riesgos detectados

Demo comercial puede enseñar zonas incompletas si se navega por todo el sidebar sin guion.

### Archivos o rutas criticas

`src/features/layout/components/app-sidebar.tsx`, `src/app/(main)`, `src/features/logistica`, `src/features/rrhh`, `src/features/marketing`.

### Incognitas para la siguiente fase

Profundidad real de logica, invariantes, autorizacion y persistencia en cada modulo.

## Fase 3 - Logic inference engine

## HANDOFF PARA SIGUIENTE FASE

### App detectada

Producto que intenta centralizar operacion de hosteleria: personal, ventas, cocina, compras, documentos, marketing, finanzas y gobierno.

### Tipo de repo y stack confirmado

Server actions y services concentran reglas; Supabase/RPC cubre persistencia y calculos especificos.

### Evidencias fuertes

`tickets-actions.ts`, `descontar-stock-por-ventas.ts`, `empleados-actions.ts`, `firmas-actions.ts`, `estudios-apertura-actions.ts`, `agora-ventas-sync.ts`, `psd2-actions.ts`.

### Hallazgos confirmados de la fase

Existen invariantes relevantes: cierre de tickets por total pagado, descuento de stock por escandallo, validacion de fichajes por radio, firmas con hash/auditoria, candidatura publica con rate limit.

### Hallazgos parciales o dudosos

Tenant boundary y autorizacion no son uniformes. Integraciones externas no fueron probadas con credenciales reales.

### Contradicciones detectadas

El sistema aplica algunas medidas de seguridad, pero tambien permite crons fail-open y bypass hardcoded.

### Riesgos detectados

Riesgo alto en server actions con service role y en crons si faltan variables de entorno.

### Archivos o rutas criticas

`src/app/api/cron/*`, `src/features/rrhh/actions/accesos-apps-actions.ts`, `src/features/auth/actions/permisos-actions.ts`, `src/proxy.ts`.

### Incognitas para la siguiente fase

Como presentar la app al usuario final sin confundir placeholders con capacidades productivas.

## Fase 4 - Menu reconstructor

## HANDOFF PARA SIGUIENTE FASE

### App detectada

SaaS departamental con menu lateral principal y hub de departamentos basado en permisos.

### Tipo de repo y stack confirmado

Next App Router con rutas privadas `(main)` y publicas separadas; sidebar declarativo en componente layout.

### Evidencias fuertes

`app-sidebar.tsx`, `MisDepartamentosView`, rutas bajo `src/app/(main)`.

### Hallazgos confirmados de la fase

La navegacion real incluye Mi Panel, Mis Departamentos, Direccion, Sala, Cocina, Logistica, Gerencia, RRHH, Marketing, Contabilidad, Gestoria, Juridico, Ajustes y utilidades.

### Hallazgos parciales o dudosos

Las paginas raiz de departamentos son menos utiles que subrutas concretas; algunas rutas existen sin backend fuerte.

### Contradicciones detectadas

La amplitud del menu sugiere producto completo, pero la madurez real varia por seccion.

### Riesgos detectados

Una auditoria visual superficial podria sobreestimar madurez por cantidad de menu.

### Archivos o rutas criticas

`src/features/layout/components/app-sidebar.tsx`, `src/app/(main)/*`, `src/features/mis-departamentos`.

### Incognitas para la siguiente fase

Como documentar una guia util que distinga uso real, uso demo y huecos.

## Fase 5 - Doc generator

## HANDOFF PARA SIGUIENTE FASE

### App detectada

Balles Hosteleros como SaaS especifico de operacion hostelera, no plantilla generica.

### Tipo de repo y stack confirmado

Documento consolidado en `docs/legacy/ANALYSIS.md` con guia funcional, evidencias, riesgos y limites.

### Evidencias fuertes

Mapa de modulos, tabla de rutas, estado por dimensiones, contradicciones y evidencia por codigo.

### Hallazgos confirmados de la fase

El entregable separa `confirmado por codigo`, `inferido`, `parcial` y `no confirmado`.

### Hallazgos parciales o dudosos

La guia no sustituye pruebas con credenciales reales ni validacion funcional con usuarios.

### Contradicciones detectadas

Se documentan contradicciones entre memoria funcional y codigo actual.

### Riesgos detectados

El documento debe actualizarse despues de reparar build y seguridad para no quedar obsoleto.

### Archivos o rutas criticas

`docs/legacy/ANALYSIS.md`, `BUSINESS_LOGIC.md`, `ROADMAP.md`.

### Incognitas para la siguiente fase

Veredicto final y decision de uso del repo por dimensiones.

## Fase 6 - SaaS evaluator

## HANDOFF PARA SIGUIENTE FASE

### App detectada

SaaS amplio, real y recuperable para Balles Hosteleros, con madurez desigual.

### Tipo de repo y stack confirmado

Next/Supabase productivo en intencion, pero no listo para produccion por build roto y riesgos confirmados.

### Evidencias fuertes

Typecheck/build fallido, modulos reales extensos, crons, proxy/auth, migraciones RLS y riesgos de service role.

### Hallazgos confirmados de la fase

Veredicto: APTO CON REPARACIONES. Puede continuar como producto propio, no como base generica ni produccion inmediata.

### Hallazgos parciales o dudosos

Valor demo alto si se controla el recorrido; valor productivo depende de reparar seguridad y tenant boundary.

### Contradicciones detectadas

Capacidad funcional grande frente a una puerta tecnica basica rota. Hardening parcial frente a fail-open por entorno.

### Riesgos detectados

Build/typecheck, crons sin secreto obligatorio, bypass hardcoded, service-role actions, multiempresa irregular y secretos en docs.

### Archivos o rutas criticas

`src/features/rrhh/actions/empleados-actions.ts`, `src/app/api/cron/*`, `src/features/rrhh/actions/accesos-apps-actions.ts`, `src/proxy.ts`, `BUSINESS_LOGIC.md`.

### Incognitas para la siguiente fase

Requiere QA-gate/security/tenant-boundary posterior antes de produccion o entrega a cliente.
