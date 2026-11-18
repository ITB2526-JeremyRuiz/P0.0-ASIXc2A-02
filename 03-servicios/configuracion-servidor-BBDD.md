# Configuración del Servidor de Base de Datos y Limpieza UTF-8 (`B-NCC`)

## 1. Descripción General del Nodo

El servidor **`B-NCC`** (`CT 106`, IP `10.10.20.10`) se encuentra desplegado de forma aislada dentro de la **Intranet** (`10.10.20.0/24`), alojando una pila de servicios orquestada mediante Docker Compose en `/root/db`:
* **MySQL 8.0 (`mysql_bcn_db`):** Motor de base de datos relacional con codificación nativa `utf8mb4_unicode_ci` y persistencia en el volumen `db_data`.
* **PHP 8.2-FPM (`php_bcn_app`):** Imagen personalizada compilada con las extensiones `pdo`, `pdo_mysql` y `mysqli`.
* **Nginx Alpine (`nginx_bcn_web`):** Servidor web ligero interno para pruebas locales del stack.

---

## 2. Estructura de Directorios (`~/db`)

```text
~/db/
├── Dockerfile          # Imagen personalizada de PHP 8.2-FPM con conectores MySQL
├── datos.csv           # Dataset original de OpenData Barcelona (2.4 MB, UTF-16 LE)
├── default.conf        # Configuración del virtual host Nginx
├── docker-compose.yml  # Orquestación de los servicios db, php y web
├── generar_sql.py      # Script Python de normalización UTF-16 -> UTF-8 e importación
├── init.sql            # Script DDL/DCL de inicialización de esquema y usuarios
└── inserts.sql         # Fichero SQL generado con las 2.710 inserciones limpias (885 KB)
```

---

## 3. Orquestación Docker Compose y Personalización PHP

### 3.1. Archivo `docker-compose.yml`
```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    container_name: mysql_bcn_db
    hostname: B-NCC
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
```

### 3.2. Archivo `Dockerfile`
```dockerfile
FROM php:8.2-fpm
RUN docker-php-ext-install pdo pdo_mysql mysqli
```

---

## 4. Esquema de Base de Datos y Permisos (`init.sql`)

```sql
CREATE DATABASE IF NOT EXISTS bcn_educacio CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE bcn_educacio;

-- Usuario obligatorio de auditoría
CREATE USER IF NOT EXISTS 'bchecker'@'%' IDENTIFIED BY 'bchecker121';
GRANT ALL PRIVILEGES ON bcn_educacio.* TO 'bchecker'@'%';

-- Usuario de aplicación web
CREATE USER IF NOT EXISTS 'user_bcn'@'%' IDENTIFIED BY 'pirineus';
GRANT ALL PRIVILEGES ON bcn_educacio.* TO 'user_bcn'@'%';
FLUSH PRIVILEGES;

CREATE TABLE IF NOT EXISTS equipaments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    register_id VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    address_road VARCHAR(255),
    address_number VARCHAR(20),
    neighborhood VARCHAR(100),
    district VARCHAR(100),
    zip_code VARCHAR(20),
    category VARCHAR(150),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 5. Resolución de Codificación *Mojibake* e Importación (`generar_sql.py`)

El dataset descargado de OpenData Barcelona (**2.710 registros**) venía codificado en **UTF-16 LE con marca BOM (`\ufeff`)**, provocando corrupción de acentos y caracteres catalanes (*mojibake*) en las primeras pruebas. Adicionalmente, en **Python 3.12** la conexión estándar de `mysql.connector` fallaba debido a la eliminación de `ssl.wrap_socket`.

El script `generar_sql.py` resuelve ambos problemas leyendo explícitamente en `encoding='utf-16'`, eliminando el BOM y conectando a MySQL con `charset='utf8mb4'` y `ssl_disabled=True`:

```python
import pandas as pd
import mysql.connector

# 1. Cargar el archivo CSV respetando la codificación UTF-16
csv_path = 'datos.csv'
df = pd.read_csv(csv_path, encoding='utf-16')

# Limpiar caracteres invisibles BOM en register_id
df['register_id'] = df['register_id'].astype(str).str.replace('\ufeff', '').str.strip()

# 2. Conectar a MySQL forzando UTF-8 MB4 y desactivando SSL legado (Python 3.12)
conn = mysql.connector.connect(
    host="127.0.0.1",
    user="user_bcn",
    password="pirineus",
    database="bcn_educacio",
    port=3306,
    charset='utf8mb4',
    ssl_disabled=True
)
cursor = conn.cursor()

# 3. Vaciar tabla e insertar registros limpios
cursor.execute("TRUNCATE TABLE equipaments;")

insert_query = """
INSERT INTO equipaments 
(register_id, name, address_road, address_number, neighborhood, district, zip_code, category, latitude, longitude)
VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
"""

count = 0
for _, row in df.iterrows():
    lat = row['geo_epgs_4326_lat'] if pd.notnull(row['geo_epgs_4326_lat']) else None
    lon = row['geo_epgs_4326_lon'] if pd.notnull(row['geo_epgs_4326_lon']) else None
    values = (
        str(row['register_id']),
        str(row['name']) if pd.notnull(row['name']) else '',
        str(row['addresses_road_name']) if pd.notnull(row['addresses_road_name']) else None,
        str(row['addresses_start_street_number']) if pd.notnull(row['addresses_start_street_number']) else None,
        str(row['addresses_neighborhood_name']) if pd.notnull(row['addresses_neighborhood_name']) else None,
        str(row['addresses_district_name']) if pd.notnull(row['addresses_district_name']) else None,
        str(row['addresses_zip_code']) if pd.notnull(row['addresses_zip_code']) else None,
        str(row['secondary_filters_name']) if pd.notnull(row['secondary_filters_name']) else None,
        lat,
        lon
    )
    cursor.execute(insert_query, values)
    count += 1

conn.commit()
print(f"¡Proceso completado! Se han importado {count} registros con codificación UTF-8 limpia.")
cursor.close()
conn.close()
```

### Verificación de Integridad de Datos
```bash
docker exec -it mysql_bcn_db mysql -ubchecker -pbchecker121 -e "USE bcn_educacio; SELECT COUNT(*) AS Total FROM equipaments;"
# Resultado esperado: Total = 2710
```
