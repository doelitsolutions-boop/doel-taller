# Auditoría — Doel TPV v6.0.28 (+ PWA Doel Taller)

**Fecha:** 2026-10-20 · **Autor:** revisión asistida (Claude Code) · **Alcance:** código propio de Doel IT Solutions

**Estado general: APROBADO con observaciones menores.** No se encontró ninguna
vulnerabilidad crítica ni explotable de forma remota. El código está bien
construido en lo que a seguridad respecta. Se aplicaron dos correcciones de
endurecimiento (defensa en profundidad) y una mejora en la PWA.

> Autorización: auditoría **defensiva sobre código propio**, dentro del supuesto
> de la skill `hacking-etico` (revisión de los propios sistemas). No se atacó
> ningún sistema en producción; todo el análisis es estático sobre el código
> subido.

---

## Resumen ejecutivo

El TPV es una aplicación Electron (proceso principal Node + interfaz React/Vite)
con un micro-servicio en Netlify (reserva online y tienda). La postura de
seguridad es sólida: aislamiento de contexto activado, sin integración de Node
en el renderer, puente IPC por canales concretos, hashing de contraseñas con
scrypt+sal y comparación en tiempo constante, SQL 100 % parametrizado, ejecución
de Python restringida a un catálogo con argumentos numéricos saneados, y lista
blanca para abrir URLs externas. Los hallazgos son de **defensa en profundidad**,
no fallos explotables hoy.

---

## Hallazgos

| # | Categoría | Descripción | Severidad | Acción |
|---|-----------|-------------|-----------|--------|
| 1 | Electron / permisos | Los manejadores de permisos tenían `|| true`, que anulaba la lista blanca: cualquier permiso (geolocalización, notificaciones, media…) y cualquier dispositivo se concedían siempre. | Medio | **Corregido** en el patch |
| 2 | Electron / navegación | No había `setWindowOpenHandler` ni guarda de `will-navigate`: nada contenía una navegación o ventana emergente si algún contenido la provocara. | Bajo | **Corregido** en el patch |
| 3 | Servidor / auth | Los tokens Bearer compartidos (`DOEL_SYNC_TOKEN`, `DOEL_ADMIN_TOKEN`) se comparan con `!==` (no es tiempo constante), mientras que las contraseñas sí usan `timingSafeEqual`. | Bajo | Recomendado (ver abajo) |
| 4 | PWA / cabeceras | La PWA Doel Taller no declaraba `Content-Security-Policy`. | Bajo | **Mejorado** (CSP añadida) |
| 5 | Electron / superficie | `plugins: true` habilita el visor PDF de Chromium (biblioteca de diagramas). Justificado, pero amplía superficie. | Info | Mantener solo si se usa el visor |
| 6 | Identidad visual | La PWA usa verde `#5fea08` y navy `#07284b`; la guía corporativa de la skill de auditoría indica verde neón `#62E70E` y azul marino `#082947`. | Menor | Unificar tokens de color |

Leyenda: Crítico = bloquea · Medio = degradación/defensa rota · Bajo/Menor = endurecimiento o cosmético.

---

## Lo que está bien (verificado)

- **Aislamiento Electron.** `contextIsolation: true`, `nodeIntegration: false`.
  El `preload.cjs` expone solo canales IPC concretos con `ipcRenderer.invoke`;
  no se filtra `ipcRenderer` genérico ni módulos de Node al renderer.
- **Contraseñas.** scrypt (N=16384, r=8, p=1) con sal aleatoria y verificación
  con `timingSafeEqual`, tanto en el TPV (`electron/claves.cjs`) como en la nube
  (`api.mjs`, `tienda.mjs`). Se mantiene verificación del formato antiguo solo
  para migración.
- **SQL.** 942 sentencias preparadas con enlace de parámetros (`?`). No hay
  concatenación de entrada de usuario en el texto SQL; las concatenaciones vistas
  arman etiquetas o se pasan como parámetro enlazado (`'%' + serie + '%'`).
- **Ejecución de Python** (`taller-python.cjs`). Solo scripts de un catálogo
  fijo, nunca una ruta que venga de la interfaz, y con argumentos acotados a
  rangos numéricos (`parseInt` + `Math.max/min`). El JSON del agente portable se
  valida en forma y tamaño y se recortan los textos.
- **Abrir URLs externas** (`sistema:abrir-url`). Lista blanca estrecha con barra
  final obligatoria para evitar el bypass tipo `https://dominio.evil.com`.
