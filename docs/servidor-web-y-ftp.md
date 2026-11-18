# Despliegue del Servidor Web (`WEB-NCC`) y Servidor FTP (`F-NCC`)

Este documento recoge la configuración de los dos servicios alojados en la Zona Desmilitarizada (**DMZ - `10.10.10.0/24`**): el portal web de gestión de equipamientos educativos (**`WEB-NCC`**) y el servidor de transferencia de ficheros en contenedor (**`F-NCC`**).

---

## 1. Servidor Web y Aplicación CRUD (`WEB-NCC` — `10.10.10.20`)

### 1.1. Arquitectura de Software
* **Sistema Operativo:** Ubuntu Server (Contenedor LXC `CT 105`).
* **Servidor Web y Motor PHP:** Nginx + PHP 8.2-FPM con extensiones `php-ldap` y `php-mysql`.
* **Dominio local:** `[http://educacion.local](http://educacion.local)` (resuelto autoritativamente por `dnsmasq` en Proxmox).
* **Ruta de despliegue:** `/var/www/html/index.php`.

### 1.2. Funcionalidades de la Aplicación Web
1. **Control de Acceso Centralizado (OpenLDAP):**
   * Valida las credenciales contra `ldap://10.10.10.30:389` (`ou=usuarios,dc=educacion,dc=local`).
   * Implementa el patrón **PRG (Post/Redirect/Get)** y regeneración de ID de sesión (`session_regenerate_id(true)`).
   * Consulta en tiempo real los atributos `pwdAccountLockedTime` y `pwdFailureTime` para mostrar los intentos restantes o el estado de bloqueo.
   * Habilita un panel superior de desbloqueo en 1 clic para miembros del grupo `admins` (`gidNumber=10000`).
2. **Operaciones CRUD sobre `bcn_educacio.equipaments` (`10.10.20.10`):**
   * **Consulta y Filtrado (`SELECT`):** Buscador combinado por texto libre (`name`, `address_road`, `category`) y selector por distrito municipal.
   * **Alta (`INSERT`):** Formulario modal para registrar nuevos centros educativos.
   * **Edición (`UPDATE`):** Modificación de registros existentes mediante ventanas modales.
   * **Eliminación (`DELETE`):** Borrado seguro con confirmación previa en JavaScript.
3. **Seguridad y Codificación:**
   * Uso exclusivo de **Consultas Preparadas** (`mysqli::prepare` y `bind_param`) para prevenir inyecciones SQL (SQLi).
   * Saneamiento de salidas HTML con `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')` frente a XSS.
   * Normalización estricta a `utf8mb4` para preservar acentos y caracteres catalanes/castellanos.

---

## 2. Servidor de Transferencia de Ficheros (`F-NCC` — `10.10.10.10`)

### 2.1. Despliegue mediante Docker Compose
En el contenedor LXC `CT 104` (`Servidor-FTP` / `10.10.10.10`) se despliega un servidor **ProFTPD** en modo pasivo orquestado mediante Docker Compose:

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
      FTP_PASSWORD: <CONTRASEÑA_USUARIO>
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

### 2.2. Puesta en Marcha y Verificación
```bash
# Levantar el servicio FTP en segundo plano
docker compose up -d

# Comprobar estado del contenedor y puertos expuestos
docker ps
ss -tulpn | grep -E '21|60000'
```
Desde los clientes (`CLI-LINUX` o `CLI-WIN`), se verifica la transferencia de archivos conectando a `ftp 10.10.10.10` o `ftp f-ncc` con el usuario `bchecker`.
