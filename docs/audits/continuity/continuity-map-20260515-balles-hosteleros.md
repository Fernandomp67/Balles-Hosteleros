# Mapa de Continuidad del SaaS

## Resumen ejecutivo

Balles Hosteleros no es una fachada global. La auditoria TRIBUNAL ya confirma una base real y amplia: Next.js App Router, Supabase, migraciones, server actions, crons, rutas publicas, CRUDs internos y modulos de dominio hostelero con logica propia.

Tampoco es un SaaS listo para produccion. El cierre TRIBUNAL es `NO-GO`: `qa-gate` falla por typecheck, lint y build; `security-auditor` reporta 9 bloqueantes; `tenant-boundary-auditor` concluye que la frontera SaaS/tenant esta solo parcialmente resuelta.

La lectura practica es:

- Se puede continuar el producto, pero antes hay que remediar base tecnica, seguridad y multiempresa.
- Se puede preparar demo controlada de algunos flujos, pero no venderlo como produccion estable.
- Los modulos mas aprovechables estan en direccion/aperturas, logistica, sala/POS, cocina, RRHH, carta/web publica, contabilidad, gerencia, gestoria/juridico y Toques/points.
- Las zonas mas cercanas a fachada o demo son calidad, formacion, camaras, partes de marketing avanzado, Google Workspace y algunas paginas raiz de departamentos.
- El orden correcto no es ampliar funcionalidad; es estabilizar gates, cerrar riesgos transversales y despues consolidar modulos reales.

Foto auditada:

| Campo | Valor |
| --- | --- |
| Repo | `https://github.com/balleshosteleros/Balles-Hosteleros/` |
| Worktree de continuidad | `/home/fernandomp/dev/Balles-Hosteleros-audit` |
| Base TRIBUNAL | `docs/audits/tribunal/*-20260515-balles-hosteleros.md` |
| Analisis funcional | `docs/legacy/ANALYSIS.md` |
| Branch auditada en informes | `upstream-feature-aperturas-fotos-ocupacion` |
| HEAD auditado en informes | `32cd5c3` |
| Commit local actual del worktree de auditoria | `c6528ac` |
| Decision de release | `NO-GO` |
| Decision de continuidad | `continuar-con-remediacion` |

## Matriz de modulos

