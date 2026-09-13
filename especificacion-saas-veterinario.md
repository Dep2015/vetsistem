# Especificación de Requisitos Técnicos y Funcionales
## SaaS de Gestión para Clínicas Veterinarias — Perú

| Campo | Valor |
|---|---|
| Versión | 1.0 |
| Fecha | 2026-09-02 |
| Estado | Borrador para diseño |
| Producto | *(nombre por definir)* |
| Modelo | SaaS multi-tenant, suscripción mensual/anual |
| Mercado | Consultorios, clínicas y cadenas veterinarias en Perú |

> Este documento consolida las decisiones tomadas durante el diseño: modelo de negocio, arquitectura, requisitos técnicos, requisitos de seguridad (OWASP Top 10 + DevSecOps + marco legal peruano) y el detalle funcional de cada módulo. Está pensado para convertirse directamente en backlog.

---

## Índice

1. Alcance y contexto
2. Arquitectura y requisitos técnicos
3. Requisitos de seguridad y cumplimiento
4. Módulos funcionales (detalle por módulo e ítem, M01–M20)
5. Matriz módulos × planes
6. Roadmap sugerido (MVP → v2)
7. Riesgos residuales

---

# 1. Alcance y contexto

## 1.1 Propuesta de valor

Sistema vertical para veterinarias peruanas con tres diferenciadores frente al mercado (OKFAC, Wirevet, POS genéricos):

1. **Facturación electrónica SUNAT nativa e incluida** en todos los planes pagos (no "a consultar").
2. **Recordatorios al tutor a costo cercano a cero** mediante PWA + web push, con SMS y WhatsApp como respaldo.
3. **Cumplimiento legal integrado**: consentimiento verificable (ley anti-spam), Ley 29733 de protección de datos y acuerdo de tratamiento con cada clínica.

## 1.2 Planes comerciales

| Plan | Precio ref. | Usuarios | Alcance |
|---|---|---|---|
| Gratis | S/0 | 1 | Historia clínica, agenda, carnet de vacunas. Sin facturación. |
| Básico | S/59/mes | 2 | + SUNAT, POS, inventario, recordatorios (cupo) |
| Pro | S/99/mes | 5 | + hospitalización, reserva online, reportes, grooming |
| Clínica / Multi-sede | S/169/mes | Ilimitados | + multi-sede, BI, app tutor completa, API |

Cobros adicionales: setup/migración asistida (S/150–300 único, opcional), cupo extra de mensajería, plan anual con 2 meses gratis.

**Modelo de adquisición: autoservicio.** La clínica se registra sola, obtiene su espacio en el plan Gratis en el acto y sube de plan cuando quiera desde la sección **"Mi plan"** de su panel (ver M20). El Superadmin supervisa, no da de alta.

## 1.3 Actores

| Actor | Descripción |
|---|---|
| Superadmin (proveedor SaaS) | Administra tenants, planes, cupos, gateways, salud del sistema |
| Admin de clínica | Dueño/administrador. Se registra por autoservicio, crea el espacio, configura la clínica, usuarios, precios, ve reportes y gestiona el plan |
| Veterinario | Atiende, escribe historia clínica, receta, ordena exámenes |
| Recepción / Caja | Agenda, cobra, factura, vende en pet shop |
| Auxiliar / Groomer | Hospitalización, peluquería, tareas asignadas |
| Tutor (dueño de mascota) | Usuario externo vía PWA: historial, carnet, citas, notificaciones |
| Dispositivo gateway SMS | Actor no humano, no confiable, con alcance mínimo |

## 1.4 Escala objetivo

- 30–100 clínicas en 12–18 meses.
- ~1,000 usuarios internos, ~50,000 tutores, ~80,000 pacientes.
- ~6,000–20,000 notificaciones/mes.
- Un solo VPS con `docker-compose` cubre este rango; crecimiento por separación de servicios, no por Kubernetes.

---

# 2. Arquitectura y requisitos técnicos

## 2.1 Stack tecnológico

| Capa | Tecnología | Notas |
|---|---|---|
| Backend / API | PHP + Laravel | Versión LTS vigente, verificada sin CVE críticos antes de fijar `composer.json` |
| Base de datos | MySQL 8.x | InnoDB, `utf8mb4`, zona horaria `America/Lima` |
| Cache / colas / locks / pub-sub | Redis 7.x | **Dos instancias**: `redis-cache` y `redis-queue` (ver 2.4) |
| Workers | Laravel Horizon | Supervisores por prioridad (ver 2.5) |
| Tiempo real | Laravel Reverb (WebSockets) | Alertas internas de clínica; backplane Redis |
| Servidor web | Nginx | TLS 1.2+, HTTP/2, cabeceras de seguridad |
| Frontend panel | Blade + Livewire o Inertia (Vue) | Elegir uno; no mezclar |
| PWA tutor | Vue/Vanilla + Service Worker + Web Push | Instalable por QR |
| Push | Firebase Cloud Messaging | Gratuito, ilimitado; cubre Android, iOS (APNs) y web |
| SMS | Pool de gateways Android (app open-source) | Rate limit y cupos por Redis |
| WhatsApp | BSP oficial (API Cloud) | Solo plantillas *utility*; cupo por plan |
| Facturación electrónica | OSE (p. ej. NubeFacT) vía API REST | Cuenta única multiempresa del proveedor |
| Object storage | S3-compatible (Cloudflare R2 / Backblaze B2) | Adjuntos clínicos; URLs firmadas |
| Contenedores | Docker + docker-compose | Imágenes oficiales `-alpine`/`-slim` |
| Monitoreo | Zabbix + logs estructurados JSON | Ya disponible en la operación |
| Automatización lateral | n8n | Reportes por correo, onboarding, integraciones |
| CI/CD | GitHub Actions (o GitLab CI) | SAST, SCA, tests, build, deploy |

## 2.2 Paradigma y organización del código

**Arquitectura por capas** (no hexagonal). Justificación: Laravel es opinado en MVC; el costo de hexagonal no se justifica con un equipo de 1–2 personas. La frontera que sí se aplica de forma estricta es la de **tenant**.

```
app/
├── Http/Controllers/     # delgados: validar request, llamar Action, responder
├── Http/Requests/        # FormRequests: validación lista blanca
├── Http/Middleware/      # ResolveTenant, EnforceRole, RateLimit
├── Actions/              # un caso de uso por clase (CrearConsulta, EmitirComprobante)
├── Services/             # integraciones externas (OseClient, FcmService, SmsGatewayPool)
├── Models/               # Eloquent; todos los de tenant heredan de TenantModel
├── Jobs/                 # trabajos en cola (EnviarRecordatorio, EmitirCpe)
├── Policies/             # autorización por recurso
├── Events/ Listeners/    # auditoría, notificaciones internas
└── Support/Tenancy/      # resolución y scope de tenant
```

Reglas:
- Ningún controlador accede a Eloquent directamente para escribir; pasa por una Action.
- Ninguna consulta a tabla de tenant sin el scope global (verificado por test).
- Toda validación y autorización ocurre en servidor; el cliente nunca es fuente de verdad.

## 2.3 Modelo multi-tenant

- **Base de datos única compartida** con columna `tenant_id` en toda tabla de negocio.
- Resolución de tenant por subdominio (`patitas.app.dominio.pe`) o por `tenant_id` del usuario autenticado; nunca por parámetro manipulable del cliente.
- `TenantModel` con scope global + asignación automática de `tenant_id` en `creating`.
- Prefijo de cache y de claves Redis por tenant (`t{ID}_`).
- Rutas de storage por tenant: `tenants/{ID}/pacientes/{PACIENTE}/…`.
- Tests obligatorios: (a) todo modelo de negocio tiene scope; (b) IDOR cross-tenant devuelve **404** (no 403).
- Multi-sede: entidad `sucursal` dentro del tenant; usuarios y stock pueden acotarse por sucursal.

## 2.4 Redis — roles e instancias

| Uso | Instancia | `maxmemory-policy` | Persistencia |
|---|---|---|---|
| Colas Horizon | `redis-queue` | `noeviction` | AOF `everysec` |
| Locks / idempotencia / rate limit | `redis-queue` | `noeviction` | AOF |
| Sesiones | `redis-queue` (DB 2) | `noeviction` | AOF |
| Cache de aplicación | `redis-cache` | `allkeys-lru` | Ninguna |
| Pub/Sub Reverb | `redis-cache` (DB 1) | — | Ninguna |

