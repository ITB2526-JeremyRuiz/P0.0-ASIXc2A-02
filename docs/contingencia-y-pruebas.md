# Procedimientos de Administración, Copias de Seguridad y Juego de Pruebas

Este documento establece los protocolos de mantenimiento, copias de seguridad, recuperación ante desastres y el juego de pruebas integral (automatización, contingencia, monitorización y seguridad) que valida la infraestructura.

---

## 1. Copias de Seguridad y Recuperación (Backups)

### 1.1. Estrategia de Respaldo
1. **Snapshots y Copias de Contenedores LXC en Proxmox (`vzdump`):**
   * Antes de cualquier modificación crítica de red o paquetes, se genera un snapshot instantáneo en Proxmox (`pct snapshot <ID> pre_cambio`).
   * Respaldo completo comprimido (`zstd`) en el almacenamiento local de Proxmox (`/var/lib/vz/dump/`):
     ```bash
     vzdump 104 105 106 --mode snapshot --compress zstd --storage local
     ```
2. **Persistencia de Base de Datos (`B-NCC`):**
   * Volumen Docker persistente `db_data` (`/var/lib/mysql`) junto con los scripts reproducibles `init.sql` e `inserts.sql` (2.710 registros limpios en UTF-8).
   * Volcado lógico bajo demanda:
     ```bash
     docker exec mysql_bcn_db mysqldump -u user_bcn -p bcn_educacio > /root/db/backup_bcn_$(date +%F).sql
     ```

### 1.2. Procedimiento de Recuperación ante Caída
1. **Recuperación de Red en Proxmox:** Verificar `sysctl net.ipv4.ip_forward` (`1`), estado de `pve-firewall` (deshabilitado para evitar bloqueos entre bridges) y recargar interfaces con `ifreload -a`.
2. **Restauración de Contenedor LXC:**
   ```bash
   pct rollback <ID> <NOMBRE_SNAPSHOT>
   # O desde archivo vzdump:
   pct restore <ID> /var/lib/vz/dump/vzdump-lxc-<ID>-*.tar.zst --force
   ```

---

## 2. Juego de Pruebas de Validación Integral (RA2)

### 2.1. Pruebas de Automatización (DHCP, DNS e Ingesta de Datos)
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **AUT-01** | Asignación automática de IP y ruta (Opción 121) | Conectar cliente `CLI-LINUX` / `CLI-WIN` a red privada y renovar IP (`ipconfig /renew`). | Recibe IP en `192.168.2.50-150`, DNS `192.168.2.1` y ruta `10.10.10.0/24 via 192.168.2.1`. | **Superada** |
| **AUT-02** | Resolución DNS autoritativa interna | Ejecutar `nslookup educacion.local` desde cliente local y por VPN. | Resuelve `educacion.local` a `10.10.10.20` sin editar `/etc/hosts`. | **Superada** |
| **AUT-03** | Limpieza automatizada UTF-16 BOM e ingesta SQL | Ejecutar `python3 generar_sql.py` en `B-NCC` (`10.10.20.10`). | Carga 2.710 registros en `bcn_educacio.equipaments` sin caracteres corruptos (*mojibake*). | **Superada** |
| **AUT-04** | Arranque automático de servicios tras reinicio | Reiniciar nodo Proxmox (`reboot`) y comprobar `pct list` y `docker ps`. | Contenedores LXC (`onboot=1`), reglas `post-up` y contenedores Docker (`restart: always`) activos. | **Superada** |

### 2.2. Pruebas de Contingencia y Tolerancia a Fallos
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **CON-01** | Migración de enrutamiento tras bloqueo `R-NCC` | Detener `CT 103` (`R-NCC`) y activar `dnsmasq` + `iptables` en `pvexx`. | Superación del filtro *Anti-MAC Spoofing* de IsardVDI; entrega DHCP y salida NAT operativas. | **Superada** |
| **CON-02** | Caída y recuperación del servicio de Base de Datos | Ejecutar `docker stop mysql_bcn_db` y posteriormente `docker start mysql_bcn_db`. | Persistencia íntegra de los 2.710 registros gracias al volumen `db_data`. | **Superada** |
| **CON-03** | Restauración de conectividad Intranet tras reinicio | Añadir alias `vmbr1:0` (`10.10.20.1/24`) en `/etc/network/interfaces` y ejecutar `ifreload -a`. | Comunicación fluida entre `WEB-NCC` (`10.10.10.20`) y `B-NCC` (`10.10.20.10`) durante el login LDAP. | **Superada** |

### 2.3. Pruebas de Monitorización y Auditoría
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **MON-01** | Auditoría de consultas DNS y concesiones DHCP | Inspeccionar `/var/log/dnsmasq.log` en Proxmox (`tail -f /var/log/dnsmasq.log`). | Registro en tiempo real de consultas a `educacion.local` y entregas `DHCPACK`. | **Superada** |
| **MON-02** | Captura de tráfico y diagnóstico de red | Ejecutar `tcpdump -ni any icmp and host 10.10.10.20` y `tcpdump -ni enp3s0 port 67`. | Verificación de flujo bidireccional sin pérdida de paquetes ni fragmentación MTU. | **Superada** |
| **MON-03** | Inspección de atributos operativos LDAP | Consultar `ldapsearch` sobre `pwdAccountLockedTime` y `pwdFailureTime`. | Visibilidad exacta de marcas temporales de fallos de autenticación por usuario. | **Superada** |

### 2.4. Pruebas de Seguridad
| ID | Prueba Ejecutada | Procedimiento / Comando | Resultado Esperado | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **SEG-01** | Protección contra fuerza bruta (`ppolicy`) | Introducir contraseña errónea 3 veces seguidas con el usuario `bchecker` en la web. | Aviso de intentos restantes (2, último intento) y bloqueo automático de 15 min al tercer fallo. | **Superada** |
| **SEG-02** | Desbloqueo administrativo controlado | Iniciar sesión con `admin` (grupo `admins`, GID `10000`) y pulsar **Desbloquear**. | Eliminación de `pwdAccountLockedTime` y restablecimiento inmediato del acceso para `bchecker`. | **Superada** |
| **SEG-03** | Prevención de Inyección SQL (SQLi) y XSS | Introducir cargas `' OR '1'='1` y `<script>alert(1)</script>` en el buscador CRUD. | Consultas preparadas (`mysqli::prepare`) y `htmlspecialchars` neutralizan la inyección sin errores. | **Superada** |
| **SEG-04** | Aislamiento de subred Intranet | Intentar conexión directa desde cliente `192.168.2.x` al puerto `3306` de `10.10.20.10`. | Acceso directo restringido; únicamente `WEB-NCC` consulta la base de datos. | **Superada** |