| Modulo | Estado | Que funciona | Que no funciona o falta | Bloqueo principal | Uso recomendado | Siguiente accion | Evidencia |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Base tecnica y gates | `bloqueado` | `npm ci` es reproducible con Node `20.20.2`; el servidor dev levanta y `/login` responde. | `npm run lint` usa `next lint`, incompatible en Next 16; `npx tsc --noEmit` falla; `npm run build` cae por typecheck. | Sin typecheck/build/lint verdes no hay base fiable para produccion ni QA funcional completa. | `no-usar` | Corregir tipos RRHH, sustituir lint por ESLint 9/Next 16 compatible y repetir `tsc`, lint, build y smoke minimo. | `qa-gate-20260515-balles-hosteleros.md`; `package.json:9`; `src/features/rrhh/actions/empleados-actions.ts:60`; `src/features/rrhh/actions/empleados-actions.ts:405`. |
| Auth, permisos y multiempresa | `bloqueado` | Hay login, roles, selector de empresa activa, `empresa_id`, `empresa_roles`, `user_roles` y helpers como `getEmpresaActivaForUser`. | La membresia `user_empresas` se usa en codigo/RLS, pero no aparece como migracion reproducible; hay bypass por email y uso amplio de service role. | Tenant-boundary incompleto y riesgo cross-tenant. | `no-usar` | Crear migracion reproducible de `user_empresas`, eliminar bypass por email, unificar resolucion de tenant y revisar todas las actions con service role. | `tenant-boundary-audit-20260515-balles-hosteleros.md`; `security-audit-20260515-balles-hosteleros.md`; `src/features/rrhh/actions/empleados-actions.ts:31`; `src/features/rrhh/actions/empleados-actions.ts:158`. |
| Direccion y aperturas | `usable-con-riesgo` | Estudios de apertura tienen UI, server actions, persistencia por `estudios_apertura`, fotos en storage, signed URLs, ocupacion, local, marca, gastronomia y link publico. | La publicacion por slug y storage dependen de tenant/RLS no cerrado; no hay smoke end-to-end de crear/editar/compartir con credenciales reales. | Riesgos transversales de tenant y estudios compartidos anon enumerables. | `demo-controlada` | Probar flujo: crear estudio, subir fotos, editar ocupacion/local/gastronomia, activar link publico, abrir `/p/[slug]`; despues endurecer share publico. | `ANALYSIS.md` seccion Direccion; `src/features/direccion/actions/estudios-apertura-actions.ts:75`; `src/features/direccion/actions/estudios-apertura-actions.ts:202`; `src/features/direccion/actions/estudios-apertura-actions.ts:245`; `security-audit` SEC-005. |
| Direccion, cronogramas y documentacion | `parcial` | Hay rutas y componentes de estructura, cronogramas, documentacion, presentaciones y organigrama; algunas acciones y migraciones existen. | Conviven seed/mock y fallback en cronogramas/organigrama; no esta validado el flujo completo con datos reales multiempresa. | Evidencia dinamica insuficiente y dependencia de `user_empresas`/RLS. | `investigar-antes` | Separar subflujos reales de fallback; validar cronogramas con dos empresas y usuarios con permisos distintos. | `ANALYSIS.md`; `src/features/direccion/hooks/useCronogramasOperativos.ts`; `src/features/direccion/data/cronogramasMockData.ts`; `src/features/direccion/components/EstructuraView.tsx`. |
| Logistica | `usable-con-riesgo` | Proveedores, productos, pedidos, albaranes, stock, inventarios, tarifas, escandallos y Agora tienen actions, tablas y UI. `createPedido` inserta pedido y lineas con `empresa_id`. | Algunas policies legacy usan `using (true)`; Agora depende de secretos/cron; necesidad de compra no tiene UI principal confirmada. | RLS legacy y cron Agora fail-open/service role. | `demo-controlada` | Demo con proveedores/productos/pedido/albaran/stock sin ejecutar cron; despues cerrar RLS y proteger Agora. | `ANALYSIS.md`; `src/features/logistica/actions/pedidos-actions.ts:24`; `src/features/logistica/actions/pedidos-actions.ts:66`; `src/features/logistica/actions/pedidos-actions.ts:110`; `src/app/api/cron/agora-sync/route.ts:18`; `supabase/migrations/011_logistica_compras.sql`. |
| Sala/POS | `usable-con-riesgo` | POS crea tickets, lineas, totales, mesas, pagos/cierres y conecta lineas con cocina; hay descuento de stock por ventas. | Parte de la action usa service role; no hay smoke de caja completa ni validacion de permisos/tenant en todos los pasos. | Service role y falta de prueba E2E de venta completa. | `demo-controlada` | Smoke dirigido: abrir caja, crear ticket, anadir lineas, enviar a cocina, cobrar, comprobar stock/comandas. | `ANALYSIS.md`; `src/features/sala/pos/actions/tickets-actions.ts:8`; `src/features/sala/pos/actions/tickets-actions.ts:106`; `src/features/sala/pos/actions/tickets-actions.ts:156`; `src/features/sala/pos/services/descontar-stock-por-ventas.ts`. |
| Cocina | `usable-con-riesgo` | Comandas, nuevas recetas, escandallos, elaboraciones, partidas y temperaturas tienen acciones, tablas y UI; cocina recibe lineas del POS. | Hay RLS legacy en varias tablas y una inconsistencia pendiente de tipos/enum producto `Compra`/`compra`. | Riesgo de datos y normalizacion antes de tocar logica. | `demo-controlada` | Validar flujo POS -> comandas -> cambio de estado; despues revisar enum/productos y policies de cocina. | `ANALYSIS.md`; `src/features/cocina/comandas/actions/comandas-actions.ts`; `src/features/cocina/actions/escandallos-actions.ts`; `supabase/migrations/010_features_restantes.sql`; `supabase/migrations/037_cocina_comandas.sql`. |
| RRHH empleados | `bloqueado` | El dominio es real: empleados, auth user, profile, user role, relacion empresa, datos personales y estados. | Rompe typecheck en conversiones de `empresas`; depende de `user_empresas`; usa admin client para crear usuarios. | Bloqueo directo de build/typecheck y tenant-boundary. | `desarrollo-prioritario` | Arreglar TS2352, asegurar migracion `user_empresas`, validar alta/edicion/baja de empleado con empresa A/B y permisos admin. | `qa-gate-20260515-balles-hosteleros.md`; `src/features/rrhh/actions/empleados-actions.ts:16`; `src/features/rrhh/actions/empleados-actions.ts:92`; `src/features/rrhh/actions/empleados-actions.ts:126`; `src/features/rrhh/actions/empleados-actions.ts:397`. |
| RRHH fichajes, firmas, horarios y solicitudes | `usable-con-riesgo` | Hay acciones, UI y migraciones para fichajes, firmas, OTP/hash/PDF/email, horarios, patrones, turnos y solicitudes. | Requiere smoke con datos reales; crons de firmas/fichajes tienen patron fail-open si falta `CRON_SECRET`. | Seguridad de crons y validacion dinamica incompleta. | `uso-interno-limitado` | Probar fichaje entrada/salida, firma publica, expiracion controlada y horarios; cerrar crons antes de uso real. | `ANALYSIS.md`; `src/features/rrhh/actions/fichajes-actions.ts`; `src/features/rrhh/actions/firmas-actions.ts`; `src/app/api/cron/firmas-expirar/route.ts`; `src/app/api/cron/cerrar-fichajes-huerfanos/route.ts`. |
| RRHH reclutamiento publico | `usable-con-riesgo` | `/empleo`, candidaturas, PDF, rate limit y Turnstile opcional aparecen implementados. | No se valido flujo externo completo ni entregabilidad/email; `frame-ancestors *` fue marcado como riesgo. | Validacion publica y compliance. | `demo-controlada` | Smoke con candidatura publica, CV/PDF, revision en RRHH y limites anti-abuso. | `ANALYSIS.md`; `src/app/empleo/[slug]/page.tsx`; `src/app/api/empleo/candidatura/route.ts`; `security-audit-20260515-balles-hosteleros.md`. |
| RRHH accesos apps | `bloqueado` | Hay tabla, UI y actions para accesos a apps externas. | La tabla almacena `usuario` y `contrasena`; RLS esta abierta a autenticados; actions usan service role y la UI puede mostrar contrasenas. | Riesgo critico de credenciales y cross-tenant. | `no-usar` | Desactivar exposicion de contrasenas, redisenar tabla con `empresa_id`, cifrado, RLS por membership y acciones server-side autorizadas. | `security-audit` SEC-003; `supabase/migrations/060_accesos_apps.sql:15`; `supabase/migrations/060_accesos_apps.sql:26`; `supabase/migrations/060_accesos_apps.sql:58`; `src/features/rrhh/actions/accesos-apps-actions.ts`. |
| Marketing campanas | `parcial` | Hay CRUD de campanas e integraciones Resend, WhatsApp Cloud y Meta Ads en codigo. | No se verificaron credenciales vivas ni envios reales; algunas zonas de marketing mantienen mocks/localStorage. | Integraciones externas no validadas y seguridad/env. | `investigar-antes` | Validar credenciales por entorno, enviar campana de prueba controlada y separar mocks de produccion. | `ANALYSIS.md`; `src/features/marketing/actions/campanas-actions.ts`; `src/features/marketing/services/resend-service.ts`; `src/features/marketing/services/whatsapp-service.ts`; `src/features/marketing/services/meta-ads-service.ts`. |
| Carta digital | `usable-con-riesgo` | Admin de categorias/items/fotos, slug publico, likes y revalidacion existen; las actions filtran por `empresa_id`. | No hay smoke publico completo ni revision de RLS/abuso de likes en entorno real. | Validacion dinamica y hardening publico. | `demo-controlada` | Crear carta, publicar slug, abrir `/carta/[slug]`, subir foto, probar like y revisar rate limit. | `ANALYSIS.md`; `src/features/marketing/carta-digital/actions/carta-admin-actions.ts:37`; `src/features/marketing/carta-digital/actions/carta-admin-actions.ts:169`; `src/app/carta/[slug]/page.tsx`. |
| Pagina web publica | `usable-con-riesgo` | Constructor de paginas, bloques, dominios, host resolver, leads y render `__site` existen; CRUD filtra por `empresa_id`. | `/` puede devolver 404 si no hay host valido; importador de URL tiene SSRF autenticado; no se valido deploy/dominios. | SSRF y configuracion publica/dominio. | `demo-controlada` | Probar con host local o preview: crear pagina, publicar, capturar lead y bloquear importador hasta fix SSRF. | `ANALYSIS.md`; `src/features/marketing/pagina-web/actions/paginas-actions.ts:29`; `src/features/marketing/pagina-web/actions/paginas-actions.ts:75`; `src/app/__site/[[...slug]]/page.tsx`; `src/app/api/pagina-web/importar-url/route.ts`; `security-audit` SEC-006. |
| Marketing fidelizacion, captacion y contenido | `parcial` | Hay rutas y vistas para calendario, contenido, fidelizacion y captacion. | Informes y codigo indican mocks/localStorage o wiring incompleto en varias areas; no hay persistencia uniforme confirmada. | Predominio parcial de UI frente a backend validado. | `desarrollo-prioritario` | Inventariar cada submodulo y decidir: conectar a Supabase, aparcar o eliminar de demo. | `ANALYSIS.md`; `src/features/marketing/components/FidelizacionView.tsx`; `src/features/marketing/components/CaptacionView.tsx`; `src/features/marketing/contexts/marketing-context.tsx`. |
| Contabilidad basica | `usable-con-riesgo` | Contactos, facturas y transacciones tienen actions CRUD con `empresa_id`; hay vistas para bancos, conciliacion, reglas, etiquetas e impuestos. | El contexto usa `profiles.empresa_id`, no tenant activo multiempresa; algunas updates no filtran por `empresa_id`; conciliacion/reglas no esta validada end-to-end. | Tenant-boundary y falta de smoke contable. | `uso-interno-limitado` | Smoke con contacto/factura/transaccion; despues unificar contexto tenant y filtrar updates por empresa. | `ANALYSIS.md`; `src/features/contabilidad/actions/contabilidad-actions.ts:5`; `src/features/contabilidad/actions/contabilidad-actions.ts:21`; `src/features/contabilidad/actions/contabilidad-actions.ts:91`; `src/features/contabilidad/actions/contabilidad-actions.ts:132`. |
| Bancos PSD2 y conciliacion | `parcial` | Hay provider GoCardless, cifrado AES-GCM y cron PSD2. | No se validaron credenciales vivas ni sincronizacion real; cron PSD2 usa service role y secreto opcional. | Integracion externa y cron fail-open. | `investigar-antes` | Validar sandbox GoCardless, cerrar cron y definir smoke de conexion/sync/conciliacion. | `ANALYSIS.md`; `src/features/contabilidad/actions/psd2-actions.ts`; `src/features/contabilidad/services/psd2/providers/gocardless.ts`; `src/app/api/cron/psd2-sync/route.ts`. |
| Gerencia | `usable-con-riesgo` | Mantenimiento, vencimientos, cierres, comunicados, encuestas e informes tienen actions/tablas; ratios tiene UI. | Ratios no tiene formula completa validada; no hay smoke de cierres/informes con datos reales. | Validacion funcional y datos reales. | `uso-interno-limitado` | Priorizar mantenimiento/vencimientos/cierres; dejar ratios como parcial hasta validar calculos. | `ANALYSIS.md`; `src/features/gerencia/actions/mantenimiento-actions.ts`; `src/features/gerencia/actions/cierres-actions.ts`; `src/features/gerencia/actions/informes-actions.ts`; `src/features/gerencia/components/RatiosView.tsx`. |
| Gestoria | `usable-con-riesgo` | Modelos AEAT, reglas, presentaciones y contrataciones existen en rutas, acciones y servicios PDF. | No se valido validez fiscal/legal ni integraciones oficiales externas. | Riesgo legal/fiscal, no tecnico puro. | `uso-interno-limitado` | Usar como borrador interno; validar modelos con asesor fiscal antes de produccion. | `ANALYSIS.md`; `src/app/(main)/gestoria/modelos/page.tsx`; `src/features/gestoria`; `src/app/api/modelos-aeat/[id]/pdf/route.ts`. |
| Juridico | `usable-con-riesgo` | Procesos y documentos juridicos tienen rutas, actions, storage y PDF. | No se valido validez juridica ni permisos finos de documentos. | Compliance y tenant-boundary. | `uso-interno-limitado` | Smoke de proceso/documento y revision de permisos/documentos antes de uso sensible. | `ANALYSIS.md`; `src/features/juridico/actions/procesos-actions.ts`; `src/features/juridico/actions/documentos-actions.ts`; `src/features/juridico/utils/reporte-pdf.ts`. |
| Mi Panel empleado | `parcial` | Hay muchas rutas de perfil, datos personales, fichajes, documentos, formacion, points, horario y equipo. | Depende de los mismos modulos base; algunas areas de formacion/documentos/cronograma mezclan datos reales con seeds/fallback. | Madurez desigual por submodulo. | `desarrollo-prioritario` | Probar solo perfil, fichajes y points al principio; aparcar submodulos dependientes de mocks. | `ANALYSIS.md`; `src/app/(main)/mi-panel/*`; `src/features/mi-panel/actions/*`; `src/features/formacion/store/use-formacion-store.ts:6`. |
| Toques/points | `usable-con-riesgo` | Hay reglas, recompensas, balances, ranking, canjes, servicios y crons de devengo/snapshot. | Falta smoke de devengo/canje y revision de permisos de administracion. | Validacion dinamica. | `demo-controlada` | Smoke con usuario empleado: ver balance, canjear, admin otorga toque, cron snapshot/devengo en entorno controlado. | `ANALYSIS.md`; `src/features/toques/actions/toques-actions.ts`; `src/features/toques/services/reglas-runner.service.ts`; `src/app/api/points/cron/devengo-diario/route.ts`. |
| Formacion | `fachada` | Hay portal, cursos, secciones, lecciones, novedades y experiencia UI usable. | El propio store declara persistencia en `localStorage` como mock pendiente de Supabase; no hay actions principales detectadas. | Sin persistencia backend real. | `desarrollo-prioritario` | Decidir si se convierte a modulo real; crear tablas/actions o excluir de demo comercial. | `ANALYSIS.md`; `src/features/formacion/store/use-formacion-store.ts:3`; `src/features/formacion/store/use-formacion-store.ts:6`; `src/features/formacion/data/seed.ts`. |
| Calidad | `fachada` | Existen rutas y componentes de dashboard, auditorias, clientes, empleados e inspecciones. | El dashboard muestra KPIs vacios y texto de modulo pendiente; no se detectaron actions/tablas propias. | Es principalmente UI/placeholder. | `no-usar` | Aparcar o reconstruir desde contrato funcional real de calidad. | `ANALYSIS.md`; `src/features/calidad/components/CalidadDashboardView.tsx:10`; `src/features/calidad/components/CalidadDashboardView.tsx:14`; `src/features/calidad/components/CalidadDashboardView.tsx:74`. |
| Google Workspace | `parcial` | Hay drawers, botones y rutas API para Gmail, Calendar, People, connect/disconnect y sync. | No se valido OAuth vivo; hay zonas de tareas/telefono/chat con UI local y riesgo XSS en HTML Gmail/firma. | Integracion externa, OAuth y XSS. | `investigar-antes` | Validar OAuth en entorno controlado, sanitizar HTML y separar drawers demo de integracion real. | `ANALYSIS.md`; `src/app/api/google/*`; `src/features/google-workspace/components/*`; `security-audit` SEC-007. |
| Soporte IA | `bloqueado` | Hay chat de soporte, base de conocimiento y fallback sin IA. | Endpoints IA son publicos, sin auth/rate limit suficiente segun security-auditor; `base-conocimiento` se declara mock. | Exposicion publica y coste/abuso de proveedor IA. | `no-usar` | Exigir sesion o rate limit fuerte, cuotas y logging antes de habilitar IA. | `security-audit` SEC-008; `src/app/api/soporte/chat/route.ts:23`; `src/app/api/soporte/chat/route.ts:65`; `src/lib/soporte/base-conocimiento.ts`. |
| Camaras/recorder | `fachada` | Hay drawer y componentes de grabacion/recorder. | Camaras usan `localStorage` y placeholder de stream RTSP/ONVIF; no hay integracion real confirmada. | Sin backend ni integracion viva. | `no-usar` | Excluir de demo salvo como prototipo visual; definir arquitectura real si se quiere mantener. | `src/features/camaras/components/CamarasDrawer.tsx:75`; `src/features/camaras/components/CamarasDrawer.tsx:513`; `src/features/recorder/*`. |
| APIs publicas y crons | `bloqueado` | Hay rutas para empleo, leads, carta, estudios, firmas, crons Agora/PSD2/fichajes/firmas/points. | `/api/*` queda publico a nivel middleware; varios crons usan service role y solo validan secreto si `CRON_SECRET` existe; importador web tiene SSRF. | Superficie publica y operaciones globales con service role. | `no-usar` | Hacer fail-closed de crons, auditar cada route handler, limitar importador y aplicar auth/rate limit por ruta. | `tenant-boundary-audit-20260515-balles-hosteleros.md`; `security-audit-20260515-balles-hosteleros.md`; `src/app/api/cron/agora-sync/route.ts:18`; `src/app/api/cron/agora-sync/route.ts:33`. |

