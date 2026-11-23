# Configuración del Servidor Web y Aplicación CRUD (`WEB-NCC`)

Este documento detalla la arquitectura, la instalación, la configuración de **Nginx** y **PHP 8.2-FPM**, la implementación de las operaciones CRUD, las medidas de seguridad frente a inyecciones SQL/XSS, el script de aprovisionamiento (`deploy.sh`) y los procedimientos de mantenimiento del servidor **`WEB-NCC`** (`CT 105` · `10.10.10.20`).

**Índice**

1. [Arquitectura técnica de la capa web](#1-arquitectura-técnica-de-la-capa-web)
2. [Instalación y dependencias](#2-instalación-y-dependencias)
3. [Configuración del virtual host Nginx](#3-configuración-del-virtual-host-nginx)
4. [Aplicación CRUD, autenticación y seguridad](#4-aplicación-crud-autenticación-y-seguridad)
5. [Script de despliegue automático (`deploy.sh`)](#5-script-de-despliegue-automático-deploysh)
6. [Mantenimiento y resolución de problemas](#6-mantenimiento-y-resolución-de-problemas)
7. [Limitaciones conocidas y mejoras previstas](#7-limitaciones-conocidas-y-mejoras-previstas)

---

## 1. Arquitectura Técnica de la Capa Web

| Componente | Tecnología / Versión | Configuración y rol |
| :--- | :--- | :--- |
| **Contenedor LXC** | `CT 105` (`WEB-NCC`) | Zona DMZ (`vmbr1`), IP estática `10.10.10.20/24`. |
| **Servidor web** | Nginx (paquete de la distribución) | Escucha en el puerto `80`, sirve los ficheros estáticos y delega las peticiones `.php` en PHP-FPM mediante el socket Unix `/run/php/php8.2-fpm.sock`. |
| **Motor backend** | PHP 8.2-FPM | Extensiones `php-mysql` (incluye `mysqli` y `pdo_mysql`) y `php-ldap`. |
| **Interfaz (UI)** | Bootstrap 5.3 + Bootstrap Icons 1.10 | Diseño responsivo con tablas adaptativas y ventanas modales. |
| **Dominio local** | `http://educacion.local` | Resuelto por `dnsmasq` en `pvexx` hacia `10.10.10.20`. |
| **Base de datos** | MySQL 8.0 en `B-NCC` | Conexión a `10.10.20.10:3306` (base `bcn_educacio`, tabla `equipaments`). |
| **Autenticación** | OpenLDAP en `LDAP-NCC` | Inicio de sesión contra `10.10.10.30:389` (ver sección 4.3). |
| **Cuenta de administración** | `bchecker` | Usuario local con shell `/bin/bash` y grupo `sudo`, según el estándar del proyecto. |

> <!-- CONFIRMAR: el stack Docker de B-NCC (php_bcn_app, nginx_bcn_web) es la versión inicial/MVP de la aplicación y el portal productivo es el de WEB-NCC. Si es así, dejarlo escrito aquí; si ambos siguen en uso, explicar el papel de cada uno. -->

---

## 2. Instalación y dependencias

Paquetes necesarios en `WEB-NCC`:

```bash
apt-get update
apt-get install -y nginx php8.2-fpm php8.2-mysql php8.2-ldap
```

Comprobación de las versiones instaladas:

```bash
nginx -v
php -v
php -m | grep -Ei 'mysqli|pdo_mysql|ldap'
ls /run/php/php8.2-fpm.sock
```

| Dependencia | Para qué se usa |
| :--- | :--- |
| `nginx` | Servidor web y proxy hacia PHP-FPM. |
| `php8.2-fpm` | Ejecución de la aplicación PHP. |
| `php8.2-mysql` | Acceso a MySQL con `mysqli` y `pdo_mysql`. |
| `php8.2-ldap` | Autenticación contra OpenLDAP. |

Los ficheros de la aplicación se despliegan en `/var/www/html`.

---

## 3. Configuración del Virtual Host Nginx

Fichero `/etc/nginx/conf.d/default.conf` (o `sites-available/educacion.local`):

```nginx
server {
    listen 80;
    server_name educacion.local www.educacion.local localhost;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Validación y aplicación de los cambios:

```bash
nginx -t
systemctl reload nginx
```

---

## 4. Aplicación CRUD, autenticación y seguridad

### 4.1. Operaciones implementadas en `/var/www/html/index.php`

1. **Consulta y filtrado (`Read`):**
   * Búsqueda textual libre con `LIKE` sobre las columnas `name`, `address_road` y `category`.
   * Filtro combinado por distrito municipal (`district`) con contador total en tiempo real (**2.710 equipamientos educativos**).
2. **Alta de centros (`Create`):**
   * Formulario modal (`+ Añadir Equipamiento`) que inserta registros en `bcn_educacio.equipaments` (`10.10.20.10`).
3. **Edición de registros (`Update`):**
   * Ventana modal precargada con los datos del registro, codificados como JSON seguro (`JSON_HEX_APOS | JSON_HEX_QUOT | JSON_UNESCAPED_UNICODE`).
4. **Eliminación (`Delete`):**
   * Confirmación previa en JavaScript y borrado por identificador entero (`intval($_GET['id'])`).

### 4.2. Prevención de inyección SQL (SQLi) y XSS

* **Consultas preparadas:** toda interacción con MySQL usa `mysqli::prepare()` y `bind_param()`, que separan la estructura SQL de los datos del usuario.
* **Saneamiento de salidas HTML (anti-XSS):** las variables que se muestran pasan por `htmlspecialchars($valor, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8')`.
* **Codificación `utf8mb4`:** garantiza la representación correcta de tildes y caracteres catalanes (*Escola Auró*, *Gràcia*, *Sant Martí*).

### 4.3. Autenticación con OpenLDAP

El acceso al portal exige iniciar sesión contra el directorio `dc=educacion,dc=local` de `LDAP-NCC`.

* **Política de bloqueo (`ppolicy`):** tras tres intentos fallidos se bloquea la cuenta durante 15 minutos.
* **Desbloqueo administrativo:** los usuarios del grupo `admins` (GID `10000`) disponen de un botón **Desbloquear** que elimina el atributo `pwdAccountLockedTime`.
* **Usuario de auditoría:** `uid=bchecker,ou=usuarios,dc=educacion,dc=local`.

La configuración del directorio está descrita en `03-servicios/03-configuracion-ldap.md`. Las pruebas correspondientes son SEG-01 y SEG-02 de `02-Redes-y-seguridad/Contingencia-y-pruebas.md`.

### 4.4. Conexión con la base de datos

La aplicación se conecta a `10.10.20.10:3306`. Este tráfico atraviesa `pvexx` (subredes distintas) y está permitido por la regla de aislamiento que solo admite como origen `10.10.10.20` (ver `01-arquitectura-red.md`). Las credenciales de acceso a MySQL no se incluyen en el repositorio.

---

## 5. Script de Despliegue Automático (`deploy.sh`)

El script instala las dependencias y levanta con Docker Compose el stack formado por MySQL, PHP-FPM y Nginx (contenedores `mysql_bcn_db`, `php_bcn_app` y `nginx_bcn_web`). Debe ejecutarse desde la carpeta que contiene `init.sql` y, opcionalmente, la carpeta `html/` con la aplicación.

**Cambios respecto a la versión anterior:**

* Las contraseñas de MySQL ya no están escritas en el script: se piden como variables de entorno y se guardan en un fichero `.env` con permisos `600` que **no debe subirse a Git**.
* El script comprueba que existe `init.sql` (antes, Docker creaba una carpeta con ese nombre si faltaba el fichero) y copia la aplicación.
* Crea el usuario estándar `bchecker` si no existe.
* Se ha eliminado la clave `version` de `docker-compose.yml`, que Compose v2 considera obsoleta.

```bash
#!/bin/bash
# ==============================================================================
# Script de despliegue automático para el proyecto BCN Educación
# Uso:
#   export DB_ROOT_PASSWORD='...' DB_PASSWORD='...'
#   sudo -E ./deploy.sh
# ==============================================================================

set -euo pipefail

: "${DB_ROOT_PASSWORD:?Define la variable DB_ROOT_PASSWORD}"
: "${DB_PASSWORD:?Define la variable DB_PASSWORD}"

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_DIR="bcn_educacio_app"

echo "1. Instalando dependencias del sistema..."
apt-get update
apt-get install -y docker.io docker-compose-v2 python3 python3-pip python3-pandas curl

echo "2. Creando el usuario estándar bchecker (si no existe)..."
if ! id bchecker >/dev/null 2>&1; then
    useradd -m -s /bin/bash bchecker
    echo "bchecker:bchecker121" | chpasswd
    usermod -aG sudo bchecker
fi

echo "3. Creando estructura de directorios..."
mkdir -p "$PROJECT_DIR/html"
cd "$PROJECT_DIR"

if [ ! -f "$SCRIPT_DIR/init.sql" ]; then
    echo "ERROR: falta $SCRIPT_DIR/init.sql" >&2
    exit 1
fi
cp "$SCRIPT_DIR/init.sql" ./init.sql
[ -d "$SCRIPT_DIR/html" ] && cp -r "$SCRIPT_DIR/html/." ./html/

echo "4. Guardando credenciales en .env (permisos 600)..."
install -m 600 /dev/null .env
cat > .env << EOF
DB_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
DB_PASSWORD=${DB_PASSWORD}
EOF

echo "5. Generando docker-compose.yml, Dockerfile y default.conf..."
cat << 'EOF' > docker-compose.yml
services:
  db:
    image: mysql:8.0
    container_name: mysql_bcn_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: bcn_educacio
      MYSQL_USER: user_bcn
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  php:
    build: .
    container_name: php_bcn_app
    restart: always
    volumes:
      - ./html:/var/www/html

  web:
    image: nginx:alpine
    container_name: nginx_bcn_web
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./html:/var/www/html
      - ./default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
      - db

volumes:
  db_data:
EOF

cat << 'EOF' > Dockerfile
FROM php:8.2-fpm
RUN docker-php-ext-install pdo pdo_mysql mysqli
EOF

cat << 'EOF' > default.conf
server {
    listen 80;
    server_name localhost;
    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

echo "6. Desplegando contenedores con Docker Compose..."
docker compose up -d --build
docker compose ps
echo "¡Despliegue finalizado con éxito!"
```

Añade al `.gitignore` del repositorio:

```
.env
bcn_educacio_app/.env
```

> **Nota:** esta versión del script está revisada pero **no se ha ejecutado**. La prueba **AUT-05** de `Contingencia-y-pruebas.md` debe repetirse con ella antes de darla por superada.

---

## 6. Mantenimiento y Resolución de Problemas

### 6.1. Operaciones habituales

| Tarea | Comando |
| :--- | :--- |
| Validar la configuración de Nginx | `nginx -t` |
| Aplicar cambios de Nginx sin cortar el servicio | `systemctl reload nginx` |
| Reiniciar PHP-FPM | `systemctl restart php8.2-fpm` |
| Estado de los servicios | `systemctl status nginx php8.2-fpm` |
| Registro de errores de Nginx | `tail -f /var/log/nginx/error.log` |
| Registro de accesos | `tail -f /var/log/nginx/access.log` |
| Copia de seguridad del contenedor | `vzdump 105 --mode snapshot --compress zstd --storage local` (desde `pvexx`) |

### 6.2. Resolución de problemas

| Síntoma | Causa probable | Comprobación y solución |
| :--- | :--- | :--- |
| `502 Bad Gateway` | PHP-FPM parado o ruta del socket incorrecta | `systemctl status php8.2-fpm` y `ls /run/php/`. La ruta de `fastcgi_pass` debe coincidir con el socket existente. |
| Se descarga el `.php` en lugar de ejecutarse | Falta el bloque `location ~ \.php$` | Revisar `default.conf` y ejecutar `nginx -t`. |
| `404 Not Found` en la raíz | Falta `index.php` o la directiva `try_files` está mal escrita | Comprobar `/var/www/html` y que `try_files` lleve espacios (`$uri $uri/ =404`). |
| Error de conexión a la base de datos | `B-NCC` caído, ruta o regla de cortafuegos | Desde `WEB-NCC`: `nc -zv 10.10.20.10 3306`. Revisar el contador de las reglas `FORWARD` en `pvexx`. |
| No se puede iniciar sesión | `LDAP-NCC` inaccesible o cuenta bloqueada | `nc -zv 10.10.10.30 389` y consultar `pwdAccountLockedTime` con `ldapsearch`. |
| `educacion.local` no resuelve | DNS del cliente | `nslookup educacion.local` debe devolver `10.10.10.20`; revisar `dnsmasq` en `pvexx`. |
| Caracteres extraños (*mojibake*) | Conexión sin `utf8mb4` | Comprobar `mysqli_set_charset($conn, 'utf8mb4')` en la aplicación. |

---

## 7. Limitaciones Conocidas y Mejoras Previstas

1. **HTTPS no implementado.** El DNAT de `pvexx` redirige también el puerto `443` hacia `WEB-NCC`, pero Nginx solo escucha en el `80`. Con un certificado autofirmado se habilitaría así:

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
       -keyout /etc/ssl/private/educacion.key \
       -out /etc/ssl/certs/educacion.crt \
       -subj "/CN=educacion.local"
   ```

   ```nginx
   server {
       listen 443 ssl;
       server_name educacion.local www.educacion.local;
       ssl_certificate     /etc/ssl/certs/educacion.crt;
       ssl_certificate_key /etc/ssl/private/educacion.key;
       # ... mismo root, index y location que el bloque del puerto 80
   }
   ```

   Estado: **pendiente de aplicar y probar**. Con un dominio público, la alternativa es un certificado de Let's Encrypt.

2. **Borrado mediante petición `GET`.** Eliminar con `?id=` hace que cualquier enlace o imagen incrustada pueda provocar un borrado si el usuario tiene sesión. Mejora: usar `POST` con un token anti-CSRF y comprobar el rol del usuario.
3. **Cabeceras de seguridad HTTP** (`X-Frame-Options`, `X-Content-Type-Options`, `Content-Security-Policy`) sin configurar en Nginx.
4. **Contraseñas en el repositorio.** `bchecker121` se mantiene porque lo exige el enunciado. Cualquier otra credencial debe vivir fuera de Git (ver `.env`).
5. **Propuestas del último MVP** (API REST con rutas CRUD en Flask o Phalcon, certificados y dominio público, honeypot en la DMZ, port knocking en SSH): no implementadas en esta entrega.
