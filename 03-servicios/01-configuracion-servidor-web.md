# Configuración del Servidor Web y Aplicación CRUD (`WEB-NCC`)

Este documento detalla la arquitectura, la configuración de **Nginx** y **PHP 8.2-FPM**, la implementación de las operaciones CRUD, las medidas de seguridad frente a inyecciones SQL/XSS y el script de aprovisionamiento automático (`deploy.sh`) desplegados en **`WEB-NCC`** (`CT 105` · `10.10.10.20`).

---

## 1. Arquitectura Técnica de la Capa Web

| Componente | Tecnología / Versión | Configuración y Rol |
| :--- | :--- | :--- |
| **Contenedor LXC** | `CT 105` (`WEB-NCC`) | Alojado en la zona DMZ (`vmbr1`, IP estática `10.10.10.20/24`). |
| **Proxy Inverso / Web Server** | Nginx (Alpine / Ubuntu) | Escucha en puerto `80`, sirve activos estáticos y delega peticiones `.php` vía FastCGI (`9000`). |
| **Motor Backend PHP** | PHP 8.2-FPM | Compilado con extensiones `php-ldap`, `php-mysql` y `pdo_mysql`. |
| **Interfaz Frontend (UI)** | Bootstrap 5.3 + Bootstrap Icons 1.10 | Diseño responsivo con tablas adaptativas y ventanas modales interactivas. |
| **Dominio Local** | `[http://educacion.local](http://educacion.local)` | Resuelto autoritativamente por `dnsmasq` en Proxmox (`10.10.10.20`). |

---

## 2. Configuración del Virtual Host Nginx (`default.conf`)

```nginx
server {
    listen 80;
    server_name educacion.local www.educacion.local localhost;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri$uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---

## 3. Detalle de Operaciones CRUD y Seguridad

### 3.1. Operaciones Implementadas en `/var/www/html/index.php`
1. **Consulta y Filtrado (`Read`):**
   * Búsqueda textual libre mediante operador `LIKE` sobre las columnas `name`, `address_road` y `category`.
   * Filtro combinado por distrito municipal (`district`) con contador total en tiempo real (**2.710 equipamientos educativos**).
2. **Alta de Centros (`Create`):**
   * Formulario modal (`+ Añadir Equipamiento`) que inserta nuevos registros en `bcn_educacio.equipaments` (`10.10.20.10`).
3. **Edición de Registros (`Update`):**
   * Ventana modal precargada con los datos del registro seleccionado mediante JSON seguro (`JSON_HEX_APOS | JSON_HEX_QUOT | JSON_UNESCAPED_UNICODE`).
4. **Eliminación Segura (`Delete`):**
   * Confirmación previa en JavaScript antes de ejecutar el borrado por identificador entero (`intval($_GET['id'])`).

### 3.2. Prevención de Inyección SQL (SQLi) y XSS
* **Consultas Preparadas (`Prepared Statements`):** Toda interacción con MySQL utiliza `mysqli::prepare()` y `bind_param()`, separando la estructura SQL de los datos introducidos por el usuario.
* **Saneamiento de Salidas HTML (Anti-XSS):** Todas las variables renderizadas en la vista pasan por `htmlspecialchars($valor, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8')`.
* **Normalización UTF-8 (`utf8mb4`):** Garantiza la correcta representación de tildes y caracteres catalanes (*Escola Auró*, *Gràcia*, *Sant Martí*).

---

## 4. Script de Despliegue Automático (`deploy.sh`)

Para automatizar la instalación de dependencias y el despliegue completo en entornos limpios, se creó el script `deploy.sh`:

```bash
#!/bin/bash
# ==============================================================================
# Script de despliegue automático para el proyecto BCN Educación
# Ejecutar como root o con sudo: sudo ./deploy.sh
# ==============================================================================

set -e

echo "1. Instalando dependencias del sistema..."
apt-get update
apt-get install -y docker.io docker-compose-v2 python3 python3-pip python3-pandas curl

echo "2. Creando estructura de directorios..."
PROJECT_DIR="bcn_educacio_app"
mkdir -p "$PROJECT_DIR/html"
cd "$PROJECT_DIR"

echo "3. Generando docker-compose.yml, Dockerfile y default.conf..."
cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  db:
    image: mysql:8.0
    container_name: mysql_bcn_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: pirineus
      MYSQL_DATABASE: bcn_educacio
      MYSQL_USER: user_bcn
      MYSQL_PASSWORD: pirineus
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
        try_files $uri$uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

echo "4. Desplegando contenedores con Docker Compose..."
docker compose up -d --build
echo "¡Despliegue finalizado con éxito!"
```
