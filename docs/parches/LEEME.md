# Lote de endurecimiento 0.1.73

`0.1.73-endurecimiento.patch` se aplica sobre el árbol de **Doel Taller 0.1.72** (el tarball, no este repo):

```
cd doel-taller            # raíz del proyecto Electron, versión 0.1.72
git apply --check docs/parches/0.1.73-endurecimiento.patch   # o: patch -p1 --dry-run < …
git apply docs/parches/0.1.73-endurecimiento.patch
npx vite build && node tools/check-puente.cjs && node tools/test-seguridad-lan.cjs && node tools/test-puesto-red.cjs && node tools/test-permisos-canal.cjs
```

Validado en contenedor Linux: compila, los cinco comprobadores en verde y 89/91 baterías (las mismas dos
que la 0.1.72 sin parche: `test-main-modulos 1b` por falta del binario de Electron y `test-pantallas` 45/47).

## Qué cambia (10 ficheros)

| Fichero | Cambio | Cierra |
|---|---|---|
| `electron/puesto-red.cjs` | Freno en `fichaje:login/admin`, `app:salir`, `caja:cajonAbrir` · sesión antes de leer el cuerpo (256 KB sin sesión, 16 MB con) · `Host` validado en `/rpc` (421) · `uid` del rastro = usuario de la sesión · `config:all` sin `*_token/_clave/_key/_hash/_pin/_hmac/_sello` para quien no tenga «Ajustes» · certificado de hoja (no CA), 825 días, formato 3 (se regenera solo) · estáticos con separador · 500 sin detalle (va al registro) | SEC‑02, 03, 04, 05, 06, 18, 24 |
| `electron/comandero.cjs` | Solo IP privadas (`esLocal`) | SEC‑17 |
| `electron/permisos-canal.cjs` | `sat:*` → ajustes · `caja:cajonProbar`, `sat:cajon-raw`, `taller:tpv-impresora-test` → cajón · `salud:mantenimiento`, `docs:borrar`, `fichaje:export` → ajustes | SEC‑09, 15 |
| `electron/main.cjs` | `endurecerVentana()`: `setWindowOpenHandler` (https → navegador del sistema, resto denegado), `will-navigate` solo `file:`, sin webview · `devTools` solo con `VITE_DEV` · nombre de impresora del cajón validado antes del respaldo `cmd /c copy` · `sistema:abrir-url` solo `https:/http:/mailto:` | SEC‑07, 08, 09, 20 |
| `electron/main/impresion.cjs` | Ventanas de impresión en sandbox, sin abrir ventanas ni navegar | SEC‑15 |
| `electron/db/taller-ordenes.cjs` | Clave del portal con `crypto.randomInt` | SEC‑24 |
| `electron/licencia.cjs` | `timingSafeEqual` en la firma | SEC‑24 |
| `reserva-online/public/index.html` | `token` validado y escapado | SEC‑13 |
| `reserva-online/netlify.toml` | X‑Frame‑Options, nosniff, Referrer‑Policy, HSTS, Permissions‑Policy | SEC‑13 |
| `tools/test-seguridad-lan.cjs` | 9b comprueba la condición nueva de los estáticos | — |

## Efectos que se notan

- El certificado HTTPS del puesto en red se **regenera** al arrancar: hay que volver a bajar el ZIP del acceso directo e instalar el `.crt` nuevo en los PCs técnicos (y conviene retirar el antiguo de «Entidades raíz de confianza»).
- `/rpc` exige entrar por la **IP** del mostrador (o `localhost`); un nombre de equipo da 421 con un texto que lo dice.
- Un perfil **sin «Ajustes»** ya no ve los tokens de la nube en Ajustes del puesto en red (un admin sí: nada se borra al guardar).
- Kit SAT, cajón de prueba y exportar fichajes piden permiso desde el puesto en red.
- Ctrl+Shift+I no abre nada en producción; en desarrollo (`npm run dev`) sigue funcionando.

Después de aplicarlo: `node tools/version-subir.cjs` y anotar en `VERSION.md`/`ESTADO.md` los SEC‑10…28 de los comentarios del código.