Regla inviolable: **nunca colas en una instancia con política de evicción.** Redis es transporte, no fuente de verdad: todo lo que pasa por cola tiene su registro en MySQL.

## 2.5 Colas, jobs y scheduler

Supervisores Horizon:

| Supervisor | Colas | Procesos | Reintentos / backoff |
|---|---|---|---|
| `critical` | `sunat` | 3 | 5 intentos, [10, 60, 300, 900, 3600] s |
| `alerts` | `alerts`, `default` | 5 | 3 intentos, [30, 120, 600] s |
| `low` | `reports`, `exports` | 1 | 2 intentos |

Reglas:
- Todo job es **idempotente** (lock por clave de negocio: `cpe:{id}`, `recordatorio:{id}`).
- Jobs fallidos van a `failed_jobs` y disparan alerta Zabbix.
- Scheduler diario (08:00 Lima): genera recordatorios de vacunas/desparasitación/controles por tenant y encola.
- Scheduler cada 5 min: reintenta comprobantes en estado `pendiente_ose`.
- Scheduler nocturno: resumen diario de boletas, expiración de tokens, limpieza de sesiones.

## 2.6 Mensajería — escalera de canales

| Prioridad | Canal | Costo/mensaje aprox. | Condición |
|---|---|---|---|
| 1 | Web Push (PWA) | S/0 | Tutor con token push válido |
| 2 | SMS (gateway propio) | ~S/0.008 | Sin push; mensaje transaccional; cupo disponible |
| 3 | WhatsApp utility | ~S/0.02–0.035 | Crítico o configurado por la clínica; cupo disponible |
| ✗ | Comercial sin consentimiento | — | Bloqueado por sistema |

- Tabla `notificaciones` (outbox) con estados `pendiente → enviado → entregado → fallido`, canal, costo, tenant.
- Cupos por plan (`cupo_sms`, `cupo_whatsapp`) descontados atómicamente en Redis y conciliados en MySQL.
- Separación **transaccional / comercial** a nivel de tipo de mensaje; el comercial exige `consentimiento_marketing`.
- Toda plantilla SMS incluye "Responde BAJA para no recibir más"; el BAJA se procesa automáticamente.

### Pool de gateways SMS
- N dispositivos Android dedicados con app gateway open-source, cada uno con credencial propia.
- Rate limit por gateway (≈12 SMS/min) y tope diario configurable (≈150–200) para no gatillar antifraude del operador.
- Round-robin y salud del gateway en Redis; gateway sin heartbeat > 5 min queda fuera de rotación.
- Recepción de respuestas: `SI` (confirmar cita), `BAJA` (opt-out), `NO` (cancelar).

## 2.7 PWA del tutor

- `manifest.json`, Service Worker con cache **solo de assets estáticos** (nunca datos clínicos).
- Web Push vía FCM; en iOS requiere PWA agregada a pantalla de inicio (mostrar tutorial de 3 pasos para Safari).
- Vinculación por QR: `https://app.dominio.pe/v/{clinica}/vincular/{token}` — token de un solo uso, TTL 72 h.
- QR impreso en boleta, carnet de vacunas y sticker de mostrador.
- Sesión del tutor por cookie `HttpOnly` de mismo origen; nunca tokens en `localStorage`.
- Métricas expuestas a la clínica: % tutores con PWA instalada, ahorro mensual estimado.

## 2.8 Integraciones externas

| Integración | Protocolo | Consideraciones |
|---|---|---|
| OSE (SUNAT) | REST/JSON | Cuenta multiempresa; certificado por clínica cifrado; reintentos; ambiente demo y producción |
| FCM | HTTP v1 | Credencial de servicio solo en backend |
| WhatsApp Cloud API | REST + webhooks | Verificación de firma de webhook; solo plantillas *utility* aprobadas |
| Gateway SMS | REST (polling desde el dispositivo) | Dispositivo hostil; alcance mínimo |
| Object storage | S3 API | Bucket privado; URLs firmadas de 5–15 min |
| n8n | Webhooks firmados | Automatizaciones fuera del core |
| API pública (plan Clínica) | REST + tokens con scopes | Rate limit por token |

## 2.9 Observabilidad

- Logs estructurados JSON (`request_id`, `tenant_id`, `user_id`, `acción`), **sin datos personales ni secretos**.
- Zabbix: profundidad de colas (`LLEN queues:sunat`), memoria Redis, jobs fallidos, latencia p95, espacio en disco, heartbeat de gateways SMS, certificados TLS por vencer.
- Health check `/health` (app, MySQL, Redis, OSE reachability) sin exponer versiones.
- Alertas: cola `sunat` > 50 pendientes por > 10 min; error rate > 2 %; gateway SMS caído; backup fallido.

## 2.10 Infraestructura y despliegue

- VPS 4 vCPU / 8 GB en región **Miami/Ashburn** (latencia 60–90 ms a Lima). La transferencia internacional de datos se documenta conforme a la Ley 29733 (ver 3.1).
- `docker-compose` con servicios: `nginx`, `app`, `worker`, `scheduler`, `reverb`, `mysql`, `redis-queue`, `redis-cache`. Cada servicio con su propio `.env` y `.env.example`; `.env` en `.gitignore`.
- Backups: MySQL diario (dump cifrado) + storage versionado; retención 30 días; **restore probado mensualmente**.
- Homelab: **solo dev/staging y réplica de respaldo**; nunca producción.
- Dominios: `app.dominio.pe` (panel y PWA), `api.dominio.pe` (API pública), `{tenant}.app.dominio.pe` (opcional).

## 2.11 Rendimiento y escalabilidad

| Métrica | Objetivo |
|---|---|
| p95 de respuesta en panel | < 400 ms |
| Emisión de comprobante (encolado) | < 200 ms al usuario; confirmación OSE asíncrona |
| Búsqueda de paciente | < 150 ms (índices por `tenant_id + nombre/documento`) |
| Disponibilidad | 99.5 % mensual (objetivo inicial) |
| Capacidad | 100 tenants / 1,000 usuarios concurrentes en un VPS |

Índices compuestos siempre con `tenant_id` como primera columna. Paginación obligatoria en toda lista. Consultas N+1 detectadas en CI.

## 2.12 CI/CD y DevSecOps

Pipeline en cada push a `main`/`release`:
1. Lint + análisis estático (PHPStan nivel ≥ 6, Larastan).
2. SAST (Semgrep o Psalm security).
3. SCA: `composer audit`, `npm audit`, Trivy sobre imágenes.
4. Escaneo de secretos (gitleaks).
5. Tests (unitarios + integración + **tests de aislamiento de tenant**).
6. Build de imágenes con tag inmutable (SHA).
7. Deploy con migraciones y rollback definido.
8. DAST programado (OWASP ZAP) contra staging semanalmente.

Nada se despliega a producción si falla cualquier paso 1–5.

---

# 3. Requisitos de seguridad y cumplimiento

## 3.1 Marco legal peruano

| Norma | Obligación para el producto |
|---|---|
| **Ley 29733 + DS 016-2024-JUS** (vigente 31/03/2025) | El SaaS actúa como **encargado del tratamiento**; cada clínica es titular del banco de datos. Acuerdo de tratamiento firmado con cada tenant. Política de seguridad documentada y de fecha cierta. Procedimiento de notificación de incidentes a la ANPDP. Evaluar designación de Oficial de Datos Personales según volumen. Documentar transferencia internacional (hosting fuera de Perú). Atención de derechos ARCO (acceso, rectificación, cancelación, oposición). |
| **Ley anti-spam (vigente 10/05/2025, art. 58 Código del Consumidor)** | Prohibido SMS/correo/llamada promocional sin consentimiento previo, expreso e inequívoco. El sistema separa mensajes transaccionales de comerciales, exige consentimiento registrado para los segundos y ofrece opt-out (BAJA). La carga de la prueba es del proveedor: se guarda evidencia del consentimiento. |
| **SUNAT (facturación electrónica)** | Comprobantes vía OSE autorizado; series/correlativos por clínica; resumen diario de boletas; comunicación de baja; conservación de XML/CDR. |
| **SENASA / DIGEMID** | Trazabilidad de productos veterinarios y medicamentos controlados: lote, vencimiento, procedencia. |

> Los datos clínicos registrados pertenecen al animal; los datos personales son los del tutor (nombre, DNI, teléfono, dirección, correo). Se tratan como **datos personales no sensibles**, salvo confirmación legal en contrario. Validar con asesoría legal antes del lanzamiento.

## 3.2 Aislamiento multi-tenant (riesgo P0)

