# Doel Taller 0.1.72 · Dossier para la revisión de seguridad externa («hacker review»)

Fecha: 15/10/2026 · Preparado por la auditoría interna (`docs/AUDITORIA_0.1.72.md`) · Para entregar al revisor junto con el instalador, el ZIP de `reserva-online/` y este documento.

Este dossier existe para que el revisor externo **no gaste su tiempo en lo que ya sabemos**. Contiene: qué es el sistema y por dónde se ataca, qué cuentas y entorno necesita, qué está prohibido, qué hallazgos ya están identificados (y cuáles corregidos) y, sobre todo, en qué queremos que se centre.

---

## 1 · Qué se entrega y qué se pide

| Entregable | Contenido |
|---|---|
| `DoelTaller-Setup-0.1.73.exe` (o la 0.1.72 + `docs/parches/0.1.73-endurecimiento.patch`) | Aplicación de escritorio Windows x64 (Electron 33, React, better‑sqlite3). |
| ZIP de `reserva-online/` | Sitio estático + funciones Netlify (`api.mjs`, `crm-productos.mjs`, `crm-proforma.mjs`, `doelprod.mjs`). Se despliega en una cuenta Netlify de **pruebas** con variables de entorno propias. |
| Código fuente (opcional, recomendado: *grey box*) | Tarball completo. Si se entrega, **rotar antes el secreto de `electron/licencia.cjs`** y no incluir `tools/generar-licencia.cjs`. |
| Este dossier + `docs/AUDITORIA_0.1.72.md` | Hallazgos conocidos y estado. |

**Lo que pedimos al revisor:**

1. Confirmar o refutar los hallazgos abiertos de la auditoría interna (§6) con prueba de concepto.
2. Buscar lo que la auditoría interna no vio, con prioridad en las superficies del §4 marcadas ★.
3. Un informe con: vía de explotación reproducible, impacto en un taller real (datos de clientes, dinero, licencia), corrección propuesta, y severidad CVSS 3.1 o equivalente.

---

## 2 · Qué es Doel Taller (en dos párrafos)

Programa de gestión para un taller de reparación de electrónica (consolas, mandos, móviles, PCs, impresoras): órdenes de trabajo con contrato y firma del cliente, presupuestos, mostrador con cobro y cajón, caja, inventario, fichaje de empleados, 22 herramientas de diagnóstico de hardware (WebHID, gamepad, PowerShell, scripts Python firmados) y copias de seguridad cifradas. **Funciona sin internet**: la base de datos es un SQLite en el PC del mostrador.

Además del PC del mostrador hay tres clientes más: el **puesto en red** (el programa entero servido por HTTP/HTTPS a otros PCs de la LAN, puerto 8735/8736), el **comandero** (PWA para el móvil del técnico y página de diagnóstico para el móvil del cliente, puerto 8734) y la **nube** (Netlify: portal del cliente para ver su orden y aceptar presupuesto, reservas, outlet, modo hotel, multitienda, enlace con el CRM de Doel). El TPV **sube** datos a la nube con un token Bearer; nunca los baja salvo reservas y solicitudes.

---

## 3 · Arquitectura y fronteras de confianza

