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
├── db/                 # Volumen de almacenamiento persistente de MySQL
└── html/               # Directorio raíz del código fuente web
    └── index.php       # Aplicación web principal con buscador, filtros y gestión CRUD
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

## 7. Aplicación Web CRUD y Gestión (`html/index.php`)

Aplicación web desarrollada en PHP y maquetada con Bootstrap 5 que proporciona interfaz gráfica para listar equipamientos, realizar filtrados avanzados (por texto o por distrito) y ejecutar operaciones CRUD complete (Crear, Leer, Editar y Eliminar):

```php
<?php
// 1. Forzar cabecera HTTP en UTF-8
header('Content-Type: text/html; charset=utf-8');

// Conexión a la base de datos MySQL en Docker
$mysqli = new mysqli("mysql_bcn_db", "user_bcn", "pirineus", "bcn_educacio");

if ($mysqli->connect_error) {
    die("Error de conexión: " . htmlspecialchars($mysqli->connect_error, ENT_QUOTES, 'UTF-8'));
}

// Configurar el juego de caracteres de la sesión MySQL a utf8mb4
$mysqli->set_charset("utf8mb4");

/**
 * Función para detectar y convertir cadenas en UTF-16 a UTF-8 y limpiar el BOM
 */
function normalizar_texto($texto) {
    if ($texto === null \vert{}\vert{}$texto === '') {
        return '';
    }
    if (!mb_check_encoding($texto, 'UTF-8')) {
        $texto = mb_convert_encoding($texto, 'UTF-8', 'UTF-16LE');
    }
    $texto = preg_replace('/\x{FEFF}/u', '',$texto);
    return trim($texto);
}

// 2. ACCIÓN: ELIMINAR REGISTRO
if (isset($_GET['action']) && $_GET['action'] === 'delete' && isset($_GET['id'])) {
    $id_del = intval($_GET['id']);
    $stmt =$mysqli->prepare("DELETE FROM equipaments WHERE id = ?");
    $stmt->bind_param("i", $id_del);
    $stmt->execute();$stmt->close();
    
    header("Location: index.php");
    exit;
}

// 3. ACCIÓN: CREAR O EDITAR REGISTRO (POST)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {$id             = !empty($_POST['id']) ? intval($_POST['id']) : null;
    $register_id    = normalizar_texto($_POST['register_id'] ?? '');
    $name           = normalizar_texto($_POST['name'] ?? '');
    $address_road   = normalizar_texto($_POST['address_road'] ?? '');
    $address_number = normalizar_texto($_POST['address_number'] ?? '');
    $neighborhood   = normalizar_texto($_POST['neighborhood'] ?? '');
    $district       = normalizar_texto($_POST['district'] ?? '');
    $category       = normalizar_texto($_POST['category'] ?? '');

    if ($id) {
        $stmt =$mysqli->prepare("UPDATE equipaments SET register_id=?, name=?, address_road=?, address_number=?, neighborhood=?, district=?, category=? WHERE id=?");
        $stmt->bind_param("sssssssi", $register_id,$name, $address_road,$address_number, $neighborhood,$district, $category,$id);
    } else {
        $stmt =$mysqli->prepare("INSERT INTO equipaments (register_id, name, address_road, address_number, neighborhood, district, category) VALUES (?, ?, ?, ?, ?, ?, ?)");
        $stmt->bind_param("sssssss", $register_id, $name,$address_road, $address_number,$neighborhood, $district,$category);
    }
    
    $stmt->execute();$stmt->close();

    header("Location: index.php");
    exit;
}

// 4. CONSULTA Y FILTRADO
$busqueda = isset($_GET['q']) ? normalizar_texto($_GET['q']) : '';$distrito = isset($_GET['distrito']) ? normalizar_texto($_GET['distrito']) : '';

$distritos_query =$mysqli->query("SELECT DISTINCT district FROM equipaments WHERE district IS NOT NULL AND district != '' ORDER BY district ASC");

$sql = "SELECT id, register_id, name, address_road, address_number, neighborhood, district, category FROM equipaments WHERE 1=1";
$params = [];$types = "";

if ($busqueda !== '') {$sql .= " AND (name LIKE ? OR address_road LIKE ? OR category LIKE ?)";
    $like_busqueda = "\%" . $busqueda . "%";
    $params[] =$like_busqueda;
    $params[] =$like_busqueda;
    $params[] =$like_busqueda;
    $types .= "sss";
}

if ($distrito !== '') {$sql .= " AND district = ?";
    $params[] =$distrito;
    $types .= "s";
}

$sql .= " ORDER BY id DESC LIMIT 50";

if (!empty($params)) {$stmt = $mysqli->prepare($sql);
    $stmt->bind_param($types, ...$params);$stmt->execute();
    $resultado =$stmt->get_result();
} else {
    $resultado = $mysqli->query($sql);
}

$total_query =$mysqli->query("SELECT COUNT(*) AS total FROM equipaments");
$total_registros =$total_query->fetch_assoc()['total'];
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BCN Educación - Buscador y Gestión CRUD</title>
    <link href="[https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css](https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css)" rel="stylesheet">
    <link href="[https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css](https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css)" rel="stylesheet">
</head>
<body class="bg-light">
    <div class="container py-4">
        
        <!-- Encabezado -->
        <div class="d-flex justify-content-between align-items-center mb-4">
            <div>
                <h1 class="h2 text-primary m-0"><i class="bi bi-building"></i> Educación Barcelona</h1>
                <p class="text-muted m-0">Mostrando resultados sobre un total de <strong><?= number_format($total_registros) ?></strong> equipamientos.</p>
            </div>
            <button class="btn btn-success" onclick="abrirModalCrear()">
                <i class="bi bi-plus-lg"></i> Añadir Equipamiento
            </button>
        </div>

        <!-- Formulario de Filtros -->
        <form method="GET" class="card card-body mb-4 shadow-sm">
            <div class="row g-3">
                <div class="col-md-6">
                    <input type="text" name="q" class="form-control" placeholder="Buscar por nombre, calle o categoría..." value="<?= htmlspecialchars($busqueda, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>">
                </div>
                <div class="col-md-4">
                    <select name="distrito" class="form-select">
                        <option value="">-- Todos los distritos --</option>
                        <?php while ($d =$distritos_query->fetch_assoc()): ?>
                            <?php $dist_val = normalizar_texto($d['district']); ?>
                            <option value="<?= htmlspecialchars($dist_val, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>" <?= $distrito ===$dist_val ? 'selected' : '' ?>>
                                <?= htmlspecialchars($dist_val, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
                            </option>
                        <?php endwhile; ?>
                    </select>
                </div>
                <div class="col-md-2 d-flex gap-2">
                    <button type="submit" class="btn btn-primary w-100">Filtrar</button>
                    <?php if ($busqueda !== '' \vert{}\vert{}$distrito !== ''): ?>
                        <a href="index.php" class="btn btn-outline-secondary" title="Limpiar filtro"><i class="bi bi-x-lg"></i></a>
                    <?php endif; ?>
                </div>
            </div>
        </form>

        <!-- Tabla de Datos -->
        <div class="card shadow-sm">
            <div class="table-responsive">
                <table class="table table-hover table-striped align-middle mb-0">
                    <thead class="table-dark">
                        <tr>
                            <th>Reg. ID</th>
                            <th>Nombre</th>
                            <th>Dirección</th>
                            <th>Barrio</th>
                            <th>Distrito</th>
                            <th>Categoría</th>
                            <th class="text-center">Acciones</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php if ($resultado &&$resultado->num_rows > 0): ?>
                            <?php while ($row =$resultado->fetch_assoc()): ?>
                                <?php
                                    $reg_id    = normalizar_texto($row['register_id']);$nombre    = normalizar_texto($row['name']);$calle     = normalizar_texto($row['address_road']);$numero    = normalizar_texto($row['address_number']);$barrio    = normalizar_texto($row['neighborhood']);$dist      = normalizar_texto($row['district']);$categoria = normalizar_texto($row['category']);$direccion = trim($calle . ' ' .$numero);
                                ?>
                                <tr>
                                    <td><code><?= htmlspecialchars($reg_id, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></code></td>
                                    <td><strong><?= htmlspecialchars($nombre, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></strong></td>
                                    <td><?= htmlspecialchars($direccion !== '' ?$direccion : '-', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></td>
                                    <td><?= htmlspecialchars($barrio !== '' ?$barrio : '-', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></td>
                                    <td><?= htmlspecialchars($dist !== '' ?$dist : '-', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></td>
                                    <td><span class="badge bg-info text-dark"><?= htmlspecialchars($categoria !== '' ?$categoria : 'General', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></span></td>
                                    <td class="text-center text-nowrap">
                                        <button class="btn btn-sm btn-outline-warning me-1" onclick='abrirModalEditar(<?= json_encode([
                                            'id' => $row['id'],
                                            'register_id' => $reg_id,
                                            'name' => $nombre,
                                            'address_road' => $calle,
                                            'address_number' => $numero,
                                            'neighborhood' => $barrio,
                                            'district' => $dist,
                                            'category' => $categoria
                                        ], JSON_HEX_APOS | JSON_HEX_QUOT | JSON_UNESCAPED_UNICODE) ?>)'>
                                            <i class="bi bi-pencil-fill"></i> Editar
                                        </button>
                                        <a href="?action=delete&id=<?= $row['id'] ?>" class="btn btn-sm btn-outline-danger" onclick="return confirm('¿Estás seguro de que deseas eliminar este equipamiento?')">
                                            <i class="bi bi-trash-fill"></i> Eliminar
                                        </a>
                                    </td>
                                </tr>
                            <?php endwhile; ?>
                        <?php else: ?>
                            <tr>
                                <td colspan="7" class="text-center py-4 text-muted">No se encontraron registros que coincidan con la búsqueda.</td>
                            </tr>
                        <?php endif; ?>
                    </tbody>
                </table>
            </div>
        </div>
    </div>

    <!-- Modal Formulario CRUD -->
    <div class="modal fade" id="modalEquipamiento" tabindex="-1" aria-hidden="true">
        <div class="modal-dialog modal-lg">
            <div class="modal-content">
                <form method="POST" action="index.php">
                    <div class="modal-header">
                        <h5 class="modal-title" id="modalTitulo">Añadir Equipamiento</h5>
                        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Cerrar"></button>
                    </div>
                    <div class="modal-body">
                        <input type="hidden" name="id" id="form_id">
                        <div class="row g-3">
                            <div class="col-md-4">
                                <label class="form-label">Reg. ID</label>
                                <input type="text" name="register_id" id="form_register_id" class="form-control" required placeholder="Ej: 9940001">
                            </div>
                            <div class="col-md-8">
                                <label class="form-label">Nombre del centro</label>
                                <input type="text" name="name" id="form_name" class="form-control" required placeholder="Ej: Escola BCN">
                            </div>
                            <div class="col-md-8">
                                <label class="form-label">Calle / Dirección</label>
                                <input type="text" name="address_road" id="form_address_road" class="form-control" placeholder="Ej: Av. Diagonal">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Número</label>
                                <input type="text" name="address_number" id="form_address_number" class="form-control" placeholder="Ej: 123">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Barrio</label>
                                <input type="text" name="neighborhood" id="form_neighborhood" class="form-control" placeholder="Ej: la Vila de Gràcia">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Distrito</label>
                                <input type="text" name="district" id="form_district" class="form-control" placeholder="Ej: Gràcia">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Categoría</label>
                                <input type="text" name="category" id="form_category" class="form-control" placeholder="Ej: Educació primària">
                            </div>
                        </div>
                    </div>
                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancelar</button>
                        <button type="submit" class="btn btn-primary" id="btnGuardar">Guardar</button>
                    </div>
                </form>
            </div>
        </div>
    </div>

    <script src="[https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js](https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js)"></script>
    <script>
        const modalElement = document.getElementById('modalEquipamiento');
        const bsModal = new bootstrap.Modal(modalElement);

        function abrirModalCrear() {
            document.getElementById('modalTitulo').innerText = 'Añadir Nuevo Equipamiento';
            document.getElementById('form_id').value = '';
            document.getElementById('form_register_id').value = '';
            document.getElementById('form_name').value = '';
            document.getElementById('form_address_road').value = '';
            document.getElementById('form_address_number').value = '';
            document.getElementById('form_neighborhood').value = '';
            document.getElementById('form_district').value = '';
            document.getElementById('form_category').value = '';
            document.getElementById('btnGuardar').innerText = 'Crear Equipamiento';
            bsModal.show();
        }

        function abrirModalEditar(data) {
            document.getElementById('modalTitulo').innerText = 'Editar Equipamiento #' + data.id;
            document.getElementById('form_id').value = data.id;
            document.getElementById('form_register_id').value = data.register_id || '';
            document.getElementById('form_name').value = data.name || '';
            document.getElementById('form_address_road').value = data.address_road || '';
            document.getElementById('form_address_number').value = data.address_number || '';
            document.getElementById('form_neighborhood').value = data.neighborhood || '';
            document.getElementById('form_district').value = data.district || '';
            document.getElementById('form_category').value = data.category || '';
            document.getElementById('btnGuardar').innerText = 'Actualizar Cambios';
            bsModal.show();
        }
    </script>
</body>
</html>
```