## Roadmap recomendado

### Fase 0 - Base obligatoria

Objetivo: que el repositorio pueda compilar y pasar gates reproducibles.

1. Corregir los errores TS2352 en `src/features/rrhh/actions/empleados-actions.ts`.
2. Sustituir `npm run lint` basado en `next lint` por un comando compatible con Next 16 y ESLint 9.
3. Repetir `npx tsc --noEmit`, `npm run lint`, `npm run build`.
4. Definir smoke minimo local con login y una ruta interna autenticada.
5. Confirmar `.env.example`, `.nvmrc`, `package-lock.json` y `.npmrc` como contrato reproducible.

No avanzar a nuevas features antes de cerrar esta fase.

### Fase 1 - Seguridad y boundaries

Objetivo: reducir los bloqueantes que hacen inviable produccion.

1. Rotar token Agora y retirar/redactar secretos versionados.
2. Eliminar bypass hardcoded por email.
3. Hacer `CRON_SECRET` obligatorio en produccion y fallar cerrado en todos los crons con service role.
4. Crear migracion reproducible de `user_empresas` con constraints, indices, RLS y backfill.
5. Unificar resolucion de tenant activo y eliminar fallback a todas las empresas salvo superadmin formal.
6. Redisenar `accesos_apps`: `empresa_id`, RLS por membership, cifrado y no devolver contrasenas al cliente.
7. Corregir SSRF del importador web y XSS de Gmail/firma.
8. Contener `SUPABASE_SERVICE_ROLE_KEY` con wrappers server-only, actor, empresa, motivo y logging.