```
                       ┌──────────────────────────── PC del mostrador (Windows) ─────────────────────────────┐
                       │  Electron main (electron/main.cjs + main/*.cjs)  ──  ~400 canales IPC `handle()`      │
                       │     ├─ electron/db/*.cjs  → doel-taller.db (SQLite, better-sqlite3)                    │
                       │     ├─ licencia.cjs (HMAC + TOTP) · seguridad-lan.cjs (freno, sesiones, cabeceras)     │
                       │     ├─ child_process: powershell, cmd, python (scripts firmados en HUELLAS.json),       │
                       │     │                 jctool.exe, pnputil/printbrm elevados (Kit SAT)                  │
                       │     └─ fetch salientes → Netlify (Bearer)                                              │
  renderer (React) ◄───┤  preload.cjs (contextBridge, window.doel.*)                                            │
  contextIsolation     │                                                                                        │
                       │  puesto-red.cjs  :8735 http / :8736 https (cert autofirmado)   ── /rpc (x-token)       │
                       │  comandero.cjs   :8734 http                                    ── /api/* (token)       │
                       └───────────────────────────────┬───────────────────────────────┬────────────────────────┘
          LAN del taller (Wi-Fi compartida con clientes) │                               │
                  PC técnico (Chrome) ◄──────────────────┘        móvil técnico / móvil cliente ◄┘
                                                                                                   
                       Internet ──► Netlify: reserva-online/public/*.html + netlify/functions/*.mjs
                                      ├─ Netlify Blobs por slug (doel-reserva-, doel-portal-, doel-outlet-, doel-hotel-, doel-mt-)
                                      └─ Supabase (CRM de Doel; claves solo en process.env de las funciones)
```

**Fronteras que importan:**

| Frontera | Quién está al otro lado | Qué le damos hoy |
|---|---|---|
| Renderer → main (IPC) | Cualquiera delante del mostrador (técnico, cliente curioso, DevTools) | Hasta la 0.1.72: todo, sin sesión. 0.1.73: DevTools cerradas. **0.1.78**: sesión local en main (`auth:login`/`auth:logout`) y permiso por canal en cada llamada IPC, misma tabla que `/rpc`. Preguntas: canales que escriben sin permiso declarado; caminos para abrir sesión sin PIN; `auth:verificar` no debe cambiar de usuario. |
| LAN → `/rpc` del puesto en red | Cualquier dispositivo en la Wi‑Fi; técnico con PIN | Login con PIN de 4 cifras, token de 32 bytes, permiso por canal (tabla `permisos-canal.cjs`), freno 5 fallos/5 min por IP. |
| LAN → comandero | Cualquier dispositivo en la Wi‑Fi | PIN de técnico para `/api/*`; `/movil` (diagnóstico del móvil del cliente) sin login. |
| Internet → funciones Netlify | Cualquiera | Escrituras del TPV con Bearer por slug (o global de comodín); portal con teléfono + clave de 8 dígitos; reservas y solicitudes de outlet sin autenticación. |
| Instalador → atacante | Cualquiera con el `.exe` | `app.asar` sin cifrar con el secreto de licencia, la semilla TOTP y la clave de copias (SEC‑01). |

---

## 4 · Superficie de ataque, por prioridad

★ = donde queremos que el revisor dedique más tiempo.

### ★ 4.1 Instalador y modelo de secretos
- `app.asar` → `electron/licencia.cjs`: `SECRETO`, `TOTP_SECRET_B32`, derivación de `CLAVE_BAK`, `cifrarTexto`, `validar`, `totpValido`, `mascaraValida`, `selloCalcular`.
- Preguntas: ¿se puede activar una licencia permanente y todos los módulos sin Doel? ¿Abrir un `.doelbak` de otro taller? ¿Descifrar `ordenes.password_cifrada` de una BD robada? ¿Generar el código del Panel de Desarrollador? (Sabemos que sí a todo: §6 SEC‑01. Queremos la PoC y cualquier derivada que no hayamos visto.)

### ★ 4.2 Puesto en red (`electron/puesto-red.cjs`, `permisos-canal.cjs`, `seguridad-lan.cjs`)
- `POST /rpc {canal, args}` con cabecera `x-token`. Lista de canales: `electron/canales.cjs` (una fila por canal con permiso y marca de red; 0.1.77).
- Preguntas: canales alcanzables sin el permiso que deberían exigir; argumentos que llegan a `child_process` o a rutas de fichero; escalada técnico → admin; DoS del mostrador; evasión del freno; fijación o robo de sesión; DNS rebinding desde una web; traversal en los estáticos; qué se filtra en errores.
- Lo que ya está cerrado en el lote 0.1.73 (verificar que lo está): freno en `fichaje:login/admin`, `app:salir`, `caja:cajonAbrir`; sesión antes de leer el cuerpo; `Host` validado; `uid` del rastro desde la sesión; `config:all` sin secretos para perfiles sin «Ajustes»; `sat:*` y cajón con permiso.

