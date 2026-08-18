# Doel Taller — generar el .apk desde GitHub

Todo esto se hace desde el navegador, sin instalar nada en tu PC.
Usamos Bubblewrap (herramienta oficial de Google, la misma que usa PWABuilder
por dentro) para envolver la PWA en una app Android instalable (TWA).

---

## Paso 1 — Crear el repositorio y subir los archivos

1. Crea un repo nuevo en GitHub, público, por ejemplo `doel-taller`.
2. Sube a la raíz estos archivos (los tienes ya generados):
   - `index.html`
   - `manifest.json`
   - `sw.js`
   - `icon.svg`
   - `.nojekyll` (archivo vacío — sin esto, GitHub Pages ignora la carpeta
     `.well-known` y la app no se abrirá en pantalla completa)
   - la carpeta `.well-known/assetlinks.json` (la rellenaremos en el paso 3)
   - la carpeta `.github/workflows/build-apk.yml`

## Paso 2 — Activar GitHub Pages

1. Settings → Pages → Source: "Deploy from a branch" → rama `main`, carpeta `/ (root)`.
2. Espera 1-2 minutos. Tu URL será:
   `https://TU-USUARIO.github.io/doel-taller/`
3. Verifica que `https://TU-USUARIO.github.io/doel-taller/manifest.json`
   carga correctamente en el navegador.

## Paso 3 — Generar el proyecto Android (una sola vez, en Codespaces)

1. En el repo, botón verde "Code" → pestaña "Codespaces" → "Create codespace on main".
   Se abre un VS Code en el navegador con terminal incluido.
2. En la terminal del Codespace:
   ```bash
   npm install -g @bubblewrap/cli
   bubblewrap init --manifest="https://TU-USUARIO.github.io/doel-taller/manifest.json" --directory=twa-project
   ```
3. Te hará varias preguntas — respuestas recomendadas:
   - **Domain**: confirma que es `TU-USUARIO.github.io`
   - **Application name**: `Doel Taller`
   - **Short name**: `Doel Taller`
   - **Package ID**: `com.doelitsolutions.taller` (o el que prefieras, formato inverso de dominio)
   - **Icon URL**: cuando pregunte por el icono, usa
     `https://TU-USUARIO.github.io/doel-taller/icon-512.png` (ya lleva el
     logo real de Doel sobre fondo navy corporativo — funciona directamente como
     icono de la app y del APK)
   - **Signing key**: cuando pregunte por la contraseña del keystore, elige una y
     **apúntala** — la necesitarás en el paso 4.
4. Cuando termine, construye el primer APK:
   ```bash
   cd twa-project
   bubblewrap build
   ```
5. Al terminar verás en la terminal una línea con el **SHA256 fingerprint** del
   certificado de firma (algo como `AB:CD:12:...`). Cópiala.
6. Edita `.well-known/assetlinks.json` en el repo (desde el propio Codespace) y
   sustituye `REEMPLAZA_ESTO_CON_LA_HUELLA...` por esa huella, sin los dos puntos
   ni espacios (formato `AB:CD:12...` tal cual la da bubblewrap, en mayúsculas).
7. Haz commit y push de:
   - la carpeta `twa-project/` completa (incluye el keystore `android.keystore` —
     si el repo es público, considera hacerlo **privado**, porque ese keystore
     firma tu app; o mejor, no subas el `.keystore` y en su lugar guárdalo como
     secreto, ver nota abajo)
   - `.well-known/assetlinks.json` ya actualizado
8. Descarga `app-release-signed.apk` desde el propio Codespace (clic derecho →
   Download) — ese es ya tu primer .apk instalable.

> **Nota sobre el keystore**: si prefieres no subir el archivo `android.keystore`
> al repo (recomendable si es público), súbelo como *secret* en Settings →
> Secrets and variables → Actions, o simplemente manténlo solo dentro del
> Codespace y descarga el APK generado ahí cada vez que actualices la app,
> sin usar el workflow automático del paso 4.

## Paso 4 — Reconstruir el APK automáticamente en el futuro (opcional)

Con `twa-project/` ya en el repo, cada vez que cambies `index.html` puedes:
1. Actualizar `twa-project/app/src/main/assets` no hace falta tocarlo — el
   contenido real lo sirve tu GitHub Pages, no el APK.
2. Ir a la pestaña **Actions** del repo → workflow "Build APK" → "Run workflow".
3. Cuando termine (2-4 min), en el resumen del run hay un artefacto
   `doel-taller-apk` descargable con el `.apk` dentro de un `.zip`.

Si guardaste la contraseña del keystore como secretos
`BUBBLEWRAP_KEYSTORE_PASSWORD` y `BUBBLEWRAP_KEY_PASSWORD` (Settings → Secrets
and variables → Actions), el workflow ya está preparado para usarlos.

## Paso 5 — Instalar el .apk en el móvil (Android)

1. Pasa el `.apk` al móvil (por USB, Drive, WhatsApp a ti mismo, etc.)
2. Ábrelo desde el explorador de archivos del móvil.
3. Android pedirá permitir "instalar apps de origen desconocido" para esa
   fuente la primera vez — actívalo solo para esa app si te lo pregunta.
4. Listo: icono propio, pantalla completa, funciona offline, y el mando
   USB/Bluetooth se detecta automáticamente igual que en el navegador.

## Paso 6 — Instalar en iPhone (gratis, sin Mac, sin cuenta Apple Developer)

Un `.ipa` nativo de verdad (App Store o sideload estable) exige Mac + Xcode +
cuenta Apple Developer (99 €/año) — desproporcionado para esto. La alternativa
gratuita da el mismo resultado práctico: icono en pantalla de inicio, pantalla
completa, funciona offline. La Gamepad API funciona en Safari iOS desde la
versión 10.3, así que el mando se detecta igual que en Android.

1. Con el sitio ya alojado (mismo GitHub Pages del paso 2), abre esa URL
   **en Safari** en el iPhone (tiene que ser Safari, no Chrome — solo Safari
   puede instalar en pantalla de inicio en iOS).
2. Toca el icono de compartir (el cuadrado con la flecha hacia arriba).
3. "Añadir a pantalla de inicio".
4. Se crea un icono con el logo de Doel Taller. Al abrirlo, entra a pantalla
   completa, sin barra de Safari — el `index.html` ya lleva las meta etiquetas
   `apple-mobile-web-app-capable` necesarias para esto.
5. Conecta el mando (USB-C con adaptador, Lightning con adaptador, o
   Bluetooth) y pulsa cualquier botón — se detecta igual que en el navegador
   de escritorio.

> Esta instalación es completamente offline una vez añadida: el service
> worker cachea la app entera, así que no hace falta cobertura ni wifi en
> la calle.

---

### Por qué no lo hago yo automáticamente

Bubblewrap necesita responder preguntas interactivas la primera vez
(nombre del paquete, contraseña de firma...) y eso no se puede automatizar
de forma fiable en un entorno sin supervisión — es una limitación conocida
de la propia herramienta, no de GitHub Actions. Por eso el primer `init` se
hace a mano una vez en Codespaces (5 minutos) y todo lo posterior sí queda
automatizado.
