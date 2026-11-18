# Base de dades Docker — B-NCC

## Entorn
- Servidor: `B-NCC`
- Adreça: `10.10.20.10`
- Xarxa: intranet
- Motor: MySQL 8.0
- Orquestració: Docker Compose
- Base de dades: `bcn_educacio`
- Taula principal: `equipaments`

La documentació del desplegament també descriu Nginx Alpine i PHP 8.2-FPM al mateix entorn Compose, amb el codi web muntat des del directori `html`.

## Fitxers del projecte
La ruta de treball documentada és `~/db` i inclou:
- `Dockerfile`: imatge PHP-FPM amb extensions de base de dades.
- `docker-compose.yml`: serveis `db`, `php` i `web`.
- `default.conf`: configuració Nginx.
- `datos.csv`: conjunt de dades original.
- `generar_sql.py`: tractament i importació de dades.
- `init.sql`: esquema de la base de dades.
- `inserts.sql`: sentències SQL generades.

## Compose
El desplegament defineix:
- `db`: imatge `mysql:8.0`, amb persistència al volum `db_data`.
- `php`: imatge pròpia basada en `php:8.2-fpm`, amb `pdo`, `pdo_mysql` i `mysqli`.
- `web`: imatge `nginx:alpine`, publica el port 80 i comparteix el directori web.

Les contrasenyes que apareixien en exemples antics s'han omès d'aquesta documentació. Els valors secrets s'han de definir en variables d'entorn o fitxers locals no versionats.

## Esquema
La base de dades `bcn_educacio` utilitza `utf8mb4` i la taula `equipaments` conté camps d'identificació, nom, adreça, barri, districte, codi postal, categoria i coordenades.

## Tractament i importació del CSV
El fitxer original conté 2.710 registres i 40 columnes i està codificat en UTF-16. El procés documentat utilitza Python/pandas per llegir-lo, netejar el BOM i normalitzar els valors abans d'inserir-los a MySQL amb connexió `utf8mb4`.

La importació descrita buida la taula abans de tornar a inserir les files. Aquesta operació elimina els registres existents: s'ha d'executar només després de disposar d'una còpia de seguretat vàlida.

## Comprovació
La consulta de recompte documentada per validar la importació és:

```sql
SELECT COUNT(*) FROM equipaments;
```

El recompte esperat indicat al projecte és de 2.710 registres.
