# Clases de vulnerabilidad — referencia

> ⚠️ **Solo entornos autorizados.** Todo lo de aquí es material educativo para
> labs, CTF y auditorías con permiso. Cada apartado incluye **cómo se corrige**:
> esa es la parte que de verdad importa. Entender el ataque sirve para
> defenderse de él.

Índice:

1. [Mass Assignment (registrarse como admin)](#1-mass-assignment)
2. [Comparaciones débiles en PHP (bypass de login)](#2-comparaciones-débiles-en-php)
3. [Credenciales en memoria / LSASS (Mimikatz)](#3-credenciales-en-memoria--lsass)
4. [Detectar el sistema operativo por mayúsculas/minúsculas](#4-detectar-el-so-por-la-ruta)
5. [Directorio `.git` expuesto](#5-git-expuesto)
6. [`.htaccess` + subida de archivos → RCE](#6-htaccess--subida-de-archivos--rce)
7. [XXE (XML External Entity)](#7-xxe)
8. [LFI (Local File Inclusion)](#8-lfi)
9. [Inyección SQL (incluida la ciega)](#9-inyección-sql)
10. [CSRF (Cross-Site Request Forgery)](#10-csrf)
11. [SSRF (Server-Side Request Forgery)](#11-ssrf)

---

## 1. Mass Assignment

**Qué es.** El servidor crea un objeto (p. ej. un usuario) volcando
*directamente* todos los campos que llegan en la petición a las propiedades del
modelo. Si el formulario de registro solo pide `usuario` y `password`, pero el
modelo tiene además un campo `role` o `is_admin`, un atacante puede **añadir ese
campo a mano** en la petición y asignarse privilegios que el formulario nunca
mostró.

**Cómo ocurre.** El registro envía normalmente:
```json
{ "username": "ana", "password": "1234" }
```
El atacante intercepta con Burp y añade el campo extra:
```json
{ "username": "ana", "password": "1234", "role": "admin" }
```
Si el backend hace algo como `User.create(request.body)` sin filtrar, el nuevo
usuario queda como administrador.

**Cómo se detecta.** Compara los campos del modelo (si tienes el código o puedes
inferirlos) con los del formulario; prueba a añadir campos sospechosos
(`role`, `admin`, `is_admin`, `verified`, `balance`) y observa el resultado.

**Cómo se corrige (lado servidor).**
- **Lista blanca** de campos aceptados (allow-list), nunca asignación masiva
  ciega. Ejemplos: `fillable`/`guarded` en Laravel, `strong parameters`
  (`params.permit(:username, :password)`) en Rails, DTOs explícitos en
  Spring/.NET.
- Separa el modelo de la entrada: mapea solo los campos permitidos del request
  al objeto de dominio.
- El rol/privilegio se asigna por lógica de servidor, nunca desde datos del
  cliente.

---

## 2. Comparaciones débiles en PHP

**Qué es.** PHP, con el operador `==` (comparación no estricta), hace
*type juggling*: convierte tipos antes de comparar. Eso produce igualdades
sorprendentes que pueden saltarse una comprobación de login o de token.

**Cómo ocurre.** Un código que valide así es vulnerable:
```php
if ($hash == $input) { /* acceso */ }        // == en vez de ===
```
Casos clásicos de type juggling: ciertas cadenas que empiezan por `0e` y
siguen con dígitos se tratan como notación científica y equivalen a `0`, de
modo que dos hashes distintos "con forma de 0e…" pueden compararse como
iguales; y una comparación entre cadena y número puede evaluar de forma
inesperada. También afecta a tokens de recuperación o firmas comparadas con `==`.

**Cómo se corrige.**
- Usa **comparación estricta** `===` (compara tipo y valor).
- Para verificar contraseñas: `password_hash()` al guardar y
  `password_verify()` al comprobar — nunca comparar hashes a mano.
- Para comparar secretos/tokens/firmas: `hash_equals()` (además es de tiempo
  constante, evita *timing attacks*).
- Valida y castea el tipo de cada entrada antes de operar con ella.

---

## 3. Credenciales en memoria / LSASS

**Qué es.** En Windows, el proceso **LSASS** (`lsass.exe`) gestiona la
autenticación y mantiene en memoria material de credenciales de las sesiones
activas. La duda habitual: *"si las credenciales están en memoria, ¿no están a
la vista?"* No: están cifradas. Pero **las claves para descifrarlas viven en el
propio proceso LSASS**, así que quien tenga privilegios suficientes sobre esa
memoria puede recuperarlas. Herramientas como **Mimikatz** automatizan
exactamente eso.

**Requisito del ataque.** No es "gratis": hace falta **privilegio de
administrador local / SeDebugPrivilege** sobre la máquina. Es una técnica de
**post-explotación** (ya estás dentro y eres admin), no de acceso inicial. Por
eso en una cadena de ataque aparece *después* de la escalada de privilegios.

**Por qué importa.** Explica el movimiento lateral: de un equipo comprometido
se extraen credenciales (incluidas las de otros usuarios que iniciaron sesión
ahí) y se reutilizan contra otros sistemas del dominio.

**Cómo se defiende (lo esencial).**
- **Credential Guard** (aísla los secretos de LSA en un entorno virtualizado,
  VBS) para que ni un admin pueda leerlos de LSASS.
- **LSA Protection** (`RunAsPPL`): protege LSASS como proceso ligero y dificulta
  el volcado.
- **Principio de mínimo privilegio**: que las cuentas con derechos altos no
  inicien sesión en equipos de baja confianza (así no dejan sus credenciales
  en esa memoria). Usa cuentas separadas para administración (modelo de niveles).
- **Deshabilitar WDigest** (no almacenar contraseñas en claro) — por defecto ya
  en sistemas modernos.
- **EDR/antimalware** que detecte el acceso a la memoria de LSASS.
- Rotación de credenciales y gestión de cuentas privilegiadas (PAM/LAPS).

---

## 4. Detectar el SO por la ruta

**Qué es.** Un truco de reconocimiento. Los sistemas de archivos de **Windows no
distinguen mayúsculas/minúsculas**, los de **Linux sí**.

**Cómo ocurre.** Pides un recurso existente cambiando una letra de la ruta a
mayúscula, p. ej. `/index.PHP` o `/Images/logo.png`:
- Si el servidor responde **igual** (200, mismo contenido) → probablemente
  **Windows**.
- Si responde **404** → probablemente **Linux**.

Es una pista rápida para orientar el resto de la enumeración (rutas, payloads,
escalada). Conviene confirmarla con otras señales (cabeceras, banners de nmap,
comportamiento de los servicios).

**Nota defensiva.** No es una "vulnerabilidad" explotable en sí, sino fuga de
información por *fingerprinting*. Se mitiga reduciendo huellas: páginas de error
genéricas, ocultar versiones en cabeceras, normalizar respuestas.

---

## 5. `.git` expuesto

**Qué es.** Si el directorio `.git/` del proyecto queda accesible por el
servidor web, no es "un archivo suelto": es **el repositorio entero con todo su
historial**. Eso incluye versiones antiguas de ficheros y credenciales que
alguien creyó haber borrado (siguen en commits previos).

**Cómo ocurre.** Se despliega el código con un `git clone`/`git pull` en la raíz
web y nunca se bloquea el acceso a `.git/`. Un atacante que detecta
`http://sitio/.git/` puede reconstruir el repo en local con herramientas como
**git-dumper** y leer todo el código fuente e historial.

**Cómo se corrige.**
- **No despliegues el `.git/` en producción.** Usa artefactos de build o
  `git archive`, o excluye el directorio del document root.
- Bloquea el acceso en el servidor (Apache/Nginx): denegar `location ~ /\.git`.
- Si ya se filtraron secretos: **rótalos** (cambiar claves/tokens), porque
  reescribir el historial no basta si ya se expusieron.
- Revisa el historial con herramientas de detección de secretos (gitleaks,
  trufflehog) antes de publicar.

---

## 6. `.htaccess` + subida de archivos → RCE

**Qué es.** Una funcionalidad de subida mal filtrada permite subir un
**`.htaccess`** (en Apache con `AllowOverride` activo). Ese fichero puede
indicar al servidor que **interprete como PHP** extensiones que normalmente son
inofensivas. Así, un archivo con extensión "permitida" acaba ejecutándose como
código → ejecución remota de comandos (RCE).

**Cómo ocurre.** El filtro de subida solo mira la extensión y bloquea `.php`,
pero permite otras. El atacante sube un `.htaccess` que asocia, por ejemplo, una
extensión tolerada al handler de PHP, y luego sube su webshell con esa
extensión. El servidor la ejecuta.

**Cómo se corrige.**
- **Lista blanca** de extensiones y validación del **tipo real** del contenido
  (magic bytes), no solo de la extensión.
- Guardar los archivos subidos **fuera del document root** o en un dominio/CDN
  que **no ejecute código**, y servirlos con `Content-Disposition`.
- Renombrar los ficheros subidos con un nombre generado por el servidor.
- Desactivar la ejecución en el directorio de subidas (`php_admin_flag
  engine off`, o servir estáticos desde Nginx sin pasar por el intérprete) y
  **no permitir `.htaccess`** ahí (`AllowOverride None`).
- Prohibir explícitamente la subida de `.htaccess`, `.phtml`, `.php*`, etc.

---

## 7. XXE

**Qué es.** *XML External Entity*. Cuando una aplicación procesa XML y su
parser tiene habilitadas las **entidades externas**, un atacante define una
entidad que apunta a un recurso del servidor; al procesar el XML, el servidor
**incluye el contenido de ese recurso** en la respuesta, exponiendo ficheros
internos. También puede derivar en SSRF o denegación de servicio.

**Cómo ocurre (ejemplo educativo).** El atacante envía un XML con una entidad
externa que referencia un fichero local; si el parser la resuelve, el
contenido del fichero acaba reflejado donde la app imprime el dato. El esquema
típico declara un `DOCTYPE` con una `ENTITY` `SYSTEM` que apunta a una ruta
local, y luego usa esa entidad en el cuerpo del documento.

**Cómo se corrige.**
- **Deshabilitar las entidades externas y los DTD** en el parser (la mitigación
  principal). En cada lenguaje hay flags para ello (p. ej. desactivar
  `external-general-entities`, `load_external_dtd`, `DOCTYPE`).
- Preferir formatos más simples (JSON) cuando no se necesite XML.
- Usar librerías/configuraciones "seguras por defecto" y mantenerlas al día.
- Validar y limitar el tamaño/estructura del XML aceptado.

---

## 8. LFI

**Qué es.** *Local File Inclusion*. Un parámetro de la URL controla qué fichero
carga/incluye el servidor. Si no se valida, el atacante manipula la ruta para
leer **ficheros internos** que nunca deberían ser públicos (configuración,
credenciales, logs). En PHP, según la configuración, puede escalar a ejecución
de código (*LFI to RCE* vía log poisoning, wrappers, sesiones, etc.).

**Cómo ocurre.** Una URL como `pagina.php?file=inicio` incluye `inicio.php`.
El atacante prueba a recorrer directorios (`../`) para salir del directorio
previsto y apuntar a ficheros del sistema. Si la app concatena el parámetro a
una ruta sin control, lo carga.

**Cómo se corrige.**
- **No uses entrada del usuario para construir rutas.** Mapea las opciones a una
  **lista blanca** fija (p. ej. un `switch`/diccionario de páginas permitidas).
- Si inevitablemente hay un nombre de fichero, **normaliza** la ruta
  (`realpath`) y verifica que queda **dentro** del directorio permitido; rechaza
  `../`, bytes nulos y rutas absolutas.
- Desactiva wrappers peligrosos y `allow_url_include` en PHP.
- Mínimo privilegio del usuario del servicio web sobre el sistema de archivos.

---

## 9. Inyección SQL

**Qué es.** En lugar de un texto normal, el atacante introduce en un campo un
fragmento que la base de datos interpreta como **parte de la consulta**. Si la
app concatena la entrada a la SQL, puede saltarse el login, leer datos de otras
tablas o, en la variante **ciega**, deducir información **carácter a carácter**
según cómo responda la aplicación (contenido o tiempo).

**Cómo ocurre.** Un login que arma la consulta concatenando
(`"... WHERE user='" + input + "'"`) permite que una entrada con comillas y un
operador lógico altere la condición y devuelva verdadero siempre, entrando sin
conocer la contraseña. En SQLi ciega no se ve el dato directamente: se hacen
preguntas booleanas o basadas en retardo y se infiere cada carácter por la
respuesta. Herramientas como `sqlmap` automatizan la explotación **en labs**.

**Cómo se corrige.**
- **Consultas parametrizadas / sentencias preparadas** (binding de parámetros).
  Es la defensa real y definitiva: los datos nunca se mezclan con el código SQL.
- Usa un **ORM** o *prepared statements* en todas las consultas, sin excepción.
- **Mínimo privilegio** del usuario de BD (que la app no pueda `DROP`, leer
  tablas ajenas, etc.).
- Validación/normalización de entrada como capa adicional (no como única
  defensa). Evitar mensajes de error detallados hacia el cliente.

---

## 10. CSRF

**Qué es.** *Cross-Site Request Forgery*. La víctima, con su sesión abierta en
una web, hace clic en un enlace o visita una página maliciosa que **lanza en su
nombre** una acción en esa web (cambiar su contraseña, su email, hacer una
transferencia…), sin que se entere. El navegador adjunta automáticamente las
cookies de sesión, así que la petición parece legítima.

**Cómo ocurre.** Si una acción sensible se ejecuta solo con una petición
(p. ej. un formulario de cambio de contraseña) y el servidor no comprueba que
la petición viene de su propia web, una página externa puede enviar esa misma
petición mientras la víctima está autenticada.

**Cómo se corrige.**
- **Token anti-CSRF** por sesión/petición (sincronizado), que el servidor
  valida en cada acción que cambie estado.
- Cookies con atributo **`SameSite`** (`Lax` o `Strict`) para que no viajen en
  peticiones cross-site.
- Reautenticar o pedir la contraseña actual en acciones críticas (cambio de
  contraseña/email).
- Para APIs: exigir cabeceras personalizadas/CORS correcto; comprobar
  `Origin`/`Referer` como capa extra.

---

## 11. SSRF

**Qué es.** *Server-Side Request Forgery*. Se engaña a la aplicación para que
sea **ella** quien realice una petición de red a un destino elegido por el
atacante — incluidos **servicios internos** a los que este nunca llegaría desde
fuera (metadatos de la nube, paneles de administración internos, otros hosts de
la red privada).

**Cómo ocurre.** Una funcionalidad que recibe una URL (previsualizar un enlace,
importar una imagen, un webhook) y la solicita desde el servidor. Si no se
restringe el destino, el atacante apunta a direcciones internas
(`127.0.0.1`, rangos privados, endpoints de metadatos del proveedor cloud) y
obtiene respuestas que no deberían ser accesibles.

**Cómo se corrige.**
- **Lista blanca** de destinos/dominios permitidos; prohíbe direcciones
  internas, de loopback y de enlace local (incluido el endpoint de metadatos).
- Resolver y validar la IP de destino **tras** la resolución DNS (evitar
  *DNS rebinding*) y volver a comprobar en redirecciones.
- Aislar en red el servicio que hace las peticiones salientes (egress
  controlado, sin acceso a la red de administración).
- En la nube, exigir IMDSv2 / deshabilitar el acceso al servicio de metadatos
  desde cargas que no lo necesiten.
- No devolver al cliente la respuesta cruda de la petición interna.