| # | Requisito | Verificación |
|---|---|---|
| S-01 | Todo modelo de negocio hereda de `TenantModel` con scope global | Test que recorre modelos y falla si falta el scope |
| S-02 | `tenant_id` se asigna en servidor; nunca se acepta del cliente | FormRequests lo rechazan como campo |
| S-03 | IDOR cross-tenant responde **404** | Test de integración con dos tenants |
| S-04 | Prefijo de cache y Redis por tenant | Middleware `ResolveTenant` |
| S-05 | Storage con ruta por tenant y URLs firmadas | Bucket privado; sin listado público |
| S-06 | Consultas raw (`DB::select`) prohibidas sin revisión | Regla de PHPStan/Semgrep personalizada |
| S-07 | Superadmin que "impersona" un tenant deja rastro en auditoría | Log obligatorio con motivo |

## 3.3 Autenticación y sesiones

| # | Requisito |
|---|---|
| A-01 | Contraseñas con **argon2id** (o bcrypt cost ≥ 12); nunca texto plano ni cifrado reversible |
| A-02 | Política mínima: 10 caracteres, verificación contra listas de contraseñas filtradas |
| A-03 | **MFA (TOTP)** obligatorio para Superadmin y Admin de clínica; opcional para el resto |
| A-04 | Bloqueo progresivo tras 5 intentos fallidos (15 min → 1 h); mensaje genérico (no enumeración de usuarios) |
| A-05 | Regeneración de ID de sesión al iniciar sesión y al cambiar privilegios |
| A-06 | Cookies `HttpOnly`, `Secure`, `SameSite=Lax`; sesión en Redis del lado servidor (no JWT en cliente) |
| A-07 | Expiración: inactividad 30 min (panel), absoluta 12 h; tutor PWA 30 días con renovación |
| A-08 | Vinculación de sesión a huella (IP/UA); cierre si cambia bruscamente |
| A-09 | "Cerrar todas las sesiones" con invalidación real en servidor |
| A-10 | Recuperación de contraseña por token de un solo uso, TTL 30 min, sin revelar existencia del correo |
| A-11 | Tokens de API con scopes, expiración y revocación; hash almacenado, nunca el valor |

## 3.4 Autorización (RBAC) — matriz mínima

| Recurso | Superadmin | Admin | Veterinario | Recepción | Auxiliar | Tutor |
|---|---|---|---|---|---|---|
| Configuración tenant | ✓ (auditado) | ✓ | – | – | – | – |
| Usuarios/roles | ✓ | ✓ | – | – | – | – |
| Historia clínica: leer | – | ✓ | ✓ | lectura básica | ✓ (hospitalizados) | propia |
| Historia clínica: escribir | – | – | ✓ | – | notas de enfermería | – |
| Recetas | – | – | ✓ | – | – | ver propias |
| Agenda | – | ✓ | propia | ✓ | propia | reservar/cancelar propias |
| Facturación / caja | – | ✓ | – | ✓ | – | ver propias |
| Inventario | – | ✓ | consumir | vender | consumir | – |
| Reportes | – | ✓ | propios | caja | – | – |
| Auditoría | ✓ | ✓ (su tenant) | – | – | – | – |
| Planes / cupos | ✓ | ver | – | – | – | – |

Reglas: **denegación por defecto**; toda acción pasa por una Policy; el rol se verifica en servidor en cada petición; el frontend solo oculta, nunca autoriza.

## 3.5 Gestión de secretos y credenciales

| # | Requisito |
|---|---|
| K-01 | Secretos solo en `.env` por servicio o gestor de secretos; nunca en código, imagen o repositorio |
| K-02 | `.env.example` por servicio con nombres y sin valores |
| K-03 | **Certificado digital SUNAT y credenciales OSE por clínica** cifrados en reposo (AES-256-GCM) con clave derivada de `APP_KEY` + sal por tenant; almacenados fuera del webroot; en `$hidden` y excluidos del reporting de excepciones |
| K-04 | Credencial FCM, tokens WhatsApp y claves de storage solo en backend |
| K-05 | Cada gateway SMS con credencial individual, rotable y revocable |
| K-06 | Rotación de `APP_KEY` planificada con re-cifrado |
| K-07 | Escaneo de secretos en CI (gitleaks) y pre-commit |

## 3.6 Criptografía y protección de datos

| # | Requisito |
|---|---|
| C-01 | TLS 1.2+ en todos los extremos; HSTS con `preload`; redirección HTTP→HTTPS |
| C-02 | Cifrado en reposo: volúmenes cifrados en el VPS; backups cifrados (age/GPG) |
| C-03 | Campos de alta sensibilidad cifrados a nivel de aplicación: certificados, tokens de integración |
| C-04 | Minimización: DNI del tutor almacenado solo si es requerido para factura; enmascarado en listados |
| C-05 | Datos de tarjeta **nunca** se almacenan ni transitan por el sistema (pagos con tarjeta se registran solo como medio de pago y referencia) |
| C-06 | Exportación de datos del tutor (derecho ARCO) y anonimización al cierre de un tenant tras el periodo de retención |

## 3.7 Mapa OWASP Top 10 (2021)

| Riesgo | Controles aplicados |
|---|---|
| A01 Control de acceso | RBAC + Policies, scope de tenant, IDOR→404, MFA en roles altos |
| A02 Fallos criptográficos | argon2id, TLS, cifrado de secretos, backups cifrados |
| A03 Inyección | Eloquent/parametrizadas, sin raw sin revisión, escape en Blade, validación lista blanca, CSP |
| A04 Diseño inseguro | Modelado de amenazas por módulo, rate limiting, errores sin detalle interno, cupos |
| A05 Configuración | Cabeceras (3.8), `APP_DEBUG=false`, sin puertos expuestos salvo 80/443, sin cuentas por defecto, ocultar `Server`/`X-Powered-By` |
| A06 Componentes vulnerables | SCA en CI, Dependabot/Renovate, imágenes escaneadas con Trivy, solo LTS |
| A07 Identificación/autenticación | Sección 3.3 completa |
| A08 Integridad de software/datos | Pipeline con pasos obligatorios, tags inmutables, SRI en scripts externos, firma de webhooks |
| A09 Registro y monitoreo | Logs JSON sin PII, auditoría append-only, alertas Zabbix, retención 12 meses |
| A10 SSRF | El servidor solo invoca destinos de una lista blanca (OSE, FCM, WhatsApp, storage); sin fetch de URLs del usuario |

## 3.8 Cabeceras, CORS, CSRF, XSS

- `Content-Security-Policy` estricta con nonces; sin `unsafe-inline` en producción.
- `X-Frame-Options: DENY` / `frame-ancestors 'none'`.
- `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` restrictiva.
- CORS explícito: solo orígenes propios (`app.dominio.pe`, PWA); nunca `*` con credenciales; métodos y cabeceras enumerados.
- Tokens CSRF en toda petición que cambia estado (Laravel por defecto); API pública con tokens Bearer y sin cookies.
- Sanitización de HTML en campos de texto libre (notas clínicas) con lista blanca de etiquetas.
- Sin source maps en producción; sin endpoints de debug; sin `console.log` con datos.

## 3.9 Token de vinculación por QR (PWA)

| # | Requisito |
|---|---|
| Q-01 | 128 bits de entropía criptográfica (`random_bytes(16)` → base64url) |
| Q-02 | Un solo uso; se invalida al canjearse |
| Q-03 | TTL 72 h; regenerable desde recepción |
| Q-04 | Rate limit en `/vincular`: 10 intentos/IP/hora; bloqueo tras abuso |
| Q-05 | Al canjear se emite sesión de tutor normal (cookie `HttpOnly`); el token nunca se reutiliza como sesión |
| Q-06 | Hash del token en base de datos, no el valor |
| Q-07 | Canje registrado en auditoría (IP, UA, fecha) |

## 3.10 Gateway SMS como dispositivo no confiable

| # | Requisito |
|---|---|
| G-01 | Credencial única por dispositivo; alcance limitado a: obtener su cola, reportar estado, enviar respuestas entrantes |
| G-02 | El gateway **nunca** consulta datos de pacientes ni tutores; recibe el texto ya renderizado |
| G-03 | Textos SMS sin DNI ni nombre completo (solo nombre de mascota y clínica) |
| G-04 | Comunicación por HTTPS con pin de certificado en la app si es viable |
| G-05 | Revocación inmediata desde panel Superadmin; dispositivo revocado no puede reconectar |
| G-06 | Heartbeat y salida de rotación automática |

## 3.11 Archivos y adjuntos clínicos

