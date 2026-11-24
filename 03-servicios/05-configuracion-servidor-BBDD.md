# Configuración del Servidor de Base de Datos y Limpieza UTF-8 (`B-NCC`)

Este documento describe el despliegue del servidor de base de datos **`B-NCC`**, el esquema y los permisos de MySQL, la normalización e importación del dataset de equipamientos educativos de Barcelona, y los procedimientos de verificación y mantenimiento.

**Índice**

1. [Descripción general del nodo](#1-descripción-general-del-nodo)
2. [Estructura de directorios](#2-estructura-de-directorios-rootdb)
3. [Origen de los datos](#3-origen-de-los-datos)
4. [Orquestación con Docker Compose](#4-orquestación-con-docker-compose)
5. [Esquema de base de datos y permisos (`init.sql`)](#5-esquema-de-base-de-datos-y-permisos-initsql)
6. [Importación y resolución del *mojibake* (`generar_sql.py`)](#6-importación-y-resolución-del-mojibake-generar_sqlpy)
7. [Verificación de la integridad de los datos](#7-verificación-de-la-integridad-de-los-datos)
8. [Mantenimiento y resolución de problemas](#8-mantenimiento-y-resolución-de-problemas)
9. [Limitaciones conocidas y mejoras previstas](#9-limitaciones-conocidas-y-mejoras-previstas)

---

## 1. Descripción General del Nodo

El servidor **`B-NCC`** (`CT 106`, IP `10.10.20.10`) está desplegado en la **Intranet** (`10.10.20.0/24`). El acceso al puerto `3306` desde el resto de la infraestructura se restringe en `pvexx` mediante `iptables`, de modo que solo `WEB-NCC` (`10.10.10.20`) puede conectarse (ver `02-Redes-y-seguridad/01-arquitectura-red.md`).

El nodo aloja una pila de servicios orquestada con Docker Compose en `/root/db`:

| Contenedor | Imagen | Función |
| :--- | :--- | :--- |
| `mysql_bcn_db` | `mysql:8.0` | Motor de base de datos con codificación `utf8mb4_unicode_ci` y persistencia en el volumen `db_data`. |
| `php_bcn_app` | Imagen propia sobre `php:8.2-fpm` | PHP-FPM con las extensiones `pdo`, `pdo_mysql` y `mysqli`. |
| `nginx_bcn_web` | `nginx:alpine` | Servidor web interno **solo para pruebas locales de la pila**. |

> El portal de producción es el de `WEB-NCC` (ver `03-servicios/01-configuracion-servidor-web.md`). Los contenedores `php_bcn_app` y `nginx_bcn_web` de este nodo sirven para validar la base de datos de forma independiente.

El *hostname* del LXC debe ser `B-NCC`, como exige el enunciado:

```bash
# En pvexx
pct set 106 --hostname B-NCC
```

---

## 2. Estructura de Directorios (`/root/db`)

```text
/root/db/
├── .env                # Credenciales de MySQL (permisos 600, NO se sube a Git)
├── Dockerfile          # Imagen personalizada de PHP 8.2-FPM con conectores MySQL
├── datos.csv           # Dataset original de OpenData Barcelona (2,4 MB, UTF-16 LE)
├── default.conf        # Virtual host Nginx de pruebas
├── docker-compose.yml  # Orquestación de los servicios db, php y web
├── generar_sql.py      # Normalización UTF-16 -> UTF-8 e importación a MySQL
├── html/               # Aplicación de pruebas servida por php y web
├── init.sql            # Inicialización de base de datos, usuario de auditoría y esquema
└── inserts.sql         # Volcado SQL con las 2.710 inserciones limpias (885 KB)
```

<!-- CONFIRMAR: cómo se genera inserts.sql. generar_sql.py inserta directamente en MySQL y no escribe este fichero. Si se obtiene con mysqldump o con otro script, indicarlo aquí. -->

---

## 3. Origen de los Datos

Dataset **«Equipaments d'educació de la ciutat de Barcelona»** del portal de datos abiertos del Ayuntamiento de Barcelona:

* Ficha: `https://opendata-ajuntament.barcelona.cat/data/es/dataset/equipament-educacio`
* Descarga directa del CSV: `https://opendata-ajuntament.barcelona.cat/data/dataset/f36b60f2-9541-4d08-b0f9-b0a9313fab3d/resource/29d9ff10-6892-4f16-9012-d5c4997857e7/download`

El fichero contiene **2.710 registros** y se guarda como `datos.csv`.

---

## 4. Orquestación con Docker Compose

### 4.1. `.env` (no versionado)

```bash
install -m 600 /dev/null .env
cat > .env << 'EOF'
DB_ROOT_PASSWORD=<contraseña_root>
DB_PASSWORD=<contraseña_user_bcn>
EOF
```

Añadir al `.gitignore` del repositorio:

```
.env
```

### 4.2. `docker-compose.yml`

```yaml
services:
  db:
    image: mysql:8.0
    container_name: mysql_bcn_db
    hostname: B-NCC
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: bcn_educacio
      MYSQL_USER: user_bcn
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "10.10.20.10:3306:3306"
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
      - "127.0.0.1:80:80"
    volumes:
      - ./html:/var/www/html
      - ./default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
      - db

volumes:
  db_data:
```

**Exposición de puertos.** `B-NCC` tiene dos interfaces (`net0` por DHCP y `net1` en la Intranet). Publicar `3306:3306` sin dirección lo abriría en ambas. Con `10.10.20.10:3306:3306` MySQL solo escucha en la interfaz de la Intranet, y el Nginx de pruebas solo en el propio nodo (`127.0.0.1`).

### 4.3. `Dockerfile`

```dockerfile
FROM php:8.2-fpm
RUN docker-php-ext-install pdo pdo_mysql mysqli
```

---

## 5. Esquema de Base de Datos y Permisos (`init.sql`)

```sql
CREATE DATABASE IF NOT EXISTS bcn_educacio CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE bcn_educacio;

-- Usuario obligatorio de auditoría (exigido por el enunciado)
CREATE USER IF NOT EXISTS 'bchecker'@'%' IDENTIFIED BY 'bchecker121';
GRANT ALL PRIVILEGES ON bcn_educacio.* TO 'bchecker'@'%';

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

### 5.1. Usuarios de MySQL

| Usuario | Cómo se crea | Privilegios | Uso |
| :--- | :--- | :--- | :--- |
| `bchecker` | `init.sql` | `ALL PRIVILEGES` sobre `bcn_educacio.*` | Auditoría (obligatorio en todos los sistemas) |
| `user_bcn` | Variables `MYSQL_USER` y `MYSQL_PASSWORD` del Compose (contraseña en `.env`) | `ALL PRIVILEGES` sobre `bcn_educacio.*` (valor por defecto de la imagen) | Aplicación web |

### 5.2. Notas importantes

* Los scripts de `/docker-entrypoint-initdb.d/` **solo se ejecutan la primera vez**, cuando el volumen `db_data` está vacío. Modificar `init.sql` después no tiene efecto sin recrear el volumen (ver sección 8).
* `init.sql` debe existir **antes** del primer `docker compose up`. Si falta, Docker crea una carpeta con ese nombre y MySQL no se inicializa.

---

## 6. Importación y Resolución del *Mojibake* (`generar_sql.py`)

El dataset de OpenData Barcelona venía codificado en **UTF-16 LE con marca BOM (`\ufeff`)**, lo que provocaba la corrupción de acentos y caracteres catalanes (*mojibake*) en las primeras pruebas. Además, en **Python 3.12** el conector `mysql.connector` fallaba porque `ssl.wrap_socket` fue eliminado de la biblioteca estándar.

El script soluciona ambos problemas:

* Lee el CSV con `encoding='utf-16'` (el códec consume la marca BOM de cabecera) y elimina cualquier `\ufeff` residual del campo `register_id`.
* Conecta con `charset='utf8mb4'` y `ssl_disabled=True`.
* Recibe las credenciales por variables de entorno, sin escribirlas en el código.

### 6.1. Dependencias y ejecución

```bash
apt-get install -y python3-venv
python3 -m venv /root/db/venv
/root/db/venv/bin/pip install pandas mysql-connector-python

export DB_USER=bchecker DB_PASSWORD=bchecker121
/root/db/venv/bin/python /root/db/generar_sql.py
```

El script usa `TRUNCATE`, que requiere el privilegio `DROP`. Por eso se ejecuta con `bchecker`, que tiene `ALL PRIVILEGES`. Es seguro repetirlo: vacía la tabla antes de insertar.

### 6.2. Código

```python
import os
import sys

import pandas as pd
import mysql.connector

# Credenciales y destino por variables de entorno
DB_HOST = os.environ.get("DB_HOST", "10.10.20.10")
DB_USER = os.environ.get("DB_USER")
DB_PASSWORD = os.environ.get("DB_PASSWORD")
if not DB_USER or not DB_PASSWORD:
    sys.exit("Define las variables de entorno DB_USER y DB_PASSWORD")

# 1. Cargar el archivo CSV respetando la codificación UTF-16
csv_path = 'datos.csv'
df = pd.read_csv(csv_path, encoding='utf-16')

# Limpiar caracteres invisibles BOM en register_id
df['register_id'] = df['register_id'].astype(str).str.replace('\ufeff', '').str.strip()

# 2. Conectar a MySQL forzando UTF-8 MB4 y desactivando SSL legado (Python 3.12)
conn = mysql.connector.connect(
    host=DB_HOST,
    user=DB_USER,
    password=DB_PASSWORD,
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
print(f"¡Proceso completado! Leídos {len(df)} registros, importados {count}, con codificación UTF-8 limpia.")
cursor.close()
conn.close()
```

> **Nota:** esta versión del script (credenciales por variables de entorno y destino `10.10.20.10`) está revisada pero **no se ha ejecutado**. Debe repetirse la prueba **AUT-03** con ella. El destino ya no es `127.0.0.1` porque MySQL solo escucha en la IP de la Intranet (sección 4.2).

---

## 7. Verificación de la Integridad de los Datos

La contraseña se pasa con `MYSQL_PWD` para evitar el aviso de MySQL y que quede en el listado de procesos.

```bash
# Total de registros (esperado: 2710)
docker exec -e MYSQL_PWD=bchecker121 mysql_bcn_db \
    mysql -ubchecker -e "USE bcn_educacio; SELECT COUNT(*) AS Total FROM equipaments;"

# Codificación del servidor y de la base de datos (esperado: utf8mb4)
docker exec -e MYSQL_PWD=bchecker121 mysql_bcn_db \
    mysql -ubchecker -e "SHOW VARIABLES LIKE 'character_set_%';"

# Ausencia de mojibake: los distritos deben mostrar tildes y caracteres catalanes
docker exec -e MYSQL_PWD=bchecker121 mysql_bcn_db \
    mysql -ubchecker -e "USE bcn_educacio; SELECT DISTINCT district FROM equipaments ORDER BY district;"

# Duplicados en el identificador de registro (esperado: ningún resultado)
docker exec -e MYSQL_PWD=bchecker121 mysql_bcn_db \
    mysql -ubchecker -e "USE bcn_educacio; SELECT register_id, COUNT(*) FROM equipaments GROUP BY register_id HAVING COUNT(*) > 1;"
```

| Comprobación | Resultado esperado |
| :--- | :--- |
| `COUNT(*)` | `2710` |
| `character_set_*` | `utf8mb4` en cliente, conexión, base de datos y resultados |
| Distritos | Nombres con acentos correctos (por ejemplo, *Gràcia*, *Sant Martí*) |
| Duplicados de `register_id` | Ninguno |

---

## 8. Mantenimiento y Resolución de Problemas

### 8.1. Operaciones habituales

| Tarea | Comando |
| :--- | :--- |
| Estado de los contenedores | `docker compose ps` |
| Registro de MySQL | `docker logs -f mysql_bcn_db` |
| Copia lógica de la base de datos | Ver `02-Redes-y-seguridad/Contingencia-y-pruebas.md`, sección 1.2 |
| Copia de seguridad del contenedor | `vzdump 106 --mode snapshot --compress zstd --storage local` (desde `pvexx`) |
| **Reinicializar la base de datos (borra los datos)** | `docker compose down -v && docker compose up -d`, y volver a ejecutar `generar_sql.py` |

### 8.2. Resolución de problemas

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| Los cambios de `init.sql` no se aplican | Los scripts solo se ejecutan con el volumen vacío | Recrear el volumen (`down -v`, ver 8.1). Atención: elimina los datos. |
| `init.sql` aparece como carpeta | Se levantó el Compose sin el fichero | `docker compose down -v`, eliminar la carpeta `init.sql`, restaurar el fichero y volver a levantar. |
| `Access denied for user` | Credenciales distintas de las del primer arranque | Las contraseñas se fijan al crear el volumen. Revisar el `.env` o recrear el volumen. |
| `WEB-NCC` no conecta | Regla de cortafuegos o puerto no publicado en la IP de la Intranet | Desde `WEB-NCC`: `nc -zv 10.10.20.10 3306`. En `B-NCC`: `ss -tln` debe mostrar `10.10.20.10:3306`. |
| `generar_sql.py` no conecta | Variable `DB_HOST` o credenciales incorrectas | Comprobar `DB_HOST`, `DB_USER` y `DB_PASSWORD`. |
| `AttributeError: ... wrap_socket` | Conector antiguo con Python 3.12 | Usar `ssl_disabled=True` (ya incluido) o actualizar `mysql-connector-python` a una versión con soporte de Python 3.12. |
| Caracteres corruptos (*mojibake*) | Lectura del CSV sin `encoding='utf-16'` o conexión sin `utf8mb4` | Revisar el script y ejecutar las comprobaciones de la sección 7. |

---

## 9. Limitaciones Conocidas y Mejoras Previstas

1. **Permisos excesivos de la aplicación.** `user_bcn` conserva `ALL PRIVILEGES`, de modo que una inyección o un fallo en la aplicación podría borrar tablas. La aplicación solo necesita las operaciones del CRUD:

   ```sql
   REVOKE ALL PRIVILEGES ON bcn_educacio.* FROM 'user_bcn'@'%';
   GRANT SELECT, INSERT, UPDATE, DELETE ON bcn_educacio.* TO 'user_bcn'@'%';
   ```

   Esto es compatible con el script de importación porque este usa `bchecker`.

2. **Tabla única desnormalizada.** Distrito, barrio y categoría se repiten como texto en cada fila. Una mejora del último MVP es segmentar la base de datos y optimizar las relaciones 1–N y N–M. Propuesta:

   | Tabla | Relación |
   | :--- | :--- |
   | `districts` | 1 distrito → N barrios |
   | `neighborhoods` | 1 barrio → N equipamientos |
   | `categories` | N–M con `equipaments` mediante una tabla intermedia, si un equipamiento tiene varias categorías |
   | `equipaments` | Referencia a barrio con clave foránea |

3. **Sin índices ni restricciones de unicidad** sobre `district`, `neighborhood` ni `register_id`. Añadir un índice en `district` aceleraría el filtro del portal.
4. **Conexión sin cifrar.** `ssl_disabled=True` y el tráfico entre `WEB-NCC` y MySQL viajan en claro. Es aceptable en la red interna del laboratorio; en producción debería activarse TLS en MySQL.
5. **Copia de seguridad en el mismo nodo.** Ver las recomendaciones de `Contingencia-y-pruebas.md`.
