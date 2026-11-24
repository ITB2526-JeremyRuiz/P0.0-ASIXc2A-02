# 07. Geolocalización Interactiva de Centros Educativos (Leaflet + MySQL)

## 1. Arquitectura y Flujo de Datos

Para enriquecer el buscador y gestor CRUD del portal institucional (`https://educacion.local`), se ha integrado un módulo de **geolocalización interactiva en tiempo real** que representa sobre un mapa cartográfico de Barcelona los equipamientos educativos almacenados en la base de datos relacional.

* **Capa de datos (`B-NCC` — `10.10.20.10`):** El contenedor Docker **`mysql_bcn_db`** (`MySQL 8.0`) almacena las coordenadas geográficas en formato **WGS84** (`latitude` y `longitude`) dentro de la tabla `bcn_educacio.equipaments`.
* **Capa de aplicación (`WEB-NCC` — `10.10.10.20`):** El controlador PHP (`/var/www/html/index.php`) ejecuta consultas preparadas (`mysqli`) según los filtros de búsqueda activos y serializa los resultados en formato JSON seguro (`JSON_HEX_TAG | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_HEX_AMP | JSON_UNESCAPED_UNICODE`) para su consumo en el cliente.
* **Capa de visualización (`Leaflet 1.9.4` + `OpenStreetMap`):** El navegador renderiza los marcadores dinámicos y las teselas cartográficas exclusivamente sobre **HTTPS**, evitando bloqueos por contenido mixto (*Mixed Content*) bajo el certificado TLS de `educacion.local`.

---

## 2. Esquema Geográfico en MySQL (`B-NCC`)

La inspección de la instancia **`mysql_bcn_db`** confirma que la tabla **`bcn_educacio.equipaments`** dispone de campos decimales de alta precisión para latitud y longitud junto con los metadatos administrativos de cada centro:

```bash
docker exec -i mysql_bcn_db mysql -u root -p -e "DESCRIBE bcn_educacio.equipaments;"
```

| Campo | Tipo de Dato | Nulo | Clave | Descripción Funcional |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `int` | `NO` | `PRI` | Identificador interno autoincremental |
| `register_id` | `varchar(50)` | `YES` | — | Código oficial de registro del equipamiento |
| `name` | `varchar(255)` | `YES` | — | Denominación del centro educativo |
| `address_road` | `varchar(255)` | `YES` | — | Nombre de la vía pública |
| `address_number` | `varchar(50)` | `YES` | — | Número postal de la vía |
| `neighborhood` | `varchar(100)` | `YES` | — | Barrio de Barcelona |
| `district` | `varchar(100)` | `YES` | — | Distrito municipal |
| `zip_code` | `varchar(20)` | `YES` | — | Código postal |
| `category` | `varchar(255)` | `YES` | — | Tipología o etapa educativa |
| `latitude` | `decimal(10,8)` | `YES` | — | Coordenada de latitud WGS84 (ej. `41.40728343`) |
| `longitude` | `decimal(11,8)` | `YES` | — | Coordenada de longitud WGS84 (ej. `2.15556691`) |

---

## 3. Implementación en el Portal Web (`WEB-NCC`)

### 3.1. Consultas preparadas y persistencia CRUD en PHP
Se ha ampliado la lógica de servidor en `/var/www/html/index.php` (conservando copia previa en `/var/www/html/index.php.bak`) para incorporar las coordenadas tanto en el listado filtrado como en las operaciones de alta y edición mediante sentencias preparadas:

```php
// Persistencia de coordenadas en alta (INSERT) y modificación (UPDATE)
$lat_raw   = trim($_POST['latitude'] ?? '');
$lng_raw   = trim($_POST['longitude'] ?? '');
$latitude  = ($lat_raw !== '' && is_numeric($lat_raw)) ? floatval($lat_raw) : null;
$longitude = ($lng_raw !== '' && is_numeric($lng_raw)) ? floatval($lng_raw) : null;

if ($id) {
    $stmt = $mysqli->prepare("UPDATE equipaments SET register_id=?, name=?, address_road=?, address_number=?, neighborhood=?, district=?, category=?, latitude=?, longitude=? WHERE id=?");
    $stmt->bind_param("sssssssddi", $register_id, $name, $address_road, $address_number, $neighborhood, $district, $category, $latitude, $longitude, $id);
} else {
    $stmt = $mysqli->prepare("INSERT INTO equipaments (register_id, name, address_road, address_number, neighborhood, district, category, latitude, longitude) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)");
    $stmt->bind_param("sssssssdd", $register_id, $name, $address_road, $address_number, $neighborhood, $district, $category, $latitude, $longitude);
}
```

### 3.2. Renderizado dinámico y controles interactivos (Leaflet.js)
Sobre la interfaz **Bootstrap 5** se añade el panel `#tarjeta-mapa` y el botón de acción rápida **Mapa** en cada fila de resultados:

* **Encuadre automático (`fitBounds`):** Al cargar la página o aplicar un filtro por texto o distrito, `L.featureGroup()` agrupa todos los marcadores válidos y ajusta el nivel de zoom automáticamente para abarcar los centros visibles.
* **Localización individual (`enfocarEnMapa(id)`):** Al pulsar el botón **Mapa** en cualquier registro de la tabla, la vista se desplaza suavemente al contenedor cartográfico, centra la cámara sobre el marcador con zoom `17` y despliega su ficha emergente (*popup*) con nombre, categoría, dirección, barrio, distrito y coordenadas GPS.
* **Gestión de visibilidad:** El botón **Ocultar / Mostrar mapa** alterna el despliegue del contenedor ejecutando `mapa.invalidateSize()` para recalcular las teselas sin deformaciones.
* **Edición geográfica en el modal CRUD:** El formulario modal de creación y edición incluye los campos numéricos `Latitud (WGS84)` y `Longitud (WGS84)` (`step="any"`).

```javascript
const mapa = L.map('mapa-equipamientos').setView([41.3874, 2.1686], 12);

L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '&copy; OpenStreetMap contributors'
}).addTo(mapa);
```

---

## 4. Verificación y Pruebas Funcionales

| ID Prueba | Escenario Evaluado | Procedimiento de Verificación | Resultado Esperado | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **GEO-01** | Carga cartográfica sobre HTTPS | Iniciar sesión en `https://educacion.local` y comprobar el panel **Mapa de Geolocalización** y la consola del navegador (`F12`). | Renderizado completo de teselas OpenStreetMap y marcadores sin errores de *Mixed Content*. | **Superada** |
| **GEO-02** | Sincronización con filtros y enfoque individual | Filtrar por un distrito (ej. `Gràcia`) y pulsar el botón **Mapa** sobre uno de los centros listados. | El mapa filtra los marcadores, realiza zoom `17` sobre el centro seleccionado y abre su *popup* informativo. | **Superada** |
| **GEO-03** | Alta CRUD con coordenadas WGS84 | Crear un nuevo equipamiento desde **Añadir Equipamiento** introduciendo latitud `41.3874` y longitud `2.1686`. | El registro se persiste en `bcn_educacio.equipaments` y aparece geolocalizado en el mapa al recargar. | **Superada** |