- Bucket privado; acceso solo por URLs firmadas de 5–15 min generadas por backend.
- Validación de tipo por contenido (magic bytes), no por extensión; límite de tamaño (20 MB imagen, 50 MB PDF).
- Nombres de objeto aleatorios; ruta por tenant/paciente.
- Escaneo antivirus (ClamAV) en subida.
- Sin ejecución ni servido directo desde el webroot.

## 3.12 Auditoría y registro

- Tabla `auditoria` **append-only** (sin `UPDATE`/`DELETE` para el usuario de aplicación): quién, qué, cuándo, desde dónde, tenant, antes/después en cambios sensibles.
- Eventos obligatorios: inicio/cierre de sesión, cambio de rol, **lectura de historia clínica**, emisión/anulación de comprobante, cambio de precios, exportación de datos, impersonación, canje de token QR, cambio de consentimiento.
- Logs de aplicación sin PII ni secretos; retención 12 meses; acceso restringido.

## 3.13 Backups y recuperación

- MySQL: dump diario cifrado + binlog para recuperación a punto en el tiempo; retención 30 días.
- Storage: versionado del bucket; retención 30 días.
- Copia fuera del proveedor principal (homelab como réplica cifrada).
- **Prueba de restore mensual** documentada (tiempo de recuperación objetivo: 4 h; punto de recuperación objetivo: 24 h, mejorable a 1 h con binlog).

## 3.14 Pruebas de seguridad

| Tipo | Herramienta sugerida | Frecuencia |
|---|---|---|
| SAST | Semgrep, Psalm/PHPStan security | Cada push |
| SCA | `composer audit`, `npm audit`, Trivy | Cada push + semanal |
| Secretos | gitleaks | Cada push |
| DAST | OWASP ZAP contra staging | Semanal |
| Tests de aislamiento | PHPUnit/Pest (suite dedicada) | Cada push |
| Pentest externo | Tercero | Antes del lanzamiento y anual |

## 3.15 Respuesta a incidentes

1. Detección (alerta Zabbix, reporte de usuario, hallazgo interno).
2. Contención: revocar credenciales, aislar tenant afectado, rotar secretos.
3. Evaluación de alcance con la auditoría.
4. Notificación a la ANPDP y a las clínicas afectadas según plazos del reglamento.
5. Post-mortem escrito y acciones correctivas.

## 3.16 Consentimiento y mensajería

- Registro `consentimientos`: tutor, tipo (transaccional/marketing), canal, fecha, origen (mostrador/PWA/web), IP, usuario que lo registró, texto mostrado.
- Opt-out por `BAJA` (SMS/WhatsApp), enlace en correo y ajuste en PWA; efecto inmediato.
- Los mensajes comerciales requieren consentimiento vigente; el sistema lo bloquea, no solo lo advierte.

---

# 4. Módulos funcionales

Cada módulo se describe con: objetivo, roles, funcionalidades (ítems), reglas de negocio, datos principales, seguridad específica y plan mínimo.

---

## M01 — Núcleo del tenant y configuración

**Objetivo:** representar a la clínica como tenant y centralizar su configuración.
**Roles:** Admin (edita), Superadmin (crea/suspende).
**Plan:** Gratis.

| Ítem | Funcionalidades |
|---|---|
| Datos de la clínica | Razón social, RUC, nombre comercial, logo, dirección, teléfono, correo, sitio web |
| Sucursales (multi-sede) | Alta de sucursales, horarios por sede, series de comprobante por sede, stock por sede |
| Horarios y feriados | Horario de atención por día, feriados nacionales precargados, cierres excepcionales |
| Catálogo de servicios | Servicio, categoría, precio, duración estimada, si requiere veterinario, IGV afecto/inafecto |
| Catálogo de especies/razas | Precargado (canino, felino, exóticos); ampliable |
| Parámetros clínicos | Esquemas de vacunación por especie, intervalos de desparasitación, unidades |
| Plantillas | Recetas, consentimientos informados, alta hospitalaria, carnet, boleta impresa |
| Personalización | Colores y logo en PWA y documentos; texto de pie de boleta |
| Preferencias de notificación | Canales activos, horas de envío, anticipación de recordatorios |

**Reglas:** el RUC se valida (formato y dígito verificador). Un tenant suspendido queda en solo lectura. Cambios de configuración fiscal quedan en auditoría.
**Datos:** `tenants`, `sucursales`, `servicios`, `especies`, `razas`, `plantillas`, `configuraciones`.
**Seguridad:** solo Admin; cambios auditados; RUC/series no editables por Recepción.

---

## M02 — Usuarios, roles y permisos

**Objetivo:** gestionar el personal de la clínica y su acceso.
**Roles:** Admin.
**Plan:** Gratis (1 usuario), Básico (2), Pro (5), Clínica (ilimitados).

| Ítem | Funcionalidades |
|---|---|
| Usuarios | Alta por invitación (correo), activar/desactivar, asignar sucursal, foto, firma para recetas |
| Roles | Admin, Veterinario, Recepción, Auxiliar; permisos granulares por módulo/acción |
| Perfil profesional | Colegiatura (CMVP), especialidad, horario, color en agenda |
| Seguridad de cuenta | Cambio de contraseña, MFA (obligatorio para Admin), sesiones activas, cerrar todas |
| Límite por plan | Bloqueo al superar usuarios del plan con aviso de upgrade |

**Reglas:** un usuario pertenece a un tenant; el correo es único por tenant. Desactivar no borra (conserva autoría en historia clínica).
**Seguridad:** sección 3.3 y 3.4 completa; invitaciones con token de un solo uso.

---

## M03 — Tutores (clientes)

**Objetivo:** ficha del dueño de la mascota, sus datos de contacto y consentimientos.
**Roles:** Recepción, Veterinario (lectura), Admin.
**Plan:** Gratis.

| Ítem | Funcionalidades |
|---|---|
| Ficha | Nombres, documento (DNI/CE/RUC), teléfono(s), correo, dirección, distrito, notas |
| Mascotas asociadas | Varias mascotas por tutor; tutor secundario/autorizado por mascota |
| Consentimientos | Transaccional (implícito al registrar contacto) y **marketing (explícito, con evidencia)**; opt-out |
| Historial | Citas, comprobantes, saldo/deuda, mensajes enviados |
| Búsqueda | Por nombre, documento, teléfono, mascota; resultados paginados |
| Fusión de duplicados | Unir fichas duplicadas manteniendo historial |
| Vinculación PWA | Generar/regenerar token QR; ver si tiene PWA instalada |
| Derechos ARCO | Exportar datos del tutor; solicitud de cancelación con flujo de anonimización |

**Reglas:** documento validado por tipo. Teléfono normalizado a E.164 (+51…). No se puede eliminar un tutor con comprobantes; se anonimiza.
**Datos:** `tutores`, `tutor_mascota`, `consentimientos`, `tokens_vinculacion`.
**Seguridad:** DNI enmascarado en listados; exportación auditada; búsqueda con rate limit.

---

## M04 — Pacientes (mascotas)

**Objetivo:** ficha del animal.
**Roles:** Recepción, Veterinario, Auxiliar.
**Plan:** Gratis.

| Ítem | Funcionalidades |
|---|---|
| Ficha | Nombre, especie, raza, sexo, esterilizado, fecha de nacimiento/edad estimada, color, señas, microchip, foto |
| Estado | Activo, fallecido (con fecha), transferido, perdido |
| Alertas clínicas | Alergias, agresividad, enfermedades crónicas, medicación permanente — visibles en cabecera siempre |
| Curva de peso | Registro por visita, gráfico, alertas de variación |
| Resumen | Última visita, próximas vacunas, próximos controles, saldo |
| Documentos | Carnet electrónico (M06), certificados de salud, certificado de viaje |

**Reglas:** un paciente tiene un tutor principal obligatorio. Fallecido bloquea agenda y recordatorios automáticamente.
**Datos:** `pacientes`, `pesos`, `alertas_clinicas`.

---

## M05 — Historia clínica electrónica (HCE)

**Objetivo:** registro clínico completo y trazable.
**Roles:** Veterinario (escribe), Admin (lee), Auxiliar (notas de enfermería), Tutor (lee resumen propio).
**Plan:** Gratis.