### ★ 4.3 Nube (`reserva-online/`)
- Funciones: `api.mjs` (≈40 rutas por `?r=`: sync, estado, reservar, anular, portal‑sync/login/datos/mensaje/aceptar, outlet‑*, hotel‑*, mt‑*, demo‑*, negocio‑*), `crm-productos.mjs`, `crm-proforma.mjs`, `doelprod.mjs`.
- Páginas: `index.html` (reserva), `portal.html`, `outlet.html`, `hotel.html`, `grupo.html`, `panel.html` (admin), `movil.html`/`taller.html` (diagnóstico), `demos/**`.
- Preguntas: aislamiento entre slugs (¿un taller lee/escribe el de otro?); fuerza bruta del portal (teléfono + 8 dígitos); abuso de `reservar`/`outlet-reservar`; XSS almacenado desde datos subidos por un TPV; sesiones (deterministas, sin caducidad); credencial maestra de demo; SSRF; inyección en filtros PostgREST (`crm-*`); qué pasa con la anon key del CRM si RLS está abierta (repo del CRM, fuera de este alcance salvo lo que estas funciones expongan).

### 4.4 Comandero y diagnóstico móvil (`electron/comandero.cjs`, `electron/diagnostico-movil.html`)
- `/api/login` (PIN), `/api/*` (token), `/movil` sin login, `/api/diag-actividad` (no existe: siempre 401).
- Preguntas: XSS en la PWA (todo pasa por `esc()`: buscar lo que no), BLE `dev.name` en `innerHTML`, código de acceso temporal validado en cliente con clave publicada.

### 4.5 Proceso principal y ejecución local (`electron/main.cjs`, `main/*.cjs`, `drivers-sat.cjs`, `taller-python.cjs`, `jc-toolkit.cjs`, `db/taller-hardware.cjs`)
- `execFile`/`exec`/`spawn` con PowerShell, `cmd`, Python, `jctool.exe`, `pnputil`, `printbrm`, `DiskSpd64.exe`.
- Preguntas: inyección por nombres de impresora, puertos COM, letras de unidad, duraciones; integridad de `HUELLAS.json` (sin firma, fuera del asar), carpeta `%TEMP%` fija, PATH de Python; `shell.openExternal`/`openPath` con esquemas raros; restaurar copia desde ruta arbitraria.

### 4.6 Renderer (`src/`)
- `print.js` (HTML de tickets/contratos en iframes `srcDoc` **sin sandbox** del mismo origen); `Croquis.jsx` (SVG interno con `dangerouslySetInnerHTML`); `DoelSplash.jsx` (`window.open`); parámetros `?herramienta=&v=`.
- Preguntas: cualquier interpolación cruda que llegue a texto libre de un cliente (nombres, averías, notas, nombres de artículo) → XSS con `window.doel`.

### 4.7 Base de datos y copias
- `electron/db/schema.sql`, `esquema.cjs` (migraciones y semillas), `sistema.cjs` (copias DOELBAK1/2), triggers fiscales.
- Preguntas: PIN de fábrica, hashes sin sal, cadena fiscal (`hash_anterior`) rompible, `resetOperativa`, numeración de órdenes, PII en claro (DNI en firmas, fotos espejadas sin cifrar).

### 4.8 PWA de mandos de este repositorio (`index.html`, `sw.js`)
- Estática, sin backend, sin almacenamiento: superficie mínima. Comprobar `escapeHtml` en `gp.id` y el `sw.js` (caché).

---

## 5 · Entorno de pruebas para el revisor

**Instalación de laboratorio (lo preparamos nosotros):**

