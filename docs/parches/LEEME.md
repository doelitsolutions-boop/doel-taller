# Parches de endurecimiento

| Fichero | Sobre | Qué lleva |
|---|---|---|
| `0.1.73-endurecimiento.patch` | 0.1.72 | 20 de 24 hallazgos de la auditoría (ver abajo) |
| `0.1.74-licencia-v2.patch` | 0.1.73 | **SEC‑01**: licencia v2 (códigos DL2 firmados + semilla TOTP por instalación), 21 ficheros. Guía: `docs/LICENCIA_V2.md` |
| `0.1.75-copias.patch` | 0.1.74 | **Punto 2**: paquete de copia DOELBAK3 (base + fotos + manifiesto cifrados; clave de instalación con frase propia), restauración completa, prueba en seco que cuenta fotos |
| `0.1.76-dependencias.patch` | 0.1.75 | **Punto 3**: Electron 33 → 44, electron-builder 26, Vite 8, better-sqlite3 13, selfsigned 5 (asíncrono: `certificado()`/`iniciar()` del puesto en red y sus baterías), `npm audit --omit=dev` a cero. No incluye `package-lock.json`: tras aplicarlo, `npm install` lo regenera. Sin firma de código (coste 200‑400 €/año; uso interno) |
| `0.1.76-acumulado-sobre-0.1.72.patch` | 0.1.72 | todos los anteriores juntos (sin `package-lock.json`) |

Tras aplicar el de la 0.1.76 hay que **probar en Windows** lo que toca a Electron: WebHID y mando, visor PDF de diagramas, impresión silenciosa, safeStorage e instalador NSIS. El contenedor no puede arrancar Electron.

Tras aplicar el de la 0.1.74, **antes de compilar el instalador de un taller**: `node tools/licencia-v2.cjs claves` en el PC de Doel (escribe la clave pública en `electron/licencia.cjs`). Sin ese paso la app arranca y funciona, pero no puede activar códigos DL2 (`check-puente` lo avisa en el paso 9). Validado igual que el anterior: build, cinco comprobadores y 89/91 baterías (las dos de entorno).

# Endurecimiento 0.1.73 (parche acumulado sobre la 0.1.72)

Dos formas de aplicarlo:

1. **Tarball completo** `doel-taller-v0.1.73.tar.gz` (entregado junto a este repo; no lleva `node_modules/`): descomprimir, `npm install`, `npm run dist`.
2. **Parche** `0.1.73-endurecimiento.patch` sobre el árbol de la 0.1.72:

```
cd doel-taller            # raíz del proyecto Electron, versión 0.1.72
git apply --check docs/parches/0.1.73-endurecimiento.patch   # o: patch -p1 --dry-run < …
git apply docs/parches/0.1.73-endurecimiento.patch
npm install
npx vite build && node tools/check-undef.cjs && node tools/check-puente.cjs && node tools/check-hooks.cjs && node tools/check-await-evento.cjs && node tools/check-catch.cjs
for t in tools/test-*.cjs; do node $t; done
```

El parche ya sube la versión a 0.1.73 y escribe las filas de `VERSION.md` y `ESTADO.md`; no hace falta `version-subir`.

Validado en contenedor Linux: compila, los cinco comprobadores en verde, 89 de 91 baterías. Las dos que no pasan aquí son las mismas que en la 0.1.72 sin tocar: `test-main-modulos 1b` (el binario de Electron no se instala en el contenedor) y `test-pantallas` 45/47 (dos pasos de `jsdom`, a confirmar en Windows).

## Qué cambia

Resumen por hallazgo en `docs/AUDITORIA_0.1.72.md → «Estado tras la 0.1.73»`. Ficheros tocados:

| Zona | Ficheros | Hallazgos |
|---|---|---|
| Proceso principal | `electron/main.cjs`, **nuevo** `electron/seguridad-app.cjs`, `electron/main/impresion.cjs`, `electron/main/fichaje.cjs`, `electron/preload.cjs`, `electron/licencia.cjs` | SEC‑01 (freno), 07, 08, 09, 11, 12, 20, 24 |
| Capa de datos | `electron/db/_nucleo.cjs`, `sistema.cjs`, `fichaje.cjs`, `esquema.cjs`, `clientes.cjs`, `taller-ordenes.cjs`, `taller-hardware.cjs`, `db.cjs` | SEC‑04, 10, 11, 14, 20, 24 |
| LAN | `electron/puesto-red.cjs`, `electron/comandero.cjs`, `electron/permisos-canal.cjs`, `electron/seguridad-lan.cjs`, `electron/diagnostico-movil.html` | SEC‑02, 03, 05, 06, 14, 15, 16, 17, 18, 24 |
| Interfaz | `src/App.jsx`, `src/components/Login.jsx`, `src/print.js`, `src/components/VistaPrevia.jsx`, `TallerPresupuestos.jsx`, `ajustes/impresoras.jsx` | SEC‑10, 16, 21 |
| Scripts | `electron/taller-python.cjs` | SEC‑22 |
| Nube | `reserva-online/netlify/functions/api.mjs`, `crm-productos.mjs`, `netlify.toml`, `public/index.html`, `movil.html`, `taller.html`, `grupo.html`, `panel.html`, `HUELLAS-NUBE.json`, `version.txt` | SEC‑13, 14, 19 |
| Baterías | `tools/test-seguridad-lan.cjs`, `test-taller.cjs`, `test-diag-actividad.cjs`, `test-herramientas-diag.cjs` | adaptadas al comportamiento nuevo |
| Documentación | `VERSION.md`, `ESTADO.md` | — |

## Efectos que se notan al instalar

- **Certificado del puesto en red**: se regenera como certificado de hoja (antes era una CA). Volver a bajar el ZIP del acceso directo e instalar el `.crt` nuevo en los PCs técnicos; conviene retirar el antiguo de «Entidades de certificación raíz de confianza».
- **Entrar por la IP**: `/rpc` y el comandero (`/api/*`) exigen que el `Host` sea la IP del mostrador o `localhost`. Un nombre de equipo da 421 con un texto que lo explica.
- **PIN**: se reescriben solos con sal + scrypt la primera vez que cada uno entra bien. Nadie teclea nada nuevo. Cinco PIN mal seguidos en el mostrador → 5 minutos sin aceptar PIN (el puesto en red y el comandero tienen su propio freno por IP y ya lo tenían).
- **Panel de Desarrollador**: el código del autenticador abre el panel 30 minutos en el proceso principal; renovar, desactivar, puesta a cero, restaurar, copia completa y los cambios de licencia/módulos por `config:set` exigen la puerta abierta. Cinco códigos mal → 5 minutos.
- **Clave de instalación** (`<datos>/clave-instalacion.bin`, protegida con DPAPI en Windows): lo que se cifra a partir de ahora (contraseñas de equipos, claves de portal, claves web de distribuidores, frase de copias) solo se abre en ESE PC y usuario. Lo cifrado antes se sigue leyendo. Si la base se restaura en otro PC: la frase de copias se vuelve a poner en Ajustes (la restauración la pide al momento, como siempre) y las contraseñas de equipos de órdenes abiertas se piden otra vez al cliente.
- **Puesta en marcha**: nuevo punto «Clave propia de las copias de seguridad».
- **Puesto en red**: un perfil sin «Ajustes» ya no ve los tokens de la nube; «Cerrar sesión» cierra la sesión también en el servidor; quitar un permiso o dar de baja afecta a la sesión viva en la siguiente llamada; Kit SAT, cajón de prueba y exportar fichajes piden permiso.
- **Órdenes**: `taller:orden-get` ya no trae `password_cifrada` (trae `tiene_password`); la contraseña se lee solo con «Ver contraseña», como antes.
- **Nube**: hay que redesplegar `reserva-online/` (ZIP `doel-reservas-v0.1.73-netlify-drop.zip`): nueva ruta `/api/diag-validar`, freno por IP en un blob `doel-freno`, sesiones del portal y multitienda con caducidad de 12 h (opcional `DOEL_SESION_SECRETO` en las variables de entorno; sin él se firma con la clave de sync). El acceso maestro de demo desaparece: el primer usuario de demo se crea desde el panel.
- **DevTools**: Ctrl+Shift+I no abre nada en producción; en desarrollo (`npm run dev`) sigue.

## Lo que NO cierra (decisiones de producto o pruebas en Windows)

- **SEC‑01**: el secreto HMAC y la semilla TOTP siguen en el `asar`. Firmar licencias con clave asimétrica cambia cómo se emiten los códigos. Mientras no se decida: rotar el secreto antes de dar código a terceros y no distribuir `tools/generar-licencia.cjs`.
- **DOELBAK1** sin frase propia sigue con la clave común (sin frase no hay recuperación en otro PC); la puesta en marcha ahora lo pide.
- **Electron 33 → actual** (SEC‑23): requiere probar WebHID, gamepad y visor PDF en Windows tras el salto.
- CSP en la nube y en `index.html` de Electron; `UNIQUE(ordenes.numero)` con contador; firma de `HUELLAS.json`; autorización por canal también en IPC.
