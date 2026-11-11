# Documentación Técnica: Servicio de Base de Datos y Aplicación (`B-NCC`)

## 1. Descripción General del Nodo

El servidor **`B-NCC`** (IP `10.10.20.10`) se encuentra desplegado de forma aislada dentro de la **Intranet**, alojando una pila de servicios orquestada mediante Docker Compose que incluye:
* **MySQL 8.0:** Motor de base de datos relacional con soporte completo para codificación UTF-8.
* **PHP 8.2-FPM:** Entorno de procesamiento dinámico con extensiones personalizadas para comunicación MySQL.
* **Nginx Alpine:** Servidor web ligero configurado como proxy inverso / servidor web interno.

---

## 2. Estructura de Directorios y Transferencia del Dataset

### 2.1. Estructura del Proyecto (`~/db`)
Todos los ficheros de configuración, scripts de datos y código fuente de la aplicación se ubican en la ruta `/root/db` del nodo `B-NCC`:

```text
~/db/
├── Dockerfile          # Definición de la imagen personalizada de PHP-FPM
├── datos.csv           # Dataset original de equipamientos educativos de Barcelona (2.4 MB)
├── default.conf        # Configuración del virtual host de Nginx para PHP-FPM
├── docker-compose.yml  # Orquestación de los tres servicios (db, php, web)
├── generar_sql.py      # Script Python para procesar el CSV, corregir codificación e importar
├── init.sql            # Script SQL de inicialización del esquema de la base de datos
├── inserts.sql         # Fichero SQL generado con las sentencias de inserción masiva (885 KB)
```

Comprobación de archivos y permisos en la máquina `B-NCC`:

```bash
root@B-NCC:~/db# ls -lh
total 3.3M
-rw-r--r-- 1 root root   65 Oct 18 20:23 Dockerfile
-rw-r--r-- 1 root root 2.4M Oct 13 21:58 datos.csv
drwxr-xr-x 3 root root 4.0K Oct 13 21:38 db
-rw-r--r-- 1 root root  388 Nov 10 14:53 default.conf
-rw-r--r-- 1 root root  711 Oct 13 22:59 docker-compose.yml
-rw-r--r-- 1 root root 2.1K Oct 19 08:21 generar_sql.py
drwxr-xr-x 2 root root 4.0K Nov 10 15:24 html
-rw-r--r-- 1 root root  551 Oct 13 21:40 init.sql
-rw-r--r-- 1 root root 885K Oct 13 21:59 inserts.sql
```

### 2.2. Transferencia del Dataset y Generación de `inserts.sql`
1. **Copia del Dataset desde Proxmox VE:** El fichero `datos.csv` (2.4 MB), correspondiente al catálogo oficial de equipamientos educativos de Barcelona (BCN Educación), fue transferido directamente desde el nodo hipervisor **Proxmox VE (`pvexx`)** hasta el directorio `~/db` del contenedor `B-NCC` mediante SCP.
2. **Normalización e Importación:** Una vez en el servidor, se ejecutó el script `generar_sql.py`. Este script lee el CSV original (codificado en UTF-16 con marcas BOM), elimina caracteres corruptos, genera el archivo `inserts.sql` (885 KB) con la estructura limpia e inserta los registros en la base de datos `bcn_educacio`.

---

## 3. Orquestación con Docker Compose (`docker-compose.yml`)

Fichero de definición para el despliegue multi-contenedor y la persistencia del volumen de datos:

```yaml
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

---

## 4. Personalización del Servicio PHP (`Dockerfile`)

Imagen basada en `php:8.2-fpm` compilada con los conectores requeridos (`mysqli`, `pdo`, `pdo_mysql`) para permitir la interacción dinámica desde PHP hacia MySQL:

```dockerfile
FROM php:8.2-fpm
RUN docker-php-ext-install pdo pdo_mysql mysqli
```

---

## 5. Esquema de Base de Datos (`init.sql`)

Define la creación del esquema `bcn_educacio` y la tabla `equipaments` asegurando compatibilidad nativa con caracteres UTF-8 (`utf8mb4`):

```sql
CREATE DATABASE IF NOT EXISTS bcn_educacio CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE bcn_educacio;

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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 6. Script de Limpieza, Generación de SQL e Importación (`generar_sql.py`)

Script en Python que procesa el archivo `datos.csv` importado desde Proxmox VE, corrige problemas de codificación UTF-16/BOM e inserta los datos limpios en MySQL:

```python
import pandas as pd
import mysql.connector

# 1. Cargar el archivo CSV respetando la codificación UTF-16
csv_path = 'datos.csv'
df = pd.read_csv(csv_path, encoding='utf-16')

# Limpiar caracteres BOM en register_id
df['register_id'] = df['register_id'].astype(str).str.replace('\ufeff', '').str.strip()

# 2. Conectar a MySQL FORZANDO UTF-8 MB4
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

# 3. Vaciar la tabla para eliminar los registros corruptos
print("Vaciando tabla 'equipaments'...")
cursor.execute("TRUNCATE TABLE equipaments;")

# 4. Insertar los registros correctamente
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
print(f"¡Proceso completado! Se han vuelto a importar {count} registros con codificación UTF-8 limpia.")

cursor.close()
conn.close()
```

---

        }
    </script>
</body>
</html>
```