| Ítem | Funcionalidades |
|---|---|
| Consulta (SOAP) | Motivo, anamnesis, examen físico (temperatura, FC, FR, mucosas, peso), diagnóstico presuntivo/definitivo, plan |
| Diagnósticos | Catálogo codificable (propio o VeNom), diagnósticos activos/resueltos por paciente |
| Tratamientos | Medicamento, dosis, vía, frecuencia, duración; descuenta de inventario si se aplica en clínica |
| Recetas | Generación PDF con firma del veterinario y colegiatura; entrega al tutor por PWA |
| Vacunación | Registro con lote, laboratorio, vencimiento, próxima dosis según esquema; genera recordatorio |
| Desparasitación | Interna/externa, producto, peso, próxima dosis |
| Procedimientos y cirugías | Tipo, anestesia, consentimiento informado firmado, informe operatorio, complicaciones |
| Órdenes | Laboratorio, imágenes, interconsulta (enlaza con M13) |
| Adjuntos | Imágenes, PDF, videos cortos; vista previa; URLs firmadas |
| Notas de evolución | Cronológicas; notas de enfermería por Auxiliar |
| Firma y cierre | Consulta cerrada = inmutable; corrección solo por adenda con motivo |
| Línea de tiempo | Vista unificada de todos los eventos del paciente |
| Certificados | Salud, vacunación, viaje (plantillas) |
| Plantillas de consulta | Por tipo (control, vacuna, urgencia) para acelerar registro |

**Reglas:** toda entrada tiene autor y fecha; el veterinario no puede editar entradas de otro. Cierre de consulta obligatorio para facturar sus servicios (configurable).
**Datos:** `consultas`, `diagnosticos`, `tratamientos`, `recetas`, `vacunaciones`, `desparasitaciones`, `procedimientos`, `adjuntos`, `notas`.
**Seguridad:** **toda lectura de HCE se audita**; adjuntos con URLs firmadas; sanitización de texto libre; consultas cerradas inmutables.

---

## M06 — Carnet electrónico de la mascota

**Objetivo:** carnet vivo dentro de la sección del tutor en la PWA, que se actualiza automáticamente con cada acto clínico registrado en la HCE (M05). No es un documento generado: es una vista dinámica de datos reales; el PDF es solo una exportación.
**Roles:** Veterinario (alimenta vía HCE), Tutor (consulta en PWA), Recepción (imprime).
**Plan:** Gratis.

| Ítem | Funcionalidades |
|---|---|
| Datos de la mascota | Foto, nombre, especie, raza, sexo, esterilizado, fecha de nacimiento y edad calculada, color/señas, microchip, estado; se refleja cualquier cambio hecho en M04 |
| Peso | Último peso registrado con fecha, curva de peso (gráfico) y variación respecto al anterior; se alimenta de cada consulta |
| Vacunas aplicadas | Tabla cronológica: vacuna, fecha, lote, laboratorio, veterinario que aplicó; aparece en el momento en que se registra en M05 |
| Próximas vacunas | Calculadas por el esquema de la especie (M01) y la última dosis: fecha, estado (al día / próxima / vencida) con semáforo |
| Desparasitaciones | Internas y externas: producto, fecha, próxima dosis; mismo semáforo |
| Alertas clínicas | Alergias, enfermedades crónicas y medicación permanente visibles en la cabecera del carnet |
| Datos del tutor | Nombre y teléfono de contacto (solo visibles para el tutor autenticado, nunca en la vista pública) |
| Datos de la clínica | Nombre, dirección, teléfono, veterinario responsable; multi-clínica: se muestra qué clínica aplicó cada vacuna |
| Actualización | En tiempo real vía push: al registrarse una vacuna o peso el tutor recibe notificación y el carnet ya refleja el dato |
| Compartir / verificar | Enlace público de solo lectura con token y QR, para guarderías, viajes o adopción; el tutor puede revocarlo |
| Exportación | PDF con el mismo contenido, firma/sello del veterinario y QR de verificación; formato A5 y tarjeta; impresión desde recepción |
| QR de vinculación PWA | Impreso en la versión física para vincular al tutor (M15) |
| Historial de versiones | Cada exportación queda fechada; el PDF indica "válido al {fecha}" |

**Reglas:** el carnet nunca se edita directamente; solo cambia cuando cambia la HCE, la ficha del paciente o el peso. Una vacuna registrada con lote vencido o sin lote muestra advertencia al veterinario antes de guardar. Paciente fallecido conserva el carnet en modo solo lectura.
**Datos:** vista sobre `pacientes`, `pesos`, `vacunaciones`, `desparasitaciones`, `alertas_clinicas`, `esquemas_vacunacion`; `enlaces_carnet` (tokens públicos).
**Seguridad:** la vista pública expone solo especie, nombre de mascota, vacunas, desparasitaciones y peso; **sin datos del tutor**. El tutor solo ve carnets de mascotas donde figura como tutor. Enlaces públicos con token de 128 bits, revocables, con rate limit. El PDF se sirve por URL firmada.

---

## M07 — Agenda y citas

**Objetivo:** gestionar la ocupación de profesionales y salas.
**Roles:** Recepción, Veterinario, Admin, Tutor (reserva online).
**Plan:** Gratis (agenda), Pro (reserva online, lista de espera).

| Ítem | Funcionalidades |
|---|---|
| Vista | Día/semana/mes; por profesional, sala o sede; colores por tipo de servicio |
| Cita | Paciente, tutor, servicio, profesional, sala, duración, notas; recurrentes |
| Estados | Reservada → confirmada → en sala de espera → en atención → atendida / no asistió / cancelada |
| Confirmación | Recordatorio previo; confirmación por respuesta `SI` (SMS/WhatsApp) o desde PWA |
| Sala de espera | Tablero en tiempo real (Reverb) con tiempos de espera |
| Reserva online | Página pública por clínica; el tutor elige servicio/profesional/horario; requiere PWA o datos mínimos |
| Lista de espera | Cupos liberados se ofrecen automáticamente |
| Bloqueos | Vacaciones, capacitación, mantenimiento de sala |
| Ausentismo | Registro de "no asistió"; métrica por tutor |

**Reglas:** no doble reserva por profesional/sala. Cancelación por tutor solo hasta X horas antes (configurable).
**Datos:** `citas`, `salas`, `bloqueos`, `lista_espera`.
**Seguridad:** reserva online con rate limit y CAPTCHA; tutor solo ve/modifica sus citas.

---

## M08 — Recordatorios y notificaciones

**Objetivo:** motor de mensajería multicanal con costo mínimo y cumplimiento legal.
**Roles:** Admin (configura), Recepción (envía manuales), sistema (automáticos).
**Plan:** Gratis (modo fijo), Básico/Pro (configurable), Clínica (configurable + campañas).

| Ítem | Funcionalidades |
|---|---|
| Recordatorios automáticos | Eventos disponibles: **próxima vacuna**, **cita** (24 h y 2 h antes), desparasitación, control post-consulta, resultado disponible, alta hospitalaria, listo para recoger (grooming), cumpleaños de mascota. Qué eventos están activos y con qué texto depende del nivel (ver tabla de niveles) |
| Niveles de mensajería | **Modo fijo (Gratis):** solo dos disparadores —próxima vacuna y cita— con plantilla **predefinida por el Superadmin**, no editable; la clínica solo activa/desactiva el módulo. **Modo configurable (Básico/Pro):** la clínica elige qué eventos disparan mensaje, con qué anticipación y por qué canal, y **edita el texto** de cada plantilla. **Modo completo (Clínica):** todo lo anterior + campañas comerciales segmentadas |
| Escalera de canales | Push → SMS → WhatsApp según disponibilidad y tipo (sección 2.6) |
| Tipos de mensaje | Transaccional (siempre permitido) vs comercial (requiere consentimiento) |
| Plantillas | Variables permitidas (`{mascota}`, `{clinica}`, `{fecha}`, `{hora}`, `{vacuna}`); vista previa; contador de caracteres SMS; el pie "Responde BAJA…" y el nombre de la clínica son fijos y no se pueden quitar. En plan Gratis las plantillas son las globales del Superadmin (M18) |
| Editor por evento (Básico+) | Pantalla "Mensajes" con un renglón por evento: activo sí/no, anticipación (p. ej. 7 y 1 días antes de la vacuna), canal preferido, texto; se puede restaurar la plantilla por defecto |
| Cupos | Push ilimitado en todos los planes. SMS/WhatsApp con cupo mensual por plan (Gratis: cupo pequeño solo SMS, sin WhatsApp); alerta al 80 %; compra de cupo extra (planes pagos); al agotar se sigue enviando solo por push |
| Outbox | Estado por mensaje, canal, costo, reintentos, error |
| Respuestas entrantes | `SI` confirma cita, `NO` cancela, `BAJA` opt-out; otros van a bandeja de recepción |
| Campañas comerciales | Segmentación (especie, edad, última visita); solo a tutores con consentimiento; límite diario |
| Panel de la clínica | % tutores con push, mensajes por canal, ahorro estimado, tasa de entrega |
| Horario de envío | Ventana permitida (p. ej. 08:00–20:00); nada en madrugada |

