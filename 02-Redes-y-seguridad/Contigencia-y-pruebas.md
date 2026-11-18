# Procedimientos de Administración, Copias de Seguridad y Juego de Pruebas

Este documento recoge las políticas de copias de seguridad, los protocolos de recuperación ante desastres y el juego de pruebas completo (automatización, contingencia, monitorización y seguridad) que valida la solución.

---

## 1. Copias de Seguridad y Recuperación ante Desastres

### 1.1. Mecanismos de Respaldo Implementados
1. **Snapshots y Copias de Seguridad de Contenedores LXC (`vzdump`):**
   * Creación de instantáneas previas a cambios críticos de red o configuración:
     ```bash
     pct snapshot 105 pre_actualizacion_ldap
     ```
   * Copia de seguridad completa comprimida (`zstd`) en el almacenamiento local de Proxmox (`/var/lib/vz/dump/`):
     ```bash
     vzdump 104 105 106 --mode snapshot --compress zstd --storage local
     ```
2. **Persistencia y Respaldo Lógico de Base de Datos (`B-NCC`):**
   * Persistencia en disco mediante volumen Docker `db_data` (`/var/lib/mysql`) junto con los scripts reproducibles `init.sql` e `inserts.sql` (2.710 registros limpios).
   * Exportación lógica bajo demanda:
     ```bash
     docker exec mysql_bcn_db mysqldump -u user_bcn -ppirineus bcn_educacio > /root/db/backup_bcn_$(date +%F).sql
     ```

### 1.2. Protocolo de Recuperación ante Caída del Servicio
* **Recuperación de enrutamiento en Proxmox (`pvexx`):**
  1. Verificar que `pve-firewall` permanece detenido (`systemctl status pve-firewall`) y `net.ipv4.ip_forward=1`.
  2. Recargar interfaces y reglas `iptables` persistentes con `ifreload -a`.
* **Restauración de un contenedor LXC:**
  ```bash
  pct rollback <ID> <NOMBRE_SNAPSHOT>
  # O restauración completa desde copia vzdump:
  pct restore <ID> /var/lib/vz/dump/vzdump-lxc-<ID>-*.tar.zst --force
  ```

---

## 2. Juego de Pruebas Integral de Validación (RA2)

### 2.1. Pruebas de Automatización
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado y Obtenido | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **AUT-01** | Entrega automática de IP y ruta estática (Opción DHCP 121) | Renovar IP en `CLI-WIN` (`ipconfig /renew`) y `CLI-LINUX` (`netplan apply`). | Asignación automática de IP (`192.168.2.50-150`), DNS `192.168.2.1` y ruta `10.10.10.0/24 via 192.168.2.1`. | **Superada** |
| **AUT-02** | Resolución DNS autoritativa sin `/etc/hosts` | Ejecutar `nslookup educacion.local` y `nslookup f-ncc` desde los clientes. | Proxmox (`192.168.2.1`) resuelve `educacion.local` -> `10.10.10.20` y `f-ncc` -> `10.10.10.10`. | **Superada** |
| **AUT-03** | Limpieza automática UTF-16 LE BOM e ingesta SQL | Ejecutar `python3 generar_sql.py` en `B-NCC` (`10.10.20.10`). | Eliminación de `\ufeff` e inserción limpia de 2.710 registros en `utf8mb4` sin *mojibake*. | **Superada** |
| **AUT-04** | Persistencia de servicios tras reinicio completo | Reiniciar nodo Proxmox (`reboot`) y auditar `pct list`, `iptables -t nat -L` y `docker ps`. | Contenedores LXC (`onboot=1`), reglas `post-up` y contenedores Docker (`restart: always`) operativos. | **Superada** |
| **AUT-05** | Despliegue automatizado mediante script `deploy.sh` | Ejecutar `sudo ./deploy.sh` en entorno limpio de pruebas. | Instalación de Docker/Python, creación de manifiestos y levantamiento automático de los 3 contenedores. | **Superada** |

### 2.2. Pruebas de Contingencia y Tolerancia a Fallos
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado y Obtenido | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **CON-01** | Migración de enrutamiento tras bloqueo *Anti-MAC Spoofing* | Detener `CT 103` (`R-NCC`) y activar `dnsmasq` + `iptables` en `pvexx`. | Tráfico DHCP emitido con la MAC autorizada de `enp3s0`; clientes reciben IP y salida NAT. | **Superada** |
| **CON-02** | Recuperación tras parada del contenedor MySQL | Ejecutar `docker stop mysql_bcn_db` y `docker start mysql_bcn_db`. | Conservación íntegra de los 2.710 registros gracias al volumen persistente `db_data`. | **Superada** |
| **CON-03** | Restauración de pasarela Intranet (`10.10.20.1`) | Declarar subinterfaz `vmbr1:0` (`10.10.20.1/24`) y ejecutar `ifreload -a`. | Comunicación restablecida entre `WEB-NCC` (`10.10.10.20`) y `B-NCC` (`10.10.20.10`). | **Superada** |
| **CON-04** | Recuperación ante fragmentación MTU en VPN | Inyectar regla `TCPMSS --clamp-mss-to-pmtu` en `FORWARD` de Proxmox. | Carga fluida de la interfaz web y sesiones TLS a través del túnel VXLAN (`MTU 1446`). | **Superada** |

### 2.3. Pruebas de Monitorización y Auditoría
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado y Obtenido | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **MON-01** | Auditoría en tiempo real de consultas DNS y DHCP | Monitorizar `/var/log/dnsmasq.log` (`tail -f /var/log/dnsmasq.log`). | Registro detallado de peticiones `query[A] educacion.local` y concesiones `DHCPACK`. | **Superada** |
| **MON-02** | Análisis de tráfico con `tcpdump` | Capturar tramas ICMP y DHCP (`tcpdump -ni any host 10.10.10.20`). | Verificación de enrutamiento simétrico sin pérdida de paquetes entre VPN, DMZ e Intranet. | **Superada** |
| **MON-03** | Auditoría de atributos operativos de seguridad LDAP | Ejecutar `ldapsearch` solicitando `pwdAccountLockedTime` y `pwdFailureTime`. | Lectura exacta de marcas de tiempo de intentos fallidos y estado de bloqueo por cuenta. | **Superada** |

### 2.4. Pruebas de Seguridad
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado y Obtenido | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **SEG-01** | Bloqueo por fuerza bruta (`ppolicy`) | Fallar 3 veces consecutivas el inicio de sesión con `bchecker` en la web. | Alerta progresiva (2 intentos, último intento) y bloqueo automático de 15 minutos al tercer fallo. | **Superada** |
| **SEG-02** | Desbloqueo administrativo en 1 clic | Autenticarse con `admin` (grupo `admins`, GID `10000`) y pulsar **Desbloquear**. | Eliminación de `pwdAccountLockedTime` y acceso inmediato restablecido para `bchecker`. | **Superada** |
| **SEG-03** | Protección frente a SQL Injection y XSS | Introducir `' OR '1'='1` y `<script>alert(1)</script>` en el buscador CRUD. | Sentencias preparadas (`mysqli::prepare`) y `htmlspecialchars` neutralizan la carga maliciosa. | **Superada** |
| **SEG-04** | Aislamiento de la base de datos en Intranet | Intentar conexión directa al puerto `3306` de `10.10.20.10` desde cliente externo. | Acceso directo bloqueado; solo `WEB-NCC` interactúa con MySQL. | **Superada** |
