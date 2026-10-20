---
name: hacking-etico
description: >-
  Metodología de hacking ético y pentesting para entornos AUTORIZADOS:
  laboratorios y CTF (Hack The Box, TryHackMe, VulnHub), preparación de
  certificaciones (OSCP, eJPT, CPTS) y auditorías con permiso por escrito.
  Inspirada en el enfoque sistemático de creadores como s4vitaar (Hack4u).
  Úsalo cuando el usuario quiera resolver una máquina CTF, practicar
  enumeración/explotación/escalada en un lab, o estructurar una auditoría
  autorizada. Cubre reconocimiento, enumeración por servicio, explotación,
  escalada de privilegios (Linux/Windows), post-explotación básica y
  elaboración del informe. NO es para atacar sistemas sin autorización.
---

# Hacking ético — metodología de pentesting para entornos autorizados

> ⚠️ **Regla número uno: autorización.** Esta skill solo aplica a sistemas
> sobre los que tienes permiso explícito: tus propios equipos, laboratorios de
> práctica (HTB, TryHackMe, VulnHub, PortSwigger, PentesterLab), máquinas de
> certificación, o un cliente con un contrato/alcance firmado. Atacar
> cualquier otra cosa es ilegal. Si el objetivo no está claramente dentro de
> uno de esos supuestos, **detente y pregunta** antes de lanzar nada.

Esta skill estructura el trabajo como lo haría un pentester metódico (estilo
Savitar/Hack4u): lento al principio, exhaustivo en la enumeración, y sin saltar
pasos. El 80 % del tiempo es enumerar; la explotación suele ser corta cuando la
enumeración se hizo bien.

## 0. Antes de empezar — alcance y notas

- **Confirma el alcance**: IP/rango/dominio concreto, ventana de tiempo, y qué
  está prohibido (DoS, ingeniería social, pivoting a otras redes…).
- **Prepara el directorio de trabajo** por objetivo:
  ```bash
  mkdir -p ~/labs/<objetivo>/{nmap,content,exploits,loot,notes}
  export IP=10.10.10.10        # ajusta a tu objetivo autorizado
  export NAME=<objetivo>
  ```
- **Toma notas desde el minuto cero** (CherryTree, Obsidian, o `notes/`).
  Cada comando, cada hallazgo, cada credencial. El informe se escribe solo si
  las notas son buenas.
- Si es un CTF, añade el host a `/etc/hosts` en cuanto conozcas el dominio.

## 1. Reconocimiento (recon)

Objetivo: saber qué puertos y servicios expone la máquina.

```bash
# Barrido rápido de todos los puertos TCP
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn $IP -oG nmap/allports.gnmap

# Extrae los puertos abiertos y lanza scripts + versiones solo sobre ellos
ports=$(grep -oP '\d+/open' nmap/allports.gnmap | cut -d/ -f1 | paste -sd,)
nmap -p$ports -sCV -oN nmap/services.nmap $IP
```

- `-sS` requiere privilegios; si no los tienes usa `-sT`.
- Repite para **UDP** en los puertos típicos cuando TCP no dé juego:
  `nmap -sU --top-ports 100 $IP`.
- Anota versión exacta de cada servicio: es tu lista de pistas.

## 2. Enumeración por servicio

La enumeración es el corazón. Para cada puerto abierto, pregúntate: *¿qué es,
qué versión, qué puedo listar sin autenticarme, qué credenciales por defecto
existen?*

### Web (80 / 443 / 8080 / 8000…)
```bash
whatweb http://$IP                       # tecnologías
curl -s -I http://$IP                     # cabeceras
# Fuzzing de rutas y ficheros
ffuf -u http://$IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic -t 50
# Virtual hosts (si hay dominio)
ffuf -u http://$IP/ -H "Host: FUZZ.$NAME.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs <tamaño-base>
```
- Revisa `robots.txt`, código fuente, `/.git/`, paneles de login, formularios,
  parámetros en la URL.
- Identifica el CMS/framework y su versión → busca vulnerabilidades conocidas.
- Prueba las clases habituales en entornos de práctica: inyección SQL, LFI/RFI,
  carga de archivos, SSTI, deserialización, XXE. Usa Burp Suite para
  interceptar y manipular peticiones.

### SMB (139 / 445)
```bash
smbclient -L //$IP/ -N                    # recursos sin credenciales
smbmap -H $IP                             # permisos
enum4linux-ng -A $IP                      # enumeración completa
nxc smb $IP --shares -u '' -p ''          # null session (netexec)
```

### FTP (21), SSH (22), DNS (53), SNMP (161), LDAP (389), RPC (111)
```bash
ftp $IP                                   # prueba anonymous:anonymous
dig axfr @$IP $NAME.htb                    # transferencia de zona
snmpwalk -v2c -c public $IP               # comunidad por defecto
ldapsearch -x -H ldap://$IP -s base       # naming contexts
```