**Reglas:** no se envía a pacientes fallecidos ni a tutores con opt-out. Un evento genera máximo un recordatorio (idempotencia). Los comerciales sin consentimiento se bloquean, no se advierten.
**Datos:** `notificaciones`, `plantillas_mensaje`, `cupos`, `respuestas_entrantes`, `campanas`.
**Seguridad:** textos sin DNI; consentimiento verificado en servidor; auditoría de campañas; rate limit por tenant.

---

## M09 — Facturación electrónica SUNAT

**Objetivo:** emitir comprobantes válidos integrados con la atención y el POS.
**Roles:** Recepción/Caja, Admin.
**Plan:** Básico.

| Ítem | Funcionalidades |
|---|---|
| Comprobantes | Boleta, factura, nota de crédito, nota de débito; ticket interno para plan Gratis |
| Series y correlativos | Por sede y tipo; correlativo atómico (lock Redis + `SELECT … FOR UPDATE`) |
| Emisión | Desde consulta, cita o venta POS; ítems de servicio y producto; IGV por ítem; descuentos; ICBPER si aplica |
| Cliente | Boleta con/sin DNI; factura exige RUC validado; consulta de RUC/DNI opcional |
| Envío a OSE | Asíncrono; estados `pendiente → enviado → aceptado / rechazado / observado`; reintentos |
| Documentos | XML firmado, CDR, PDF con logo y QR; envío al tutor por PWA/correo |
| Anulación | Comunicación de baja dentro del plazo; nota de crédito fuera del plazo |
| Resumen diario | Automático para boletas; reenvío ante error |
| Conciliación | Panel de comprobantes rechazados con motivo y acción |
| Reportes | Ventas por periodo, por tipo, por vendedor; exportación para contabilidad (Excel/PLE simplificado) |
| Ambiente | Demo (pruebas del tenant al activar) y producción |

**Reglas:** un comprobante aceptado es inmutable. No se emite factura sin RUC válido. Cambios de serie auditados.
**Datos:** `comprobantes`, `comprobante_items`, `series`, `envios_ose`, `resumenes_diarios`.
**Seguridad:** certificado por tenant cifrado (K-03); cola `sunat` aislada; idempotencia por `comprobante_id`; PDF servido por URL firmada.

---

## M10 — POS y caja

**Objetivo:** ventas rápidas de mostrador y control de efectivo.
**Roles:** Recepción/Caja, Admin.
**Plan:** Básico.

| Ítem | Funcionalidades |
|---|---|
| Venta | Búsqueda rápida por nombre/código de barras; servicios y productos; carrito; descuentos con permiso |
| Medios de pago | Efectivo, Yape, Plin, tarjeta (referencia), transferencia; pago mixto |
| Cuentas por cobrar | Venta a crédito con límite por tutor; abonos; estado de cuenta |
| Caja | Apertura con monto inicial, movimientos (ingresos/egresos), cierre con arqueo y diferencias |
| Turnos | Caja por usuario/turno; cierre obligatorio para abrir otro |
| Impresión | Ticket térmico 80 mm; boleta/factura A4 |
| Devoluciones | Con nota de crédito o vale |
| Reporte de caja | Por turno, por medio de pago, por usuario |

**Reglas:** no se vende con caja cerrada. Descuentos por encima del umbral requieren Admin. Diferencias de arqueo quedan registradas.
**Datos:** `cajas`, `movimientos_caja`, `ventas`, `pagos`, `cuentas_por_cobrar`.
**Seguridad:** nunca se almacenan datos de tarjeta; anulaciones auditadas; permiso explícito para descuentos.

---

## M11 — Inventario y farmacia

**Objetivo:** control de stock con trazabilidad sanitaria.
**Roles:** Admin, Recepción (vender), Veterinario/Auxiliar (consumir).
**Plan:** Básico.

| Ítem | Funcionalidades |
|---|---|
| Productos | Código, nombre, categoría, presentación, unidad, precio compra/venta, IGV, stock mínimo, foto |
| Lotes y vencimientos | Ingreso por lote con fecha de vencimiento; salida FEFO (primero en vencer); alertas 60/30/7 días |
| Kardex | Movimientos: compra, venta, consumo clínico, ajuste, merma, transferencia entre sedes |
| Compras y proveedores | Órdenes de compra, recepción parcial, proveedor, costo promedio |
| Consumo clínico | Descuento automático al aplicar tratamiento/vacuna en HCE |
| Fraccionamiento | Venta por unidad de una presentación (ej. tabletas de una caja) |
| Controlados | Marcado de productos DIGEMID/SENASA controlados; registro de quién dispensó y a qué paciente |
| Inventario físico | Conteo cíclico y ajuste con motivo |
| Alertas | Stock mínimo, vencimientos, productos sin movimiento |
| Reportes | Valorización, rotación, mermas, vencidos, por proveedor |

**Reglas:** no se permite stock negativo (configurable). Ajustes con motivo obligatorio. Lote obligatorio para vacunas y controlados.
**Datos:** `productos`, `lotes`, `movimientos_inventario`, `proveedores`, `ordenes_compra`.
**Seguridad:** ajustes y mermas auditados; consumo de controlados trazado a paciente y usuario.

---

## M12 — Hospitalización

**Objetivo:** seguimiento de pacientes internados.
**Roles:** Veterinario, Auxiliar, Admin.
**Plan:** Pro.

| Ítem | Funcionalidades |
|---|---|
| Ingreso | Motivo, diagnóstico, jaula/área, veterinario responsable, presupuesto y anticipo |
| Hoja de control | Tratamientos programados por hora (medicación, fluidos, alimentación, paseo), marcados por Auxiliar |
| Signos vitales | Registro periódico; gráficos; alertas fuera de rango |
| Notas de evolución | Diarias por veterinario; notas de enfermería |
| Consumos | Todo insumo/medicamento se carga a la cuenta del internamiento |
| Tablero | Vista de jaulas/áreas en tiempo real; pendientes de la hora |
| Alta | Informe de alta, indicaciones al tutor (PWA), liquidación y comprobante |
| Comunicación | Actualización opcional al tutor (push) con foto/estado |

**Reglas:** tarea vencida sin marcar genera alerta interna. Alta cierra la cuenta y libera la jaula.
**Datos:** `internamientos`, `tareas_internamiento`, `signos_vitales`, `jaulas`.
**Seguridad:** Auxiliar solo ve internados de su sede; fotos al tutor por URL firmada.

---

## M13 — Laboratorio e imágenes

**Objetivo:** gestionar órdenes y resultados de exámenes.
**Roles:** Veterinario, Auxiliar, Admin.
**Plan:** Pro.

| Ítem | Funcionalidades |
|---|---|
| Catálogo de exámenes | Propios y externos (laboratorio de referencia), precio, tiempo estimado |
| Órdenes | Desde HCE; estado `solicitado → tomado → en proceso → resultado → informado` |
| Resultados | Carga manual con valores de referencia por especie; adjunto PDF/imagen |
| Alertas | Valores fuera de rango resaltados |
| Imágenes | Radiografía, ecografía; visor básico; anotaciones |
| Entrega al tutor | Resultado disponible en PWA con notificación |
| Integración futura | Importación de resultados por archivo/API de analizadores |

**Datos:** `examenes`, `ordenes_examen`, `resultados`, `adjuntos`.
**Seguridad:** resultados solo visibles al tutor tras validación del veterinario.

---

## M14 — Peluquería y grooming

**Objetivo:** agenda y servicios estéticos.
**Roles:** Recepción, Groomer, Admin.
**Plan:** Pro.

| Ítem | Funcionalidades |
|---|---|
| Agenda independiente | Por groomer; duración por tamaño/raza |
| Ficha de grooming | Preferencias, comportamiento, alergias a productos, fotos antes/después |
| Paquetes | Baño, corte, uñas, oídos; combos y precios por tamaño |
| Estado | Recibido → en proceso → listo para recoger (push al tutor) |
| Ventas | Integrado con POS |

---

## M15 — Portal / PWA del tutor

**Objetivo:** canal de bajo costo con el dueño de la mascota.
**Roles:** Tutor.
**Plan:** Gratis (básico), Clínica (completo).

