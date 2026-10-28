# 🐚 Bash Scripting para Administración de Sistemas Linux

![Bash](https://img.shields.io/badge/Bash-4EAA25?logo=gnubash&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?logo=linux&logoColor=black)
![Red Hat](https://img.shields.io/badge/RHEL%20%2F%20CentOS%20%2F%20Rocky-EE0000?logo=redhat&logoColor=white)
![ShellCheck](https://img.shields.io/badge/ShellCheck-lint-blue)

Colección de **scripts de Bash orientados a la administración real de servidores Linux**: monitoreo, seguridad de cuentas, respaldos, gestión de procesos, logging centralizado e inventario. Cada script resuelve una tarea repetitiva que un administrador de sistemas hace a mano todos los días, y está pensado para ejecutarse **de forma desatendida con `cron`** y notificar por correo.

> Construido a partir del curso *Linux Bash Shell Scripting through Real Life Examples* (Imran Afzal) y de la preparación para la certificación **RHCSA (RHEL 10)**. Varios scripts son versiones propias que amplían los ejemplos del curso con validaciones y buenas prácticas de producción.

---

## 📋 Contenido

- [Qué demuestra este repositorio](#-qué-demuestra-este-repositorio)
- [Catálogo de scripts](#-catálogo-de-scripts)
- [Ejemplos destacados](#-ejemplos-destacados)
- [Buenas prácticas aplicadas](#-buenas-prácticas-aplicadas)
- [Estructura del repositorio](#-estructura-del-repositorio)
- [Uso](#-uso)
- [Automatización con cron](#-automatización-con-cron)
- [Requisitos](#-requisitos)
- [Próximos pasos](#-próximos-pasos)

---

## 🎯 Qué demuestra este repositorio

| Área | Habilidades |
| :--- | :--- |
| **Fundamentos de Bash** | Variables, sustitución de comandos `$(...)`, aritmética `$(( ))`, expansión de parámetros (`${var%.txt}`), `read`, `case`, `for`, `while`, funciones |
| **Control de flujo y errores** | `test` / `[ ]`, códigos de salida (`$?`, `exit N`), operadores `&&` / `\|\|`, validación de argumentos |
| **Procesamiento de texto** | `grep -E`, `awk`, `sed -i`, `cut`, `sort \| uniq`, `tail -f`, pipelines encadenados |
| **Administración de usuarios** | `useradd` / `usermod -L`, `lastlog`, `last`, `/etc/passwd` y `/etc/shadow`, `getent` |
| **Sistemas de archivos** | `df` / `du`, `find -mtime -exec`, `tar` + `gzip`, permisos, detección de directorios huérfanos |
| **Procesos** | `ps`, `pgrep` / `pkill`, `kill` vs. `kill -9`, `xargs` |
| **Red** | `ping` con *timeout*, `scp` a múltiples hosts, autenticación por llaves SSH |
| **Automatización** | `cron` con rutas absolutas y logs, `nohup`, notificación con `mail` |
| **Calidad** | `shellcheck`, `set -euo pipefail`, entrecomillado de variables, cabeceras estándar |

---

## 📦 Catálogo de scripts

### 🔍 Monitoreo

| Script | Qué hace |
| :--- | :--- |
| `check_disk.sh` | Alerta si alguna partición supera un umbral de uso (excluye `tmpfs`/`devtmpfs`) |
| `ping_hosts.sh` | Comprueba la conectividad de una lista de servidores leída de un archivo e informa `OK` / `NOT OK` |
| `count_files.sh` | Verifica que un proceso haya generado la cantidad esperada de archivos |
| `admin_report.sh` | Reporte rápido de salud: carga, memoria, disco y top de procesos (`top -b -n 1`) |

### 📜 Logs

| Script | Qué hace |
| :--- | :--- |
| `pull_errors.sh` | Extrae errores, avisos y fallos de `/var/log/messages` |
| `log_tail.sh` | Vigila el log **en tiempo real** y guarda las líneas con palabras clave (`error`, `fail`, `refused`, `invalid`...) |
| `log_alert.sh` | Cada 15 min envía por correo los mensajes filtrados, deduplicados con `sort \| uniq`, y limpia el archivo |

### 👤 Usuarios y seguridad

| Script | Qué hace |
| :--- | :--- |
| `create_user.sh` | Alta interactiva de cuentas: valida usuario y UID duplicados y obliga a registrar una descripción (GECOS) |
| `disable_inactive_users.sh` | Bloquea (no borra) las cuentas sin acceso en más de 90 días, excluyendo las de sistema |
| `check_orphan_dirs.sh` | Detecta directorios en `/home` sin usuario asociado, un riesgo de seguridad típico tras un `userdel` sin `-r` |
| `users_logged_today.sh` | Lista los usuarios que iniciaron sesión hoy o en una fecha indicada |

### 💾 Archivos y respaldos

| Script | Qué hace |
| :--- | :--- |
| `backup_fs.sh` | Respalda `/etc` y `/var` en un `.tar.gz` con fecha, verifica que se generó hoy y lo copia a un servidor remoto |
| `delete_old_files.sh` | Elimina (o renombra a `.old`) los archivos con más de N días de antigüedad |
| `rename_ext.sh` | Cambia la extensión de archivos en lote con expansión de parámetros |
| `copy_to_hosts.sh` | Distribuye un archivo a múltiples servidores con `scp` |

### ⚙️ Procesos

| Script | Qué hace |
| :--- | :--- |
| `kill_process.sh` | Localiza procesos por patrón y los termina de forma ordenada (`SIGTERM`) |

### 🗂️ Utilidades

| Script | Qué hace |
| :--- | :--- |
| `inventory.sh` | Mini CMDB por consola: alta y baja de equipos (hostname, IP, descripción), con validación de duplicados y un menú `case` |
| `new_script.sh` | **Generador de scripts**: crea un archivo con la cabecera estándar (autor, fecha, descripción), le da permisos y lo abre en el editor |

---

## ✨ Ejemplos destacados

### Alerta de espacio en disco

Versión en una línea con `awk`, robusta frente a coincidencias parciales:

```bash
#!/usr/bin/env bash
# Description: Alerta si alguna partición supera el umbral de uso
set -euo pipefail

THRESHOLD="${1:-90}"

df -hP | awk -v t="$THRESHOLD" 'NR > 1 && $1 !~ /tmpfs/ && 0+$5 >= t {
    printf "ALERTA: %s al %s (%s)\n", $6, $5, $1
}'
```

### Bloqueo de usuarios inactivos

Solo cuentas humanas (UID ≥ 1000), bloqueo reversible y registro de lo que se hizo:

```bash
#!/usr/bin/env bash
# Description: Bloquea cuentas sin inicio de sesión en más de N días
set -euo pipefail

DAYS="${1:-90}"
LOG="/var/log/disabled_users.log"

lastlog -b "$DAYS" | tail -n +2 | awk '{print $1}' | while read -r user; do
    uid=$(id -u "$user" 2>/dev/null) || continue
    if (( uid >= 1000 )); then
        usermod -L "$user"
        echo "$(date '+%F %T') bloqueado: $user" >> "$LOG"
    fi
done
```

### Monitoreo de logs en tiempo real

```bash
#!/usr/bin/env bash
# Description: Filtra en vivo /var/log/messages por palabras clave
OUT="/tmp/filtered_messages"
PATTERN='refused|invalid|error|fail|lost|shut down|offline'

tail -F -n0 /var/log/messages | while read -r line; do
    grep -Eiq "$PATTERN" <<< "$line" && echo "$line" >> "$OUT"
done
```

### Respaldo verificado con fecha

```bash
#!/usr/bin/env bash
# Description: Respalda /etc y /var y verifica el archivo generado
set -euo pipefail

BACKUP="/tmp/backup-$(date +%F).tar.gz"

tar czf "$BACKUP" /etc /var 2>/dev/null || true

if [[ -s "$BACKUP" ]] && find "$BACKUP" -mtime -1 -type f | grep -q .; then
    echo "Respaldo creado: $BACKUP"
    # scp "$BACKUP" backup@srv-respaldos:/respaldos/
else
    echo "Error: el respaldo no se generó" >&2
    exit 1
fi
```

---

## ✅ Buenas prácticas aplicadas

- **Cabecera estándar** en cada script: shebang, autor, fecha, descripción y fecha de modificación.
- **`set -euo pipefail`** para fallar pronto ante errores, variables sin definir o pipelines rotos.
- **Variables siempre entrecomilladas** (`"$var"`) para no romper con espacios o globbing.
- **Códigos de salida significativos**: `exit 0` en éxito y `exit 1` (o mayor) en error, para que `cron` u otros scripts puedan reaccionar.
- **Patrones anclados** (`grep "^$user:" /etc/passwd`) para evitar falsos positivos (`test` frente a `test123`).
- **Probar antes de destruir**: los scripts que borran se prueban primero con `-exec ls -l` o `echo`.
- **Bloquear en lugar de borrar** cuentas, y registrar cada acción en un log.
- **Rutas absolutas** en todo lo que corre en `cron`, porque su `PATH` es mínimo.
- **Herramientas modernas**: `grep -E` en lugar del obsoleto `egrep`, `pgrep`/`pkill` en lugar de `ps | grep | awk`, `$(...)` en lugar de comillas invertidas.
- **Validación con [ShellCheck](https://www.shellcheck.net/)** antes de cada commit.

---

## 🗂️ Estructura del repositorio

```
bash-scripting/
├── monitoring/       # check_disk.sh, ping_hosts.sh, count_files.sh, admin_report.sh
├── logs/             # pull_errors.sh, log_tail.sh, log_alert.sh
├── users/            # create_user.sh, disable_inactive_users.sh, check_orphan_dirs.sh, users_logged_today.sh
├── backup/           # backup_fs.sh, delete_old_files.sh, rename_ext.sh, copy_to_hosts.sh
├── processes/        # kill_process.sh
├── utils/            # inventory.sh, new_script.sh
├── cron/             # ejemplos de crontab
└── README.md
```

---

## 🚀 Uso

```bash
git clone https://github.com/jolucoga/bash-scripting.git
cd bash-scripting
chmod u+x */*.sh

# Ejemplos
./monitoring/check_disk.sh 85
./monitoring/ping_hosts.sh hosts.txt
sudo ./users/disable_inactive_users.sh 90
```

> Los scripts que leen logs del sistema o modifican cuentas requieren privilegios de `root` (`sudo`).

---

## ⏰ Automatización con cron

```cron
# Espacio en disco cada hora, alerta por correo
0 * * * *    /opt/scripts/monitoring/check_disk.sh 90 | mail -s "Disk status $(hostname)" admin@empresa.com

# Reporte de errores del log cada 15 minutos
*/15 * * * * /opt/scripts/logs/log_alert.sh >> /var/log/log_alert.log 2>&1

# Respaldo diario a las 2:00
0 2 * * *    /opt/scripts/backup/backup_fs.sh >> /var/log/backup.log 2>&1

# Bloqueo de cuentas inactivas, lunes a las 6:00
0 6 * * 1    /opt/scripts/users/disable_inactive_users.sh 90
```

> En distribuciones con systemd, estas tareas también pueden migrarse a **systemd timers** (`OnCalendar=`), que dan logs centralizados en `journalctl` y ejecución persistente si el equipo estuvo apagado.

---

## 🔧 Requisitos

- Bash 4 o superior
- Probado en distribuciones de la familia Red Hat (RHEL, CentOS Stream, Rocky, Alma). La mayoría funciona igual en Debian/Ubuntu (allí el log general es `/var/log/syslog`).
- Paquetes opcionales: `sysstat` (`iostat`), `mailx` y un relay SMTP para las notificaciones por correo.
- Llaves SSH configuradas para los scripts que copian a hosts remotos.

---

## 🛣️ Próximos pasos

- [ ] Pruebas automatizadas con [bats-core](https://github.com/bats-core/bats-core)
- [ ] Pipeline de GitHub Actions que ejecute `shellcheck` en cada push
- [ ] Opciones con `getopts` (`-h`, `-v`, `-n` *dry-run*) en todos los scripts
- [ ] Unidades `.service` + `.timer` de systemd como alternativa a cron
- [ ] Llevar estos mismos flujos a playbooks de **Ansible** para gestionar flotas de servidores

---

## 👤 Autor

**Luis Corona** — Administración de sistemas Linux · Automatización · Preparación RHCSA

Si este repositorio te resulta útil o quieres hablar de automatización en Linux, no dudes en contactarme.
