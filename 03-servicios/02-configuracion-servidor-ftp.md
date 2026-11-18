# Configuración del Servidor de Transferencia de Ficheros (`F-NCC`)

Este documento describe el despliegue, la configuración de red pasiva y la validación del servidor de transferencia de archivos **`F-NCC`** (`CT 104` / `Servidor-FTP`), alojado en la Zona Desmilitarizada (**DMZ — `10.10.10.10/24`**).

---

## 1. Especificaciones del Servicio

* **Contenedor LXC:** `CT 104` (`Servidor-FTP` / `F-NCC`).
* **Ubicación de Red:** Zona DMZ (`vmbr1`), IP estática `10.10.10.10/24`, Gateway `10.10.10.1`.
* **Resolución DNS:** Resuelto autoritativamente como `f-ncc` y `f-ncc.educacion.local` (`10.10.10.10`) por `dnsmasq` en Proxmox.
* **Motor FTP:** Imagen Docker `delfer/proftpd` configurada en **modo pasivo** (puerto de control `21` y rango de datos `60000-60010`) para atravesar reglas NAT y cortafuegos sin bloqueos de conexión.
* **Credenciales de Auditoría:** Usuario `bchecker` / Contraseña `bchecker121` con persistencia en volumen Docker `ftp_data` (`/home/bchecker`).

---

## 2. Despliegue mediante Docker Compose (`docker-compose.yml`)

```yaml
version: '3.8'

services:
  ftp:
    image: delfer/proftpd
    container_name: ftp_bcn
    hostname: F-NCC
    restart: always
    environment:
      FTP_USER: bchecker
      FTP_PASSWORD: bchecker121
      PASV_ADDRESS: 10.10.10.10
      PASV_MIN_PORT: 60000
      PASV_MAX_PORT: 60010
    ports:
      - "21:21"
      - "60000-60010:60000-60010"
    volumes:
      - ftp_data:/home/bchecker

volumes:
  ftp_data:
```

---

## 3. Puesta en Marcha y Comandos de Verificación

### 3.1. Despliegue en el Servidor `F-NCC`
```bash
# Iniciar el contenedor ProFTPD en segundo plano
docker compose up -d

# Verificar que el contenedor está activo
docker ps

# Comprobar puertos escuchando (21 y rango pasivo)
ss -tulpn | grep -E '21|60000'
```

### 3.2. Validación desde Clientes (`CLI-LINUX` / `CLI-WIN`)
```bash
# Prueba de resolución DNS y conectividad ICMP
ping -c 3 f-ncc

# Conexión FTP interactiva y transferencia de prueba
ftp 10.10.10.10
# Usuario: bchecker
# Contraseña: bchecker121
# ftp> passive
# ftp> ls
# ftp> put datos_prueba.txt
# ftp> bye
```
