# Procedimientos de Administración, Copias de Seguridad y Juego de Pruebas

Este documento recoge las políticas de copias de seguridad, los protocolos de recuperación ante desastres, los criterios de rendimiento y disponibilidad, y el juego de pruebas completo (automatización, contingencia, monitorización y seguridad) que valida la solución.

**Índice**

1. [Copias de seguridad y recuperación ante desastres](#1-copias-de-seguridad-y-recuperación-ante-desastres)
2. [Criterios de rendimiento y disponibilidad](#2-criterios-de-rendimiento-y-disponibilidad)
3. [Juego de pruebas integral de validación (RA2)](#3-juego-de-pruebas-integral-de-validación-ra2)
4. [Registro de evidencias](#4-registro-de-evidencias)

---

## 1. Copias de Seguridad y Recuperación ante Desastres

### 1.1. Alcance de las copias

| Elemento | Qué se copia | Mecanismo | Destino |
|---|---|---|---|
| Contenedores LXC (`F-NCC`, `W-NCC`, `LDAP-NCC`, `B-NCC`) | Disco completo del contenedor | `vzdump` en modo `snapshot` con compresión `zstd` | `/var/lib/vz/dump/` en `pvexx` |
| Configuración del router (`pvexx`) | Red, reglas `iptables` persistentes, DNS/DHCP | Copia comprimida de los ficheros de configuración | `/root/backup-config/` en `pvexx` |
| Base de datos (`B-NCC`) | Esquema y datos de `bcn_educacio` | Volumen Docker `db_data` y exportación lógica `mysqldump` | `/root/db/` en `B-NCC` |

> **Nota:** sustituye `<ID_LDAP>` por el identificador real del contenedor `LDAP-NCC` en los comandos de esta sección. Debe incluirse en la copia, ya que el portal depende de él para autenticar.

### 1.2. Mecanismos de respaldo implementados

1. **Snapshots y copias de seguridad de contenedores LXC (`vzdump`):**
   * Instantánea previa a cambios críticos de red o configuración:
     ```bash
     pct snapshot 105 pre_actualizacion_ldap
     ```
   * Copia de seguridad completa comprimida (`zstd`) de todos los servidores:
     ```bash
     vzdump 104 105 106 <ID_LDAP> --mode snapshot --compress zstd --storage local
     ```

2. **Copia de la configuración del router (`pvexx`):**
   ```bash
   mkdir -p /root/backup-config
   tar czf /root/backup-config/pvexx-config-$(date +%F).tar.gz \
       /etc/network/interfaces /etc/dnsmasq.conf /etc/sysctl.conf
   ```

3. **Persistencia y respaldo lógico de la base de datos (`B-NCC`):**
   * Persistencia en disco mediante el volumen Docker `db_data` (`/var/lib/mysql`), junto con los scripts reproducibles `init.sql` e `inserts.sql` (2.710 registros limpios).
   * Exportación lógica bajo demanda. La contraseña **no se escribe en el comando**: se lee de un fichero de variables con permisos restringidos.
     ```bash
     # Una sola vez: crear el fichero con las credenciales (solo legible por root)
     install -m 600 /dev/null /root/.db.env
     echo 'MYSQL_PWD=<contraseña_del_usuario_de_backup>' > /root/.db.env

     # Exportación
     docker exec --env-file /root/.db.env mysql_bcn_db \
         mysqldump -u user_bcn bcn_educacio > /root/db/backup_bcn_$(date +%F).sql
     ```
   * **Copia fuera del nodo:** una copia que reside en el mismo equipo que los datos no protege ante la pérdida de ese equipo. Tras cada exportación se copia a `pvexx`:
     ```bash
     scp /root/db/backup_bcn_$(date +%F).sql bchecker@10.10.20.1:/root/backup-db/
     ```

### 1.3. Protocolo de recuperación ante caída del servicio

* **Recuperación del enrutamiento en Proxmox (`pvexx`):**
  1. Verificar que `pve-firewall` permanece detenido (`systemctl status pve-firewall`) y que `net.ipv4.ip_forward=1`.
  2. Recargar interfaces y reglas `iptables` persistentes con `ifreload -a`.
  3. Si la configuración se ha perdido, restaurarla desde la copia de `/root/backup-config/`.
* **Restauración de un contenedor LXC:**
  ```bash
  pct rollback <ID> <NOMBRE_SNAPSHOT>
  # O restauración completa desde copia vzdump:
  pct restore <ID> /var/lib/vz/dump/vzdump-lxc-<ID>-*.tar.zst --force
  ```
* **Restauración de la base de datos:**
  ```bash
  docker exec -i --env-file /root/.db.env mysql_bcn_db \
      mysql -u user_bcn bcn_educacio < /root/db/backup_bcn_<FECHA>.sql
  ```

---

## 2. Criterios de Rendimiento y Disponibilidad

Los objetivos siguientes se miden mediante las pruebas de la sección 3. Cada criterio indica cómo se mide y en qué prueba se verifica.

| ID | Criterio | Objetivo | Cómo se mide | Prueba |
|---|---|---|---|---|
| **CRI-01** | Tiempo de respuesta del portal web | Respuesta completa del buscador CRUD en menos de 2 s | `curl -o /dev/null -s -w '%{time_total}\n' http://educacion.local` | PER-01 |
| **CRI-02** | Tiempo de consulta a la base de datos | Consulta de búsqueda en menos de 500 ms | `time mysql ... -e "SELECT COUNT(*) FROM <tabla>"` | PER-02 |
| **CRI-03** | Tiempo de recuperación de un contenedor (RTO) | Servicio restablecido en menos de 15 min desde `vzdump` | `time pct restore <ID> ...` | CON-05 |
| **CRI-04** | Pérdida máxima de datos (RPO) | Como máximo 24 h (exportación diaria de la BBDD) | Fecha del último `backup_bcn_*.sql` frente a la hora del fallo | CON-02 |
| **CRI-05** | Disponibilidad tras reinicio | Todos los servicios operativos sin intervención manual tras un `reboot` | Auditoría de `pct list`, `docker ps` y reglas `iptables` | AUT-04 |
| **CRI-06** | Integridad de los datos | 2.710 registros tras cualquier incidencia | `SELECT COUNT(*)` antes y después | CON-02 |

> Los valores objetivo son los que adopta el equipo para esta práctica. Si en la ejecución real se obtienen otros tiempos, se registran en la columna de resultado de la prueba y se justifica la diferencia.

---

## 3. Juego de Pruebas Integral de Validación (RA2)

**Convenciones:** cada prueba indica el **resultado esperado** y el **resultado obtenido** por separado, junto con la fecha de ejecución y la evidencia (ver sección 4). Los estados posibles son **Superada**, **Fallida** y **Pendiente de ejecutar**.

### 3.1. Pruebas de Automatización

| ID | Prueba | Procedimiento / Comando | Resultado esperado | Resultado obtenido | Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **AUT-01** | Entrega automática de IP y ruta estática (Opción DHCP 121) | Renovar IP en `CLI-WIN` (`ipconfig /renew`) y `CLI-LINUX` (`netplan apply`). | IP en `192.168.2.50-150`, DNS `192.168.2.1` y ruta `10.10.10.0/24 via 192.168.2.1`. | _Completar con la salida real_ | `evidencias/AUT-01.png` | **Superada** |
| **AUT-02** | Resolución DNS autoritativa sin `/etc/hosts` | `nslookup educacion.local` y `nslookup f-ncc` desde los clientes. | `educacion.local` → `10.10.10.20` y `f-ncc` → `10.10.10.10`. | _Completar_ | `evidencias/AUT-02.png` | **Superada** |
| **AUT-03** | Limpieza UTF-16 LE BOM e ingesta SQL | `python3 generar_sql.py` en `B-NCC` (`10.10.20.10`). | Eliminación de `\ufeff` e inserción de 2.710 registros en `utf8mb4` sin *mojibake*. | _Completar_ | `evidencias/AUT-03.png` | **Superada** |
| **AUT-04** | Persistencia de servicios tras reinicio completo | `reboot` de `pvexx` y auditoría de `pct list`, `iptables -t nat -L` y `docker ps`. | LXC con `onboot=1`, reglas `post-up` activas y contenedores Docker (`restart: always`) operativos. | _Completar_ | `evidencias/AUT-04.png` | **Superada** |
| **AUT-05** | Despliegue automatizado con `deploy.sh` | `sudo ./deploy.sh` en un entorno limpio. | Instalación de Docker/Python, creación de manifiestos y arranque de los 3 contenedores. | _Completar_ | `evidencias/AUT-05.png` | **Superada** |
| **AUT-06** | Resolución del router (`R-NCC` y `R`) | `nslookup r-ncc` y `nslookup r` desde un cliente. | Ambos nombres resuelven a las direcciones del router (`10.10.10.1`, `10.10.20.1`, `192.168.2.1`). | _Completar_ | `evidencias/AUT-06.png` | **Pendiente de ejecutar** |

### 3.2. Pruebas de Contingencia y Tolerancia a Fallos

| ID | Prueba | Procedimiento / Comando | Resultado esperado | Resultado obtenido | Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **CON-01** | Migración del enrutamiento tras el bloqueo *Anti-MAC Spoofing* | Detener `CT 103` (`R-NCC`) y activar `dnsmasq` + `iptables` en `pvexx`. | DHCP emitido con la MAC autorizada de `enp3s0`; los clientes reciben IP y salida NAT. | _Completar_ | `evidencias/CON-01.png` | **Superada** |
| **CON-02** | Recuperación tras parada del contenedor MySQL | `docker stop mysql_bcn_db` y `docker start mysql_bcn_db`; contar registros antes y después. | Se conservan los 2.710 registros gracias al volumen `db_data` (CRI-04 y CRI-06). | _Completar_ | `evidencias/CON-02.png` | **Superada** |
| **CON-03** | Restauración de la pasarela Intranet (`10.10.20.1`) | Declarar `vmbr1:0` (`10.10.20.1/24`) y ejecutar `ifreload -a`. | Comunicación restablecida entre `W-NCC` (`10.10.10.20`) y `B-NCC` (`10.10.20.10`). | _Completar_ | `evidencias/CON-03.png` | **Superada** |
| **CON-04** | Recuperación ante fragmentación MTU en la VPN | Inyectar `TCPMSS --clamp-mss-to-pmtu` en `FORWARD` de `pvexx`. | Carga fluida del portal y sesiones TLS a través del túnel VXLAN (`MTU 1446`). | _Completar_ | `evidencias/CON-04.png` | **Superada** |
| **CON-05** | Restauración completa de un contenedor desde `vzdump` (RTO) | `pct stop <ID>`, `time pct restore <ID> <copia> --force`, `pct start <ID>`. | Servicio restablecido en menos de 15 min (CRI-03). | _Registrar el tiempo medido_ | `evidencias/CON-05.png` | **Pendiente de ejecutar** |
| **CON-06** | Restauración de la BBDD desde `mysqldump` | Borrar una tabla de prueba y restaurar con el comando de la sección 1.3. | Tabla y registros recuperados; recuento final de 2.710. | _Completar_ | `evidencias/CON-06.png` | **Pendiente de ejecutar** |

### 3.3. Pruebas de Monitorización y Auditoría

| ID | Prueba | Procedimiento / Comando | Resultado esperado | Resultado obtenido | Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **MON-01** | Auditoría en tiempo real de consultas DNS y DHCP | `tail -f /var/log/dnsmasq.log` mientras un cliente consulta y renueva. | Entradas `query[A] educacion.local` y concesiones `DHCPACK`. | _Completar_ | `evidencias/MON-01.png` | **Superada** |
| **MON-02** | Análisis de tráfico con `tcpdump` | `tcpdump -ni any host 10.10.10.20` durante un `ping` y una petición web. | Enrutamiento simétrico sin pérdida de paquetes entre VPN, DMZ e Intranet. | _Completar_ | `evidencias/MON-02.png` | **Superada** |
| **MON-03** | Auditoría de atributos de seguridad LDAP | `ldapsearch` solicitando `pwdAccountLockedTime` y `pwdFailureTime`. | Marcas de tiempo de intentos fallidos y estado de bloqueo por cuenta. | _Completar_ | `evidencias/MON-03.png` | **Superada** |

### 3.4. Pruebas de Rendimiento

| ID | Prueba | Procedimiento / Comando | Resultado esperado | Resultado obtenido | Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **PER-01** | Tiempo de respuesta del portal | `curl -o /dev/null -s -w '%{time_total}\n' http://educacion.local` (repetir 10 veces). | Media inferior a 2 s (CRI-01). | _Registrar media_ | `evidencias/PER-01.png` | **Pendiente de ejecutar** |
| **PER-02** | Tiempo de consulta en la BBDD | Ejecutar una consulta de búsqueda con `time` en `B-NCC`. | Inferior a 500 ms (CRI-02). | _Registrar tiempo_ | `evidencias/PER-02.png` | **Pendiente de ejecutar** |

### 3.5. Pruebas de Seguridad

| ID | Prueba | Procedimiento / Comando | Resultado esperado | Resultado obtenido | Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **SEG-01** | Bloqueo por fuerza bruta (`ppolicy`) | Fallar 3 veces el inicio de sesión con `bchecker` en la web. | Alerta progresiva (2 intentos, último intento) y bloqueo automático de 15 minutos al tercer fallo. | _Completar_ | `evidencias/SEG-01.png` | **Superada** |
| **SEG-02** | Desbloqueo administrativo en 1 clic | Autenticarse con `admin` (grupo `admins`, GID `10000`) y pulsar **Desbloquear**. | Se elimina `pwdAccountLockedTime` y `bchecker` recupera el acceso. | _Completar_ | `evidencias/SEG-02.png` | **Superada** |
| **SEG-03** | Protección frente a SQL Injection y XSS | Introducir `' OR '1'='1` y `<script>alert(1)</script>` en el buscador CRUD. | Sentencias preparadas (`mysqli::prepare`) y `htmlspecialchars` neutralizan la carga. | _Completar_ | `evidencias/SEG-03.png` | **Superada** |
| **SEG-04** | Aislamiento de MySQL desde clientes y DMZ | Desde `CLI-LINUX`: `nc -zv -w 3 10.10.20.10 3306`. Desde `CLI-WIN`: `Test-NetConnection 10.10.20.10 -Port 3306`. Desde `F-NCC`: `nc -zv -w 3 10.10.20.10 3306`. | Tiempo de espera agotado en los tres casos; el contador de la regla `DROP` aumenta en `iptables -L FORWARD -n -v`. | _Completar_ | `evidencias/SEG-04.png` | **Pendiente de ejecutar** |
| **SEG-05** | Acceso legítimo de la aplicación a MySQL | Desde `W-NCC`: `nc -zv 10.10.20.10 3306` y abrir `http://educacion.local`. | Conexión establecida y el portal muestra los 2.710 registros. | _Completar_ | `evidencias/SEG-05.png` | **Pendiente de ejecutar** |
| **SEG-06** | Persistencia de las reglas de aislamiento | `ifreload -a` dos veces y `iptables -S FORWARD`. | Las reglas del puerto `3306` aparecen una sola vez, sin duplicados. | _Completar_ | `evidencias/SEG-06.png` | **Pendiente de ejecutar** |

> **Reglas de aislamiento (referencia):** las pruebas SEG-04 a SEG-06 verifican las reglas `iptables` del bloque `vmbr1` de `/etc/network/interfaces` descrito en `01-arquitectura-red.md`. Hasta que esas reglas estén aplicadas, estas pruebas no pueden darse por superadas.

---

## 4. Registro de Evidencias

Para cada prueba se guarda una captura o la salida del comando en la carpeta `evidencias/` del repositorio, con el nombre del identificador de la prueba (por ejemplo, `evidencias/AUT-01.png`).

Cada evidencia debe mostrar, en una misma imagen o fichero:

1. El comando ejecutado.
2. La salida obtenida.
3. El nombre del equipo (prompt) y la fecha y hora.

Una vez ejecutada la prueba, se rellenan las columnas **Resultado obtenido** y **Estado**, y se anota quién la ejecutó y la fecha en el mensaje del `git commit` que sube la evidencia.
