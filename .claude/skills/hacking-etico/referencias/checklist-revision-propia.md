# Checklist de auto-revisión — tus páginas y apps

> ✅ **Uso previsto: revisar TU PROPIO código** (Doel IT: TPV, Hospitality,
> NetPulse, doelitcard, doelsplash, Doel Tools, etc.). Es una auditoría
> defensiva. Por cada clase de fallo tienes: **qué buscar**, un **`grep` para
> localizar el patrón** en tu repo, y **cómo arreglarlo**. Detalle de cada
> vulnerabilidad en [`vulnerabilidades-web.md`](vulnerabilidades-web.md).

## Cómo usar este checklist

1. Sitúate en la raíz del proyecto a revisar.
2. Lanza los `grep` de cada apartado. Un resultado **no es** automáticamente un
   fallo: es un punto que **debes mirar a mano**.
3. Marca cada casilla cuando lo hayas verificado o corregido.
4. Si tienes `/security-review` o la skill `doel-code-audit` en la sesión,
   úsalas en paralelo; este checklist las complementa con foco ofensivo.

```bash
# Prepara la revisión
cd /ruta/a/tu/proyecto
# Excluye dependencias en todas las búsquedas:
GREPX='--exclude-dir=node_modules --exclude-dir=vendor --exclude-dir=.git -rniI'
```

---

## 1. Registro / usuarios — ¿Mass Assignment?

- [ ] El registro/edición de usuario **no** vuelca el body entero al modelo.
- [ ] El rol/permiso se fija en el servidor, nunca desde datos del cliente.

```bash
grep $GREPX -E 'create\(.*req(uest)?\.(body|all|params)|->fill\(|mass_assign|\$_POST\)' .
grep $GREPX -E '"?(role|is_admin|admin|isAdmin|verified|premium|balance)"?' .
```
**Arreglo:** lista blanca de campos (`permit`, `fillable`, DTO). El rol se asigna
por lógica de servidor.

## 2. Login / comparaciones — ¿bypass por `==` en PHP?

- [ ] Contraseñas con `password_hash()` + `password_verify()`.
- [ ] Tokens/firmas comparados con `hash_equals()`, no con `==`.

```bash
grep $GREPX -E '==[^=]' --include=*.php .
grep $GREPX -E 'md5\(|sha1\(|strcmp\(' --include=*.php .
```
**Arreglo:** `===`, `password_verify()`, `hash_equals()`. Nunca comparar hashes
a mano.

## 3. Inyección SQL

- [ ] **Todas** las consultas usan sentencias preparadas / binding.
- [ ] Ninguna concatena entrada del usuario dentro del SQL.
- [ ] El usuario de BD tiene privilegios mínimos.