- **Generación de códigos.** `crypto.randomInt` (CSPRNG), no `Math.random`.
- **Integridad fiscal.** Triggers SQLite que abortan el `DELETE` de tickets y el
  `UPDATE` de los campos de la cadena de hash (serie, número, fecha, total, hash).
- **Asistencia remota** (`remoto.cjs`). El TPV solo envía; la contraseña permanente
  del equipo nunca entra en el heartbeat; la sesión va por el servidor RustDesk.
- **Sin secretos en el repositorio.** Todas las claves viven en variables de
  entorno de Netlify. No hay `console.log` de depuración en cliente ni en el
  proceso principal.
- **PWA.** El nombre del mando reportado por el dispositivo se escapa con
  `escapeHtml` antes de ir al DOM; sin recursos externos; funciona offline.

---

## Correcciones aplicadas

### 1 y 2 — Endurecimiento del proceso Electron (`auditoria/tpv-v6.0.28-endurecimiento-electron.patch`)

El patch, listo para aplicar sobre el TPV con `git apply` (o `patch -p1`), hace:

- **Permisos reales.** Quita el `|| true` de `setPermissionRequestHandler` y
  `setPermissionCheckHandler`, de modo que la lista blanca
  (`gamepad, hid, serial, media, usb`) vuelve a aplicar. `setDevicePermissionHandler`
  pasa a conceder solo dispositivos `hid`/`serial`/`usb` (las herramientas del
  taller) en lugar de cualquiera.
- **Navegación contenida.** Añade `setWindowOpenHandler` que deniega ventanas
  emergentes y una guarda de `will-navigate` que solo permite el propio origen
  (`file://` en producción, `localhost:5173` en desarrollo). Los enlaces externos
  siguen pasando por `sistema:abrir-url`, que ya tiene lista blanca.

Ambos archivos resultantes pasan `node --check`. Es un cambio de bajo riesgo: no
toca la lógica de negocio, solo cierra permisos que hoy estaban abiertos.

### 4 — CSP en la PWA Doel Taller (este repositorio)

Se añadió a `index.html` una `Content-Security-Policy` (y `referrer: no-referrer`).
La app es un único fichero autocontenido, sin recursos externos, con estilos y
scripts en línea, así que la política permite `'unsafe-inline'` para script/estilo
pero bloquea orígenes externos, `object-src`, `frame-ancestors` y `form-action`.
Se subió `CACHE_NAME` del service worker a `doel-taller-v2` para que las
instalaciones ya existentes reciban la nueva cabecera.

---

## Mejoras recomendadas (no aplicadas)

1. **Tokens Bearer en tiempo constante (hallazgo 3).** Reutilizar el patrón de
   `claveOk`: comparar un `sha256` del token recibido contra el del esperado con
   `timingSafeEqual`, en lugar de `!==`. Afecta a `api.mjs`, `tienda.mjs`,
   `crm-productos.mjs` y `crm-proforma.mjs`. Riesgo real bajo (ataque de timing
   sobre red casi inviable), pero deja la autenticación coherente.
2. **Peso de los módulos de interfaz.** `Ajustes.jsx` (~8 900 líneas) y varias
   vistas pesadas (`DiagnosticoMando`, `Agenda`, `Domicilio`, `Fichaje`) son
   candidatas a `React.lazy()` para acortar el arranque.
3. **Unificar tokens de color** con la guía corporativa (`#62E70E` / `#082947`).
4. **`asset-links` / paquete.** `README-APK.md` sugiere `com.doelitsolutions.taller`
   pero `.well-known/assetlinks.json` declara `io.github.doelitsolutions_boop.twa`
   (con huella real). Es correcto si ese es el paquete firmado; conviene dejar el
   README alineado para no confundir en la próxima firma.

---

## Preparado para `/hacking-etico`

La skill `hacking-etico` queda instalada en este repositorio bajo
`.claude/skills/hacking-etico/` (con sus referencias
`checklist-revision-propia.md` y `vulnerabilidades-web.md`), de modo que
`/hacking-etico` es invocable aquí. Esta auditoría ya recorrió su checklist de
auto-revisión: Mass Assignment, comparaciones débiles, inyección SQL, LFI, subida
de ficheros, XXE, SSRF, CSRF, XSS, `.git`/secretos expuestos y endurecimiento.
Resultado: sin hallazgos explotables; los puntos abiertos son los de la tabla.

## Próximos pasos

1. Aplicar el patch al repositorio del TPV y publicar como v6.0.29.
2. Decidir sobre la mejora 1 (tokens en tiempo constante) en la nube de Netlify.
3. Desplegar la PWA con la CSP nueva y verificar en el móvil que todo sigue
   funcionando (mando, WebHID de batería).