1. Windows 10/11 x64 con el instalador. Primer arranque: taller «Mi Taller», usuario admin de fábrica (**cambiar el PIN y anotarlo para el revisor**; no entregar nunca el de fábrica), un perfil «Técnico» sin «Ajustes», «Cajón» ni «Inventario», y 20‑30 órdenes y clientes **ficticios** (`node tools/seed-taller-pruebas.cjs`).
2. Ajustes → Puesto en red: activado, HTTPS activado; Ajustes → Comandero: activado. Anotar la IP del mostrador.
3. Un segundo PC o VM en la misma red con Chrome, con el `.crt` instalado según el LEEME.
4. Cuenta Netlify de pruebas con el ZIP de `reserva-online/` y variables `DOEL_SYNC_TOKEN`, `DOEL_ADMIN_TOKEN`, `DOEL_SYNC_TOKEN_<SLUG>`, `SUPABASE_URL`/`SUPABASE_SECRET_KEY` apuntando a un proyecto Supabase **de pruebas**. Ajustes → Sistema del TPV enlazado a ese sitio.
5. Entregar al revisor: PIN de admin y de técnico, la URL del sitio Netlify de pruebas y el slug, la clave de un cliente del portal, el instalador y (si *grey box*) el tarball con el secreto rotado.

**Reglas:**

- Solo contra el laboratorio. Nunca contra `doel-reservas.netlify.app` ni ningún sitio en producción, ni contra el CRM real.
- Sin DoS contra Netlify de producción; en el laboratorio, DoS del mostrador y de las funciones está permitido y es de interés (SEC‑06, SEC‑19).
- Sin ingeniería social sobre personas de Doel ni de talleres clientes.
- Los datos del laboratorio son ficticios; si el revisor encuentra datos reales en algún sitio, es un hallazgo en sí mismo (y se para).

---

## 6 · Hallazgos ya conocidos (auditoría interna 15/10/2026)

El detalle, con fichero y línea, está en `docs/AUDITORIA_0.1.72.md`. Aquí el resumen para que el revisor sepa qué está abierto, qué está parcheado en el lote 0.1.73 y qué queremos que **verifique**.