```bash
# PHP
grep $GREPX -E 'query\(.*\$|mysqli_query|->query\(|SELECT .*\.\s*\$|WHERE.*\$_' --include=*.php .
# Node / otros
grep $GREPX -E 'query\(\s*[`"'\''].*\$\{|\+\s*req\.' .
# Python
grep $GREPX -E 'execute\(.*%|execute\(.*\.format\(|execute\(f["'\'']' --include=*.py .
```
**Arreglo:** consultas parametrizadas / ORM en todas partes, sin excepción.

## 4. LFI / rutas construidas con entrada del usuario

- [ ] Ningún `include`/`require`/lectura de fichero usa un parámetro sin validar.
- [ ] Las opciones se mapean por lista blanca; se normaliza con `realpath`.

```bash
# PHP
grep $GREPX -E '(include|require)(_once)?\s*\(?.*\$_(GET|POST|REQUEST)' --include=*.php .
grep $GREPX -E 'file_get_contents\(.*\$_|fopen\(.*\$_|readfile\(.*\$_' --include=*.php .
# Node / Python
grep $GREPX -E 'readFile\(.*(req\.|params)|open\(.*request\.' .
```
**Arreglo:** lista blanca de ficheros; `realpath` + comprobar que queda dentro
del directorio permitido; rechazar `../`.

## 5. Subida de archivos → RCE

- [ ] Validas **tipo real** (magic bytes), no solo la extensión.
- [ ] Guardas fuera del document root o en zona que **no ejecuta** código.
- [ ] Renombras el fichero; prohíbes `.php*`, `.phtml`, `.htaccess`.
- [ ] En el dir de subidas: `AllowOverride None` y ejecución desactivada.

```bash
grep $GREPX -E 'move_uploaded_file|multer|FileUpload|request\.files|save\(.*upload' .
grep $GREPX -lE 'AllowOverride|\.htaccess' .
find . -name '.htaccess' -not -path '*/vendor/*'
```
**Arreglo:** ver apartado 6 de `vulnerabilidades-web.md`.

## 6. XXE (si procesas XML)

- [ ] Los parsers XML tienen **entidades externas y DTD deshabilitados**.

```bash
grep $GREPX -E 'simplexml_load|DOMDocument|loadXML|XMLReader|libxml|parseString|SAXParser|DocumentBuilder' .
```
**Arreglo:** desactivar entidades externas/DTD (`LIBXML_NOENT` **fuera**,
`load_external_dtd=false`, etc.). Mejor JSON si no necesitas XML.

## 7. SSRF (si pides URLs del lado servidor)

- [ ] Las funciones que piden una URL del usuario validan el destino.
- [ ] Bloqueas `127.0.0.1`, rangos privados y el endpoint de metadatos cloud.

```bash
grep $GREPX -E 'curl_exec|file_get_contents\(\$|fetch\(|requests\.(get|post)\(|http\.get\(|urlopen\(' .
```
**Arreglo:** lista blanca de dominios; validar la IP resuelta; sin acceso a red
interna; IMDSv2 en la nube.

## 8. CSRF

- [ ] Las acciones que cambian estado piden token anti-CSRF.
- [ ] Cookies con `SameSite=Lax`/`Strict`.
- [ ] Acciones críticas (cambio de pass/email) reautentican.

```bash
grep $GREPX -E 'csrf|SameSite|X-CSRF|_token|anti.?forgery' .
grep $GREPX -E 'setcookie\(|set_cookie|res\.cookie\(|Set-Cookie' .
```
**Arreglo:** token sincronizado + `SameSite` + pedir contraseña en lo sensible.

## 9. XSS (clásico en páginas/PWA — revísalo siempre)

- [ ] No inyectas datos del usuario en el DOM sin escapar.
- [ ] Evitas `innerHTML`/`document.write` con datos no confiables.

```bash
grep $GREPX -E 'innerHTML|outerHTML|document\.write|dangerouslySetInnerHTML|v-html|\|\s*raw' .
grep $GREPX -E 'echo \$_(GET|POST|REQUEST)|print \$_' --include=*.php .
```
**Arreglo:** escapado por contexto; `textContent` en vez de `innerHTML`;
plantillas con autoescape; cabecera **Content-Security-Policy**.

## 10. `.git` y secretos expuestos

- [ ] El `.git/` **no** se despliega en producción (bloqueado en el servidor).
- [ ] No hay claves/tokens/contraseñas en el repo ni en el historial.

```bash
grep $GREPX -E 'api[_-]?key|secret|password\s*=|token\s*=|BEGIN (RSA|PRIVATE)|AKIA[0-9A-Z]{16}' .
# ¿Está accesible el .git en una web ya publicada? (solo contra tu propio sitio)
curl -s -o /dev/null -w '%{http_code}\n' https://TU-DOMINIO/.git/HEAD
```
**Arreglo:** bloquear `.git` en Apache/Nginx; mover secretos a variables de
entorno; **rotar** cualquier credencial que ya se haya filtrado; `gitleaks`.

## 11. Credenciales en memoria / endurecimiento del servidor Windows

Aplica si tus apps corren en servidores Windows (p. ej. el controlador Omada o
el TPV sobre Windows):
- [ ] Cuentas de administración separadas; no inicies sesión con cuentas de
      dominio privilegiadas en equipos de cliente.
- [ ] Credential Guard / LSA Protection donde sea posible.
- [ ] EDR/antivirus activo y actualizado.

---

## Cabeceras y configuración (revisión rápida del sitio publicado)

```bash
curl -sI https://TU-DOMINIO | grep -iE 'strict-transport|content-security|x-frame|x-content-type|set-cookie|server|x-powered-by'
```
- [ ] HTTPS forzado (**HSTS**).
- [ ] **CSP** definida.
- [ ] `X-Frame-Options`/`frame-ancestors` (anti-clickjacking).
- [ ] `X-Content-Type-Options: nosniff`.
- [ ] Cookies con `Secure`, `HttpOnly`, `SameSite`.
- [ ] Oculta versiones (`Server`, `X-Powered-By`).

## Específico de tu stack

- **PWA / Hospitality Guest App:** XSS (apartado 9), CSP, validación en el
  servidor (no te fíes solo del cliente), control de acceso por habitación.
- **Android WebView:** `setJavaScriptEnabled` solo si hace falta; no exponer
  interfaces JS sensibles; cargar solo orígenes de confianza; validar la URL.
- **doelitcard / landings:** XSS, formularios con validación servidor, sin
  secretos en el JS del cliente.
- **Python/Tkinter (Doel Tools):** inyección de comandos (`os.system`,
  `subprocess(shell=True)`), rutas y backups; revisa:
  ```bash
  grep $GREPX -E 'os\.system|subprocess.*shell=True|eval\(|exec\(' --include=*.py .
  ```