### Fase 2 - Consolidar modulos reales

Objetivo: convertir lo ya real en producto demostrable y mantenible.

1. Direccion/aperturas: crear/editar/compartir estudio con fotos y ocupacion.
2. Logistica: proveedor/producto/pedido/albaran/stock.
3. Sala/POS + cocina: venta, envio a cocina, cobro y descuento de stock.
4. RRHH basico: alta/edicion/baja empleado, fichajes y firmas.
5. Carta digital y pagina web publica: publicar carta/web y capturar lead.
6. Contabilidad basica: contacto, factura, transaccion y banco en modo sandbox.
7. Toques/points: balance, canje, otorgamiento y cron controlado.

### Fase 3 - Convertir parciales

Objetivo: decidir que modulos parciales merecen backend real.

1. Marketing fidelizacion/captacion/contenido: conectar persistencia o retirar de demo.
2. Google Workspace: validar OAuth, Gmail/Calendar y sanitizacion.
3. Gerencia ratios: fijar formulas y datos fuente.
4. Mi Panel avanzado: documentos, formacion y cronogramas con datos reales.
5. Direccion cronogramas: eliminar dependencia de mocks/fallback cuando haya empresa activa.

### Fase 4 - Decidir fachada

Objetivo: no arrastrar UI que parezca producto pero no lo sea.