| Ítem | Funcionalidades |
|---|---|
| Vinculación | Por QR (token de un solo uso); vincular más mascotas con nuevos QR |
| Instalación | Prompt en Android; tutorial para iOS; detección de instalación |
| Mis mascotas | Ficha resumida, foto, próximas vacunas, peso |
| Historial | Consultas cerradas (resumen), recetas, resultados validados |
| Carnet electrónico | Vista viva por mascota (M06): datos, peso, vacunas y desparasitaciones al día, semáforo de próximas dosis, compartir por enlace, exportar PDF |
| Citas | Ver, confirmar, cancelar, reservar online (plan Pro+) |
| Notificaciones | Push; centro de notificaciones; preferencias de canal y opt-out |
| Comprobantes | Descargar boletas/facturas |
| Pagos (futuro) | Enlace de pago Yape/Plin/pasarela |
| Contacto | Botón WhatsApp/llamada a la clínica; dirección y horarios |
| Cuenta | Datos de contacto, consentimientos, exportar mis datos, eliminar cuenta |
| Multi-clínica | Un tutor puede estar vinculado a varias clínicas (tenants) con separación estricta |

**Seguridad:** cookie `HttpOnly`; sin datos clínicos en cache del Service Worker; el tutor solo accede a mascotas donde figura como tutor; rate limit en todas las rutas públicas.

---

## M16 — Reportes y dashboard

**Objetivo:** visibilidad operativa y financiera.
**Roles:** Admin (todo), Veterinario (propios), Caja (caja).
**Plan:** Básico (básicos), Pro (completos), Clínica (BI y exportaciones).

| Ítem | Funcionalidades |
|---|---|
| Dashboard | Ingresos del día/mes, citas de hoy, pacientes nuevos, alertas de stock, cupo de mensajería |
| Ventas | Por periodo, servicio, producto, vendedor, medio de pago, sede |
| Clínicos | Consultas por veterinario, diagnósticos frecuentes, vacunas aplicadas, cirugías |
| Agenda | Ocupación, ausentismo, tiempos de espera |
| Clientes | Nuevos vs recurrentes, inactivos (> 6 meses), valor por tutor |
| Inventario | Valorización, rotación, vencimientos |
| Mensajería | Entregas por canal, adopción de PWA, ahorro estimado |
| Exportación | Excel/CSV; programación de envío por correo (n8n) |
| Comparativo multi-sede | Solo plan Clínica |

**Reglas:** reportes pesados se generan en cola `reports` y se notifican al terminar.
**Seguridad:** exportaciones auditadas; datos personales minimizados en exportaciones.

---

## M17 — Auditoría y cumplimiento

**Objetivo:** trazabilidad y evidencia legal.
**Roles:** Admin (su tenant), Superadmin.
**Plan:** Básico.

| Ítem | Funcionalidades |
|---|---|
| Registro de auditoría | Consulta filtrable por usuario, acción, recurso, fecha; solo lectura |
| Accesos a HCE | Quién vio qué historia y cuándo |
| Consentimientos | Registro y evidencia por tutor; historial de cambios |
| Sesiones | Sesiones activas por usuario; cierre remoto |
| Derechos ARCO | Bandeja de solicitudes; exportación y anonimización con plazo |
| Acuerdo de tratamiento | Aceptación digital en onboarding, versión y fecha |
| Política de seguridad | Documento vigente descargable |
| Incidentes | Registro interno de incidentes y notificaciones realizadas |

**Seguridad:** tabla append-only; el usuario de aplicación no tiene `DELETE` sobre ella.

---

## M18 — Administración del SaaS (Superadmin)

**Objetivo:** operar el negocio multi-tenant.
**Roles:** Superadmin.

| Ítem | Funcionalidades |
|---|---|
| Tenants | Listado de espacios creados por autoservicio (M20); plan, estado (gratis/activo/moroso/suspendido), sedes, usuarios, uso; suspensión y reactivación manual |
| Planes y precios | Definición de planes, límites, cupos, precios; cambios de plan con prorrateo |
| Suscripciones | Vista de todas las suscripciones (M20): cobros, fallos de pago, morosidad, comprobantes emitidos, descuentos y cupones, cambios manuales con motivo |
| Mensajería | Cupos globales, costo real por canal, consumo por tenant, margen |
| Gateways SMS | Registro de dispositivos, credenciales, estado, revocación, límites, métricas |
| OSE | Estado de la cuenta multiempresa, consumo de comprobantes, errores por tenant |
| Onboarding | Seguimiento del checklist de cada tenant (M20): verificación de correo, acuerdo de tratamiento, certificado SUNAT, series, migración; clínicas estancadas para contacto |
| Antiabuso de registro | Registros por IP/correo, dominios bloqueados, espacios sin actividad a purgar, límites de espacios por persona |
| Migración de datos | Importadores CSV/Excel (tutores, pacientes, vacunas, productos) con validación |
| Impersonación | Acceso a un tenant con motivo obligatorio y auditoría |
| Salud del sistema | Colas, jobs fallidos, latencias, backups, alertas |
| Plantillas globales | Textos predefinidos de los mensajes del plan Gratis (próxima vacuna, cita) y valores por defecto para los demás planes; versionadas; vista previa por canal; cambio con aviso a las clínicas |
| Comunicados | Avisos en panel a todos los tenants (mantenimientos, novedades) |

**Seguridad:** MFA obligatorio; IP permitidas opcional; toda acción auditada; impersonación con expiración.

---

## M19 — Integraciones y API

**Objetivo:** extensibilidad para clínicas grandes y automatizaciones.
**Roles:** Admin (plan Clínica), Superadmin.
**Plan:** Clínica.

| Ítem | Funcionalidades |
|---|---|
| API REST | Recursos: tutores, pacientes, citas, comprobantes (lectura), inventario (lectura) |
| Tokens | Por tenant, con scopes, expiración, revocación; rate limit por token |
| Webhooks salientes | Eventos: cita creada/confirmada, comprobante aceptado, vacuna registrada; firma HMAC |
| n8n | Plantillas de automatización (reporte semanal, aviso de vencimientos, sincronización a hoja) |
| Importación/exportación | Formatos estándar; respaldo completo del tenant bajo demanda |

**Seguridad:** lista blanca de destinos de webhook; sin SSRF (no se invocan URLs arbitrarias sin validación); tokens con hash en base de datos.

---

## M20 — Registro, onboarding y suscripción (autoservicio)

**Objetivo:** que una clínica se registre sola, tenga su espacio operativo en el plan Gratis en menos de 2 minutos y pueda subir de plan cuando quiera desde una sección específica de su panel, sin intervención del proveedor.
**Roles:** Visitante (registro), Admin de clínica (plan y pagos), Superadmin (supervisión, M18).
**Plan:** transversal.

### Registro público

| Ítem | Funcionalidades |
|---|---|
| Formulario mínimo | Nombre de la clínica, nombre del responsable, correo, contraseña, teléfono. **No se pide RUC** en el registro; se pide al activar SUNAT |
| Verificación de correo | Enlace de un solo uso, TTL 24 h; hasta verificar, el espacio funciona en modo limitado (sin invitar usuarios ni exportar) |
| Aceptación de términos | Términos de servicio, política de privacidad y **acuerdo de tratamiento de datos** (versionado, con fecha e IP) |
| Creación del espacio | Provisión automática del tenant: subdominio (`slug.app.dominio.pe`) validado y único, usuario Admin, configuración por defecto, catálogos precargados (especies, razas, esquemas de vacunación, servicios de ejemplo) |
| Datos de ejemplo | Opción "cargar datos de demostración" (2 tutores, 3 mascotas, citas) borrable con un clic |
| Primer inicio | Asistente de 5 pasos: logo y datos básicos, horario, servicios y precios, invitar equipo, instalar PWA de prueba |
| Ingreso | Correo + contraseña; opción de "Iniciar con Google" (OAuth) para reducir fricción |
| Recuperación | Recuperación de contraseña y de acceso al espacio (por correo verificado) |

### Sección "Mi plan" (dentro del panel de la clínica)