| ID | Resumen | Estado | Para el revisor |
|---|---|---|---|
| SEC‑01 | Secreto HMAC + semilla TOTP + clave de copias en `app.asar`; licencias y copias de cualquier taller | **Abierto** (rediseño en curso) | Verificar; buscar derivadas (otros datos cifrados con la misma clave). |
| SEC‑02 | PIN de todos por `fichaje:login` en `/rpc` sin freno | Parcheado | Verificar que el freno no se salta (otros canales con PIN, rotación de IP, `::ffff:`). |
| SEC‑03 | `config:all` devuelve tokens y claves | Mitigado (filtro por permiso) | Buscar otros canales que devuelvan `config` o secretos (`empresa:get`, `licencia:estado`, informes). |
| SEC‑04 | Claves de portal y contraseñas de equipos por `clientes:list`/`orden-get`/`pass-ver` | Mitigado (`uid` de sesión) | Verificar la exposición de columnas; `SELECT *` hacia la red. |
| SEC‑05 | Certificado CA en Root de los PCs técnicos | Mitigado (certificado de hoja) | Verificar el nuevo certificado; MITM durante la descarga del ZIP por HTTP. |
| SEC‑06 | 16 MB antes de sesión → DoS | Parcheado | Verificar; probar slowloris y conexiones concurrentes. |
| SEC‑07 | DevTools en producción; IPC sin sesión | Parcheado (0.1.78: sesión local + permiso por canal en IPC) | Otros caminos al puente sin PIN (arrastrar enlace, `?herramienta=`, ventanas de herramienta); canales con permiso `null` que escriban algo sensible. |
| SEC‑08 | `window.open` hereda el preload | Parcheado | Verificar con una página que llame a `window.doel`. |
| SEC‑09 | Inyección `cmd /c copy` por nombre de impresora | Parcheado | Otros `exec`/`cmd` con datos (COM, IP:puerto, rutas del kit SAT). |
| SEC‑10 | PIN 4 cifras, SHA‑256 sal fija, defaults de fábrica, sin freno local | **Abierto** | Verificar; tiempo real de romper la tabla `usuarios`. |
| SEC‑11 | Copias DOELBAK1 con clave común; fotos sin cifrar; restore sin PIN | **Abierto** | Verificar. |
| SEC‑12 | `config:set` re‑sella la licencia; `dev:*` sin puerta | **Abierto** | Verificar; desde LAN con «Ajustes». |
| SEC‑13 | XSS reflejado en `index.html` pública; sin cabeceras | Parcheado (sin CSP aún) | Buscar más XSS en `public/*.html` (sobre todo `grupo.html`) y en `demos/`. |
| SEC‑14 | Clave HMAC de códigos de diagnóstico publicada | **Abierto** | Verificar; valorar si hay impacto más allá de saltarse el código. |
| SEC‑15 | Canales sensibles sin permiso en `/rpc` | Parcheado (0.1.77: `canales.cjs` único + deny‑by‑default) | Revisar fila a fila `electron/canales.cjs`: canales sin `permiso` que escriban o lancen procesos; `red: 'local'` que falte. |
| SEC‑16 | Sin logout; permisos congelados en sesión | **Abierto** | Verificar. |
| SEC‑17 | Freno solo por IP | Mitigado (`esLocal` en comandero) | IP alias / DHCP; tiempo real para 10⁴ PIN. |
| SEC‑18 | DNS rebinding | Mitigado (`Host` en `/rpc`) | Comandero sigue sin validar `Host`. |
| SEC‑19 | Nube: fuerza bruta, aislamiento entre slugs, abuso de reservas, maestro demo, sesiones eternas | **Abierto** | **Prioridad**: es lo único expuesto a Internet. |
| SEC‑20 | `abrir-url` con esquemas; DiskSpd `exec(string)` | Mitigado (esquemas) | DiskSpd y `tpv-impresora-test` como SSRF. |
| SEC‑21 | iframes `srcDoc` sin sandbox; interpolaciones crudas en `print.js` | **Abierto** | Encontrar un campo de texto libre que llegue crudo. |
| SEC‑22 | `HUELLAS.json` sin firma; scripts desde `%TEMP%` | **Abierto** | Solo con escritura local: persistencia. |
| SEC‑23 | Electron 33 EOL; `selfsigned`/`node-forge` | **Abierto** | Explotabilidad real de los CVE de Electron en esta app (sin `webview`, con `contextIsolation`). |

---

## 7 · Lista de comprobación que usará el revisor (y que podemos pasar antes nosotros)

Adaptada de OWASP ASVS 4.0 nivel 2 y de la guía de seguridad de Electron. Marcar ✅/❌/➖.

**Electron**
- [ ] `contextIsolation: true`, `nodeIntegration: false`, `sandbox` (hoy solo en impresión), `webSecurity` por defecto, sin `allowRunningInsecureContent`, sin `<webview>`.
- [ ] `setWindowOpenHandler` y `will-navigate` en **todas** las ventanas (lote 0.1.73).
- [ ] `devTools` cerradas en producción (lote 0.1.73); menú de aplicación sin «Toggle DevTools».
- [ ] CSP en `index.html` (pendiente); `plugins:true` solo para el visor PDF.
- [ ] Ningún `loadURL` remoto en producción; `shell.openExternal` solo `https:/http:/mailto:` (lote).
- [ ] Todo `child_process` sin shell y con argumentos validados (pendiente: DiskSpd).
- [ ] ASAR integrity / fuses de Electron (`@electron/fuses`: `RunAsNode=false`, `EnableNodeCliInspectArguments=false`, `OnlyLoadAppFromAsar=true`) — **no configurados hoy**: recomendado.
- [ ] Firma de código del instalador (Authenticode) — no hay certificado de firma hoy: recomendado (SmartScreen y manipulación del `.exe`).