1. Calidad: reconstruir o ocultar hasta que tenga contrato funcional, tablas y actions.
2. Formacion: convertir `localStorage` a Supabase o marcar como prototipo.
3. Camaras: excluir salvo que se defina integracion real RTSP/ONVIF o proveedor.
4. Soporte IA: no habilitar hasta cerrar auth, rate limit, cuotas y logging.
5. Raices de departamentos placeholder: redirigir a submodulos reales o convertir en dashboards utiles.

## Smoke tests dirigidos

Estos smoke tests no sustituyen la remediacion. Sirven para confirmar que modulos con valor real pueden mostrarse en demo controlada despues de Fase 0 y de los bloqueos criticos de Fase 1 que afecten a cada flujo.

| Smoke | Modulos cubiertos | Pasos minimos | Criterio de exito |
| --- | --- | --- | --- |
| Login y contexto tenant | Auth, permisos, multiempresa | Iniciar sesion, seleccionar empresa, entrar a `/mis-departamentos`, comprobar menu visible. | El usuario ve solo empresa y modulos esperados; no aparecen errores de sesion/contexto. |
| Estudio de apertura | Direccion/aperturas | Crear estudio, editar local, subir foto, rellenar ocupacion, activar compartir, abrir `/p/[slug]`. | Datos persisten, fotos generan signed URL y el publico ve solo el estudio compartido. |
| Pedido y stock | Logistica | Crear proveedor/producto, crear pedido con lineas, confirmar albaran, revisar stock. | El pedido queda guardado, lineas se crean y stock cambia segun flujo previsto. |
| POS a cocina | Sala/POS, cocina | Abrir caja, crear ticket, anadir productos, enviar a cocina, cambiar estado de comanda, cobrar. | Ticket y comanda se reflejan; el cobro cierra el flujo sin error. |
| RRHH basico | RRHH empleados | Crear empleado, asignar empresa, editar datos, cambiar estado y comprobar Mi Panel. | El empleado tiene auth/profile/empleado y acceso a la empresa correcta. |
| Firma publica | RRHH firmas | Crear documento, enviar token/OTP, abrir `/firmar/[token]`, firmar y descargar/ver PDF. | Firma queda auditada y el documento cambia de estado. |
| Carta digital | Marketing carta | Crear categoria/item/foto, configurar slug, abrir `/carta/[slug]`, probar like. | Carta publica carga con datos correctos y el like respeta limites. |
| Pagina publica y lead | Marketing web | Crear pagina, publicar, abrir host/preview, enviar formulario lead. | La pagina renderiza y el lead queda persistido. |
| Contabilidad basica | Contabilidad | Crear contacto, factura y transaccion; revisar listados. | Los tres registros quedan filtrados por empresa. |
| Toques/points | Toques | Otorgar toque, ver balance, canjear recompensa, ejecutar snapshot/devengo en entorno controlado. | Balance y canje cambian de forma trazable. |

