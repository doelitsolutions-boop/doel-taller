# Licencia v2 de Doel Taller (0.1.74) · guía para Doel

Cierra **SEC‑01** de la auditoría 0.1.72: el instalador ya no lleva ningún secreto con el que fabricar licencias, abrir el Panel de Desarrollador o descifrar copias de otros talleres.

## Qué cambia, en una frase

Antes: un secreto HMAC y una semilla TOTP iguales en todos los instaladores. Ahora: **la app solo lleva la clave pública de Doel**; la privada vive en tu PC. Cada taller recibe **un código DL2 firmado** (se pega una vez) que trae dentro **su propia semilla TOTP**, y de ella salen los códigos cortos por teléfono, que solo valen para ese equipo.

## Puesta en marcha (una sola vez, en tu PC)

```
node tools/licencia-v2.cjs claves
```

- Genera `~/.doel/licencia-doel.json` (par Ed25519 + clave maestra de semillas, permisos 600). **Haz copia fuera del PC**: sin ese fichero no se emiten licencias ni códigos. Si prefieres otra carpeta: variable `DOEL_LICENCIA_DIR`.
- Escribe la **clave pública** en `electron/licencia.cjs` (`CLAVE_PUBLICA_DOEL`). Después `npm run dist`: ese instalador es el que activa códigos DL2.
- `node tools/check-puente.cjs` avisa (paso 9) si una compilación sale sin clave pública. No falla el ritual, porque en desarrollo puede faltar; en el instalador que entregas no debe faltar.

## Por cada taller

1. El cliente te lee el **ID de equipo** (Ajustes → Sistema → Licencia, o la pantalla de activación): `AB12-CD34-EF56`. Es el mismo ID de siempre.
2. Emites el código:
   ```
   node tools/licencia-v2.cjs emitir --equipo AB12-CD34-EF56 --cliente "Taller X" --modulos crm,iso,kb,diagnostico --modo permanente
   ```
   Opciones: `--modulos todos | ninguno | lista`, `--modo permanente | mensual | anual`, `--caduca 2027-12-31` (fecha fija).
3. Se lo mandas por WhatsApp o correo (es largo: empieza por `DL2.`). Lo pega **una vez** en Ajustes → Licencia o en la pantalla de activación. Ya está activado con esos módulos.

## Por teléfono, después

| Quieres | Comando en tu PC | Qué teclea el cliente |
|---|---|---|
| Renovar (mensual +31 / anual +366; permanente confirma) | `node tools/licencia-v2.cjs codigo --equipo AB12-CD34-EF56` | 6 dígitos |
| Abrir el Panel de Desarrollador de ese equipo | el mismo `codigo` | 6 dígitos |
| Fijar módulos | `node tools/licencia-v2.cjs modulos --equipo AB12-CD34-EF56 --modulos crm,iso` | 12 dígitos `MMMMMM-TTTTTT` |
| Cambiar módulos sin teléfono | `emitir` otra vez con la lista nueva | pega el DL2 nuevo |
| Autenticador en el móvil para ese taller | `node tools/licencia-v2.cjs qr --equipo AB12-CD34-EF56` → enlace `otpauth://` | — (una entrada por cliente) |

Los códigos cortos valen ±60 segundos. Un código de un equipo **no vale en otro**: ya no existe un código universal.

## Instalaciones activadas antes de la 0.1.74

Siguen funcionando tal cual («activación heredada»): al arrancar se re‑sellan con la clave de instalación y Ajustes → Licencia muestra un aviso. Solo necesitan su DL2 **para volver a renovar o cambiar módulos por teléfono**; se emite igual, con su ID de equipo. El Panel de Desarrollador de una instalación heredada también pide su DL2 primero (tú tienes la herramienta).

## Qué protege y qué no

- Con el instalador en la mano ya **no** se fabrican licencias ni se abre el Panel Dev de nadie: no hay secreto que extraer.
- Un código DL2 robado solo sirve en el equipo para el que se emitió (va ligado al ID de equipo y firmado).
- El sello de integridad (v11) detecta ediciones a mano de módulos, modo y caducidad en la base.
- **Límite honesto**: quien tenga acceso total al PC del taller (clave de instalación + base) tiene la semilla de *ese* taller y puede calcular *sus* códigos cortos. Es inherente a cualquier activación offline: el verificador necesita el secreto. Antes ese mismo atacante tenía los de *todos* los talleres.
- La clave de las copias DOELBAK1 sigue en la app como constante derivada: hace falta para leer copias y textos cifrados antes de la 0.1.73. Las copias con frase propia (DOELBAK2) no dependen de ella; la puesta en marcha pide la frase.

## Ficheros

- `electron/licencia.cjs` · verificación DL2, semilla, códigos cortos, sello v11 (sin secretos).
- `tools/licencia-v2.cjs` · tu herramienta (no va en el instalador: `package.json` excluye `tools/**`).
- `tools/test-licencia-v2.cjs` · batería (43 comprobaciones) con claves de prueba en una carpeta temporal.
- Borrados: `tools/generar-licencia.cjs`, `codigo-instalacion.cjs`, `codigo-renovacion.cjs`, `test-licencia-modulos.cjs`.
- `.gitignore`: `licencia-doel.json` y `*.pem` nunca entran en el repo.