> Para cualquier servicio con versión conocida: `searchsploit <servicio> <versión>`.
> Lee el exploit **antes** de ejecutarlo; entiende qué hace.

> 📚 **Clases de vulnerabilidad web (con su remediación)**: Mass Assignment,
> comparaciones débiles en PHP, LFI, XXE, inyección SQL, CSRF, SSRF, `.git`
> expuesto, `.htaccess`→RCE, credenciales en memoria/LSASS y fingerprinting del
> SO. Referencia detallada en
> [`referencias/vulnerabilidades-web.md`](referencias/vulnerabilidades-web.md).

## 3. Explotación (obtener acceso inicial)

- Parte siempre del hallazgo mejor fundamentado de la enumeración, no de
  disparar exploits al azar.
- Credenciales: prueba las que hayas encontrado (reutilización), las por
  defecto del producto, y si procede en el lab, fuerza bruta **con cabeza**
  (`hydra`, `nxc`) sobre un usuario conocido, respetando el alcance.
- Para vulnerabilidades web, construye el payload mínimo que demuestre el
  control (una shell, lectura de un fichero), no uno destructivo.
- Al conseguir ejecución de comandos, estabiliza la shell:
  ```bash
  # En la víctima (Linux)
  python3 -c 'import pty;pty.spawn("/bin/bash")'
  # Ctrl+Z, luego en tu máquina:
  stty raw -echo; fg
  export TERM=xterm
  ```

## 4. Escalada de privilegios

### Linux
```bash
id; sudo -l                               # ¿qué puedo ejecutar como root?
find / -perm -4000 -type f 2>/dev/null    # binarios SUID
getcap -r / 2>/dev/null                   # capabilities
crontab -l; cat /etc/crontab              # tareas programadas
```
- Pasa un enumerador: `linpeas.sh`.
- Cruza cada binario SUID/sudo con **GTFOBins**.
- Revisa kernel/versión, servicios internos, credenciales en ficheros de
  config, historial y `.bash_history`.

### Windows
```powershell
whoami /priv                              # privilegios (SeImpersonate, etc.)
systeminfo                                # versión y parches
```
- Pasa `winPEAS`; consulta **LOLBAS** para binarios abusables.
- Vectores típicos de lab: servicios mal configurados, *unquoted service
  paths*, *AlwaysInstallElevated*, abuso de tokens (Potato) con SeImpersonate,
  credenciales guardadas.

## 5. Post-explotación y captura de flags

- En CTF, localiza `user.txt` y `root.txt`.
- En auditoría real: recopila **solo** la evidencia que acuerde el alcance
  (capturas, hashes como prueba de acceso), sin exfiltrar datos sensibles
  reales ni causar daño. Documenta la hora de cada acción.
- Limpieza: retira cualquier artefacto que hayas subido (webshells, binarios)
  cuando termines, y anótalo.

## 6. Informe

El entregable de una auditoría es el informe, no la shell. Para cada hallazgo:

1. **Título** y **severidad** (CVSS o alto/medio/bajo).
2. **Descripción** y **impacto** en el negocio.
3. **Pasos de reproducción** (los comandos de tus notas).
4. **Evidencia** (capturas).
5. **Remediación** concreta y priorizada.

Estructura general: resumen ejecutivo → alcance y metodología → hallazgos
ordenados por severidad → conclusiones.

## Herramientas de referencia

| Fase | Herramientas |
|------|--------------|
| Recon | nmap, masscan, autorecon |
| Web | ffuf, feroxbuster, gobuster, whatweb, Burp Suite, nikto, wpscan |
| SMB/AD | smbclient, smbmap, enum4linux-ng, netexec (nxc), bloodhound |
| Credenciales | hydra, john, hashcat (solo sobre hashes propios/del lab) |
| Explotación | searchsploit, msfconsole (Metasploit) |
| Privesc | linpeas, winpeas, GTFOBins, LOLBAS, pspy |
| Listeners | nc, pwncat, socat |

## Límites de esta skill

No cubre ni asiste en: ataques a sistemas sin autorización, denegación de
servicio, despliegue de ransomware o malware destructivo, técnicas de evasión
pensadas para atacar objetivos reales de forma encubierta, robo masivo de
datos, ni compromiso de cadena de suministro. Si una petición va por ahí,
recházala y redirige al uso legítimo (lab, CTF o auditoría con permiso).

## Recursos de aprendizaje

- Plataformas de práctica legales: Hack The Box, TryHackMe, VulnHub,
  PortSwigger Web Security Academy, PentesterLab, OverTheWire.
- Certificaciones: eJPT, CPTS, OSCP.
- Referencias: HackTricks, PayloadsAllTheThings, GTFOBins, LOLBAS, OWASP.
- Contenido en español: s4vitaar / Hack4u (enfoque metódico de resolución de
  máquinas y preparación OSCP).