## Decisiones de producto/desarrollo

Decision final: `continuar-con-remediacion`.

No conviene abandonar el repo: contiene dominio real, modulos conectados, migraciones, UI interna amplia y varias superficies publicas implementadas.

No conviene seguir ampliando funcionalidad ahora: el producto esta bloqueado por gates, seguridad y tenant-boundary.

No conviene presentarlo como produccion: aunque varios modulos son demostrables, el `NO-GO` TRIBUNAL impide considerar seguro el uso real con datos sensibles.

Decision por frente:

| Frente | Decision |
| --- | --- |
| Produccion | Bloqueada hasta cerrar Fase 0 y Fase 1. |
| Demo controlada | Permitida solo con modulos reales y datos no sensibles. |
| Desarrollo inmediato | Enfocar en gates, seguridad, tenant y modulos reales. |
| Modulos fachada | Ocultar, aparcar o reconstruir; no mezclarlos en demo principal. |
| Nuevas features | Posponer hasta que el core compile y tenga boundaries seguros. |

Orden recomendado de trabajo si un equipo pide ayuda para continuar:

1. Reparar gates de build/typecheck/lint.
2. Cerrar bloqueantes de seguridad y tenant que afectan a datos o credenciales.
3. Preparar demo controlada con aperturas, logistica, POS/cocina, carta/web y RRHH basico.
4. Consolidar persistencia y tests de los modulos reales.
5. Decidir uno por uno los modulos parciales: convertir, ocultar o eliminar.