| Ítem | Funcionalidades |
|---|---|
| Plan actual | Plan, fecha de renovación, uso vs límites (usuarios, mensajes, sedes, almacenamiento) con barras de consumo |
| Comparador | Tabla de planes con precios mensual/anual y diferencias; resalta lo que desbloquea el upgrade |
| Subir de plan | Selección de plan y periodicidad; cobro inmediato con prorrateo; activación instantánea de módulos |
| Bajar de plan | Permitido al final del periodo pagado; validación de límites (si excede usuarios/sedes debe reducir antes); los datos de módulos desactivados se conservan en solo lectura |
| Medios de pago | Tarjeta vía pasarela con tokenización (Culqi / Mercado Pago / Izipay / Niubiz); Yape/Plin por enlace de pago; transferencia con confirmación manual (plan anual) |
| Cobro recurrente | Suscripción mensual/anual con tarjeta tokenizada; reintentos ante fallo (día 1, 3, 7); avisos por correo y en panel |
| Comprobante | Boleta/factura electrónica emitida al tenant por cada cobro (con RUC si lo registra); historial descargable |
| Cupos extra | Compra de paquetes de mensajería y almacenamiento adicional desde la misma sección |
| Cupones | Código promocional en el checkout (descuento %, meses gratis) con validez y usos limitados |
| Morosidad | Periodo de gracia 7 días con aviso → modo solo lectura → suspensión a los 30 días → purga a los 90 (con aviso y exportación previa) |
| Cancelación | Autoservicio: baja al final del periodo, exportación completa de datos, confirmación por correo, anonimización tras retención |
| Cambio de titular | Transferir la propiedad del espacio a otro usuario Admin (con verificación) |

### Reglas de negocio

- El plan Gratis es permanente (no es prueba con vencimiento); sus límites se aplican por sistema, no por confianza.
- En plan Gratis la mensajería funciona en **modo fijo**: solo próxima vacuna y cita, con el texto global del Superadmin; ni el evento ni el texto son editables. Para mensajes propios o más eventos se sube de plan.
- Un correo puede ser Admin de varios espacios (cadenas), pero el registro de espacios nuevos se limita (3 por correo verificado) para frenar abuso.
- Al subir de plan los módulos se activan al instante; al bajar, nunca se borran datos.
- Un espacio Gratis sin actividad en 12 meses recibe avisos y luego se archiva (exportación disponible).
- El acuerdo de tratamiento aceptado en el registro es la base legal como encargado; cambios de versión requieren re-aceptación por el Admin.

**Datos:** `tenants` (estado, plan, slug), `suscripciones`, `pagos_suscripcion`, `cupones`, `aceptaciones_legales`, `verificaciones_correo`.

**Seguridad específica:**

| # | Requisito |
|---|---|
| R-01 | Registro con CAPTCHA y rate limit por IP (5 registros/hora) y por correo |
| R-02 | Bloqueo de correos desechables y validación de dominio MX |
| R-03 | Slug de subdominio validado (lista negra: `admin`, `api`, `www`, `sunat`, marcas), sin caracteres que permitan homógrafos |
| R-04 | Provisión de tenant idempotente y transaccional; un fallo a medio camino no deja espacios huérfanos |
| R-05 | Verificación de correo obligatoria para invitar usuarios, exportar datos o activar SUNAT |
| R-06 | **Nunca** se almacenan ni transitan datos de tarjeta por el sistema; solo el token de la pasarela y los últimos 4 dígitos |
| R-07 | Webhooks de la pasarela verificados por firma; el estado de la suscripción solo cambia por webhook confirmado o por Superadmin auditado |
| R-08 | Cambios de plan, cancelaciones y transferencias de titularidad auditados y confirmados por correo |
| R-09 | Los límites de plan se aplican en servidor (Policies + middleware), nunca solo ocultando botones |
| R-10 | OAuth (Google) solo para identidad; no se solicitan permisos adicionales |
| R-11 | Espacios purgados: borrado verificable de datos y de objetos en storage, con registro de la fecha |

---

# 5. Matriz módulos × planes

| Módulo | Gratis | Básico | Pro | Clínica |
|---|---|---|---|---|
| M01 Núcleo/configuración | ✓ | ✓ | ✓ | ✓ multi-sede |
| M02 Usuarios/roles | 1 | 2 | 5 | ∞ |
| M03 Tutores | ✓ | ✓ | ✓ | ✓ |
| M04 Pacientes | ✓ | ✓ | ✓ | ✓ |
| M05 HCE | ✓ | ✓ | ✓ | ✓ |
| M06 Carnet electrónico | ✓ | ✓ | ✓ | ✓ |
| M07 Agenda | básica | básica | + online, lista de espera | ✓ |
| M08 Recordatorios | modo fijo: vacuna + cita, texto predefinido; push ∞, SMS 50 | configurable: elige eventos y edita texto; SMS/WA 300 | configurable; SMS/WA 1,000 | completo + campañas; SMS/WA 3,000 |
| M09 SUNAT | – | ✓ | ✓ | ✓ |
| M10 POS/caja | – | ✓ | ✓ | ✓ |
| M11 Inventario | – | ✓ | ✓ | ✓ multi-sede |
| M12 Hospitalización | – | – | ✓ | ✓ |
| M13 Laboratorio/imágenes | – | – | ✓ | ✓ |
| M14 Grooming | – | – | ✓ | ✓ |
| M15 PWA tutor | básica | básica | + reserva | completa |
| M16 Reportes | – | básicos | completos | + BI, comparativo |
| M17 Auditoría | – | ✓ | ✓ | ✓ |
| M19 API/webhooks | – | – | – | ✓ |
| M20 Registro / Mi plan | ✓ | ✓ | ✓ | ✓ |

---

# 6. Roadmap sugerido

| Fase | Alcance | Objetivo |
|---|---|---|
| **0 — Fundaciones (4 sem.)** | Docker, tenancy, auth+MFA, RBAC, auditoría, CI/CD con SAST/SCA, backups | Base segura antes de cualquier módulo |
| **1 — MVP clínico (6 sem.)** | M20 registro autoservicio, M01–M07, M15 básica | Plan Gratis usable por autoservicio; primeras clínicas piloto |
| **2 — MVP comercial (6 sem.)** | M20 "Mi plan" con pasarela, M09, M10, M11, M08 con push+SMS, M16 básicos, M17 | Plan Básico vendible por autoservicio |
| **3 — Pro (8 sem.)** | M12, M13, M14, reserva online, WhatsApp, reportes completos | Plan Pro |
| **4 — Clínica (6 sem.)** | Multi-sede, BI, M19, PWA completa | Plan Clínica |
| Transversal | Pentest antes de fase 2; asesoría legal (acuerdo de tratamiento, consentimientos) antes de fase 2 | — |

---

# 7. Riesgos residuales

| Riesgo | Mitigación | Estado |
|---|---|---|
| Operador móvil suspende líneas del gateway SMS | Pool de gateways, topes diarios conservadores, WhatsApp como respaldo | Abierto — validar umbral real |
| Baja adopción de PWA (especialmente iOS) | QR en boleta/carnet, tutorial iOS, métrica visible a la clínica, SMS como respaldo | Abierto — medir en piloto |
| Cambios normativos SUNAT (validaciones 2026–2027) | OSE actualiza reglas; ambiente demo para probar; cola aislada con reintentos | Mitigado parcialmente |
| Clasificación legal de datos (sensibles vs no) | Asesoría legal previa; diseño ya minimiza y cifra | Abierto |
| Transferencia internacional (hosting fuera de Perú) | Documentar en política y acuerdo de tratamiento; alternativa de hosting local | Abierto — decisión documentada |
| Equipo de 1–2 personas: soporte y operación | Setup cobrado, plan anual, n8n para automatizar onboarding, runbooks | Mitigado parcialmente |
| Dependencia de un solo VPS | Backups probados, réplica en homelab, plan de restauración en 4 h | Mitigado |
| Compromiso de certificado SUNAT de un tenant | Cifrado por tenant, sin logs, rotación y revocación con la clínica | Mitigado |
| XSS en notas clínicas de texto libre | Sanitización lista blanca + CSP con nonces | Mitigado |
| Abuso de reserva online / vinculación QR | Rate limit, CAPTCHA, tokens de un solo uso | Mitigado |
| Abuso del registro gratuito (espacios basura, spam, uso del SMS gratuito) | CAPTCHA, rate limit, verificación de correo; en Gratis el texto es fijo y solo se dispara por vacuna/cita reales, cupo SMS mínimo y sin WhatsApp; purga de inactivos | Mitigado |
| Fallos de cobro recurrente / disputas con la pasarela | Reintentos, periodo de gracia, solo lectura antes de suspender, webhooks firmados | Mitigado parcialmente |

> Ningún sistema es 100 % seguro. El objetivo es reducir la superficie de ataque al mínimo, detectar rápido y poder demostrar diligencia ante la ANPDP y las clínicas.