**Autenticación y sesiones**
- [ ] PIN con hash lento y sal por usuario (pendiente SEC‑10).
- [ ] Freno en todo punto que valide un PIN: Electron local (pendiente), `/rpc` (lote), comandero (sí), Netlify (pendiente).
- [ ] Tokens aleatorios (sí), en cabecera (sí), con caducidad absoluta y logout (pendiente SEC‑16).
- [ ] PIN de fábrica forzado a cambiar (pendiente).

**Autorización**
- [x] Permiso por canal en `/rpc` con deny‑by‑default (0.1.77) y también en IPC (0.1.78): revisar la clasificación de `electron/canales.cjs`.
- [ ] El usuario del rastro lo pone el servidor (lote).
- [ ] Lecturas con proyección de columnas, nunca `SELECT *` hacia la red (pendiente SEC‑04).

**Datos y cripto**
- [ ] Ningún secreto en el código distribuido (❌ SEC‑01).
- [ ] Clave de copias y de cifrado por instalación (pendiente).
- [ ] Tokens de la nube en `safeStorage` (pendiente).
- [ ] Copias cifradas enteras, fotos incluidas (pendiente SEC‑11).
- [ ] Cadena fiscal inmutable (✅), `resetOperativa` con puerta real (pendiente).

**Red local**
- [ ] Solo IP privadas (✅ puesto; lote en comandero); `Host` validado (lote en `/rpc`; pendiente comandero).
- [ ] Certificado de hoja (lote), huella visible en Ajustes (pendiente), ZIP por HTTPS o con huella comparada (pendiente).
- [ ] Tope de cuerpo según sesión (lote); `headersTimeout`/`requestTimeout` explícitos (pendiente).
- [ ] Errores sin detalle (lote).

**Nube**
- [ ] Cabeceras de seguridad (lote parcial; CSP pendiente).
- [ ] Escape de todo dato en `innerHTML` (pendiente `grupo.html`).
- [ ] Freno por IP en logins (pendiente); clave por slug obligatoria (pendiente); sesiones con caducidad (pendiente).
- [ ] Sin credenciales fijas en código (❌ maestro de demo).
- [ ] `timingSafeEqual` en tokens (pendiente).

**Dependencias y build**
- [ ] `npm audit --omit=dev` a cero (❌ Electron, selfsigned).
- [ ] Electron con soporte (❌ 33 → ≥ 42).
- [ ] `HUELLAS.json` firmado y con la versión de la app (pendiente).

---

## 8 · Qué cerrar ANTES de entregar al revisor

Si el revisor recibe la 0.1.72 tal cual, el informe dirá lo que ya dice `docs/AUDITORIA_0.1.72.md`. Para que su tiempo valga:

1. Aplicar el lote 0.1.73 y pasar el ritual en Windows.
2. **SEC‑01**: al menos rotar el secreto y, si se entrega código, retirar `tools/generar-licencia.cjs`. Lo ideal es entregar ya la firma asimétrica.
3. **SEC‑10**: hash lento con sal por usuario y PIN de fábrica forzado.
4. **SEC‑19**: freno por IP en `portal-login`/`demo-login`/`mt-login`, eliminar el maestro de demo, clave por slug obligatoria.
5. Electron a la estable actual (SEC‑23) para que los CVE de Chromium no ocupen la mitad del informe.

Lo demás puede quedar abierto y ser lo que el revisor confirme o amplíe.

---

## 9 · Formato de informe que pedimos

Por hallazgo: ID, título, severidad (CVSS 3.1), componente y fichero:línea si hay código, precondiciones (¿PIN de técnico? ¿en la LAN? ¿solo con el instalador?), pasos reproducibles (curl, captura, script), impacto para un taller (datos de clientes, dinero, licencia, disponibilidad), corrección propuesta. Resumen ejecutivo de una página y tabla de hallazgos ordenada por severidad. Si encuentra datos reales de personas en cualquier punto del laboratorio, avisar de inmediato y detener la prueba en ese componente.