## Evidencia y limites

Fuentes usadas:

- `docs/legacy/ANALYSIS.md`
- `docs/audits/tribunal/tribunal-audit-20260515-balles-hosteleros.md`
- `docs/audits/tribunal/final-audit-20260515-balles-hosteleros.md`
- `docs/audits/tribunal/qa-gate-20260515-balles-hosteleros.md`
- `docs/audits/tribunal/security-audit-20260515-balles-hosteleros.md`
- `docs/audits/tribunal/tenant-boundary-audit-20260515-balles-hosteleros.md`
- Codigo local en `/home/fernandomp/dev/Balles-Hosteleros-audit/src`
- Migraciones locales en `/home/fernandomp/dev/Balles-Hosteleros-audit/supabase/migrations`

Consultas de codigo realizadas para este mapa:

- Rutas reales bajo `src/app`.
- Actions y servicios bajo `src/features`.
- Migraciones Supabase, especialmente RLS, `accesos_apps`, POS, logistica, aperturas, carta, pagina web y RRHH.
- Busquedas de `localStorage`, `mock`, `placeholder`, `createAdminClient`, `SUPABASE_SERVICE_ROLE_KEY`, `CRON_SECRET`, `dangerouslySetInnerHTML`, `user_empresas` y `using (true)`.

Limites:

- No se han ejecutado fixes.
- No se han ejecutado smoke tests reales en esta pasada.
- No se han validado credenciales vivas de Supabase, Vercel, Agora, GoCardless, Resend, WhatsApp, Meta, Google, R2, OpenRouter ni Turnstile.
- La clasificacion puede mejorar despues de cerrar gates y ejecutar smokes.
- La etiqueta `operativo` no se ha usado porque el repo tiene `NO-GO` global y no hay smokes end-to-end verdes posteriores a la auditoria.

