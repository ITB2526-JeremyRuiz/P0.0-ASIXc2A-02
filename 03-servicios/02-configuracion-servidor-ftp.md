# Configuración del Servidor de Transferencia de Ficheros (`F-NCC`)

Este documento describe el despliegue, la configuración de red pasiva, la validación y el mantenimiento del servidor de transferencia de archivos **`F-NCC`** (`CT 104` / `Servidor-FTP`), alojado en la Zona Desmilitarizada (**DMZ — `10.10.10.10/24`**).

**Índice**

1. [Especificaciones del servicio](#1-especificaciones-del-servicio)
2. [Requisitos previos y dependencias](#2-requisitos-previos-y-dependencias)
3. [Despliegue mediante Docker Compose](#3-despliegue-mediante-docker-compose)
4. [Puesta en marcha y verificación](#4-puesta-en-marcha-y-verificación)
5. [Pruebas del servicio](#5-pruebas-del-servicio)
6. [Mantenimiento y resolución de problemas](#6-mantenimiento-y-resolución-de-problemas)
7. [Limitaciones conocidas y mejoras previstas](#7-limitaciones-conocidas-y-mejoras-previstas)

---

## 1. Especificaciones del Servicio

| Parámetro | Valor |
| :--- | :--- |
| **Contenedor LXC** | `CT 104` (`Servidor-FTP` / `F-NCC`) |
| **Ubicación de red** | Zona DMZ (`vmbr1`), IP estática `10.10.10.10/24`, puerta de enlace `10.10.10.1` |
| **Resolución DNS** | `f-ncc` → `10.10.10.10`, resuelto por `dnsmasq` en `pvexx` |
| **Motor FTP** | Imagen Docker `delfer/proftpd`, contenedor `ftp_bcn` |
| **Modo de transferencia** | Pasivo: puerto de control TCP `21` y rango de datos TCP `60000-60010` |
| **Usuario de auditoría** | `bchecker` (contraseña `bchecker121`, según el estándar del proyecto) |
| **Persistencia** | Volumen Docker `ftp_data`, montado en `/home/bchecker` |

**Alcance del acceso.** El servicio FTP es accesible desde la red de clientes (`192.168.2.0/24`, mediante la ruta que inyecta la Opción DHCP 121) y desde la propia DMZ. No existe redirección (DNAT) del puerto `21` en `pvexx`: el FTP **no** se publica por la interfaz VPN, que solo redirige los puertos `80` y `443` hacia `WEB-NCC`.

**Nomenclatura.** El enunciado exige el *hostname* `F-NCC`. El contenedor Docker lo declara (`hostname: F-NCC`); el nombre del LXC en Proxmox debe coincidir:

```bash
# En pvexx
pct set 104 --hostname F-NCC
```

---

## 2. Requisitos previos y dependencias

Paquetes en el LXC `F-NCC`:

```bash
apt-get update
apt-get install -y docker.io docker-compose-v2
```

Docker dentro de un contenedor LXC necesita que la anidación esté activada. Comprobación desde `pvexx`:

```bash
pct config 104 | grep features
# Esperado: features: nesting=1 (y keyctl=1 si el contenedor no es privilegiado)
```

Configuración de red de `F-NCC`: ver `02-Redes-y-seguridad/01-arquitectura-red.md`, sección 4.2.

---

## 3. Despliegue mediante Docker Compose

Fichero `docker-compose.yml`:

```yaml
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

**Por qué el modo pasivo.** En el modo pasivo es el cliente quien abre las conexiones de datos hacia el servidor. `PASV_ADDRESS` es la dirección que el servidor anuncia al cliente: debe ser la IP real de `F-NCC` (`10.10.10.10`), alcanzable desde la red de clientes. El rango `60000-60010` limita los puertos que hay que permitir en cualquier cortafuegos intermedio.

---

## 4. Puesta en marcha y verificación

### 4.1. Despliegue en `F-NCC`

```bash
# Iniciar el contenedor ProFTPD en segundo plano
docker compose up -d

# Verificar que el contenedor está activo
docker ps

# Comprobar los puertos en escucha (21 y rango pasivo)
ss -tln | grep -E ':(21|6000[0-9]|60010)\b'
```

### 4.2. Validación desde los clientes

**`CLI-LINUX`:**

```bash
# Resolución DNS y conectividad ICMP
ping -c 3 f-ncc

# Fichero de prueba y sesión FTP interactiva
echo "prueba F-NCC" > datos_prueba.txt
ftp 10.10.10.10
# Usuario: bchecker
# Contraseña: bchecker121
# ftp> passive
# ftp> ls
# ftp> put datos_prueba.txt
# ftp> get datos_prueba.txt copia_descargada.txt
# ftp> bye
```

**`CLI-WIN`:** el cliente `ftp.exe` de la línea de comandos de Windows **no soporta el modo pasivo**, por lo que el listado y las transferencias fallarán contra este servidor. Debe usarse un cliente gráfico con modo pasivo (FileZilla o WinSCP) o el Explorador de archivos (`ftp://10.10.10.10`).

---

## 5. Pruebas del servicio

Las pruebas siguientes deben incorporarse a `02-Redes-y-seguridad/Contingencia-y-pruebas.md`. Cada una se documenta con su evidencia en `evidencias/` (ver ese documento, sección 4).

| ID | Prueba | Procedimiento | Resultado esperado | Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **FTP-01** | Resolución DNS del servidor | `nslookup f-ncc` desde un cliente | Devuelve `10.10.10.10` | `evidencias/FTP-01.png` | **Pendiente de ejecutar** |
| **FTP-02** | Autenticación y listado en modo pasivo | `ftp 10.10.10.10`, usuario `bchecker`, `passive`, `ls` (`CLI-LINUX`) | Login correcto y listado sin error | `evidencias/FTP-02.png` | **Pendiente de ejecutar** |
| **FTP-03** | Subida y descarga de un fichero | `put datos_prueba.txt` y `get datos_prueba.txt` | Transferencia completa (`226 Transfer complete`); el contenido coincide (`diff`) | `evidencias/FTP-03.png` | **Pendiente de ejecutar** |
| **FTP-04** | Acceso desde Windows | Conexión con FileZilla o WinSCP a `10.10.10.10` desde `CLI-WIN` | Listado y transferencia correctos en modo pasivo | `evidencias/FTP-04.png` | **Pendiente de ejecutar** |
| **FTP-05** | Persistencia de los datos | Subir un fichero, `docker compose down && docker compose up -d`, volver a listar | El fichero sigue en `/home/bchecker` (volumen `ftp_data`) | `evidencias/FTP-05.png` | **Pendiente de ejecutar** |
| **FTP-06** | Rechazo de credenciales incorrectas y acceso anónimo | Iniciar sesión con una contraseña errónea y con el usuario `anonymous` | Ambos intentos rechazados (`530 Login incorrect`) | `evidencias/FTP-06.png` | **Pendiente de ejecutar** |
| **FTP-07** | Arranque automático tras reinicio | Reiniciar `pvexx` (prueba AUT-04) y ejecutar `docker ps` en `F-NCC` | `ftp_bcn` activo (`restart: always`) | `evidencias/FTP-07.png` | **Pendiente de ejecutar** |

---

## 6. Mantenimiento y resolución de problemas

### 6.1. Operaciones habituales

| Tarea | Comando |
| :--- | :--- |
| Estado del contenedor | `docker ps` |
| Registro del servicio | `docker logs -f ftp_bcn` |
| Reiniciar el servicio | `docker compose restart` |
| Actualizar la imagen | `docker compose pull && docker compose up -d` |
| Copia de seguridad (desde `pvexx`) | `vzdump 104 --mode snapshot --compress zstd --storage local` |

El volumen `ftp_data` forma parte de la copia `vzdump` del contenedor `CT 104`.

### 6.2. Resolución de problemas

| Síntoma | Causa probable | Comprobación y solución |
| :--- | :--- | :--- |
| Tiempo de espera al conectar al puerto 21 | Ruta o servicio caído | `ping f-ncc`; en el cliente, comprobar que existe la ruta `10.10.10.0/24` vía `192.168.2.1`; en `F-NCC`, `docker ps`. |
| El login funciona, pero `ls` o `put` se quedan colgados o dan `425 Can't open data connection` | Rango pasivo bloqueado o `PASV_ADDRESS` incorrecta | Verificar `PASV_ADDRESS=10.10.10.10` y que los puertos `60000-60010` están publicados (`ss -tln`). |
| `530 Login incorrect` | Usuario o contraseña distintos de los definidos | Revisar `FTP_USER` y `FTP_PASSWORD`; recrear el contenedor tras cambiarlos (`docker compose up -d --force-recreate`). |
| Falla solo desde Windows | `ftp.exe` no soporta modo pasivo | Usar FileZilla, WinSCP o el Explorador de archivos. |
| `f-ncc` no resuelve | DNS del cliente | `nslookup f-ncc 192.168.2.1`; revisar `dnsmasq` en `pvexx`. |
| `docker compose` falla al arrancar en el LXC | Anidación no activada | Revisar `features: nesting=1` en `pct config 104`. |

---

## 7. Limitaciones Conocidas y Mejoras Previstas

1. **FTP sin cifrar.** El protocolo FTP transmite usuario, contraseña y datos en claro. Es aceptable en una red interna de pruebas, pero no en un entorno real. Mejoras: FTPS (FTP sobre TLS) con certificado, o SFTP sobre SSH. Coincide con la propuesta de certificados para FTP del último MVP.
2. **Credenciales fijas en `docker-compose.yml`.** `bchecker121` se mantiene porque lo exige el enunciado. Cualquier otra credencial de servicio debe guardarse fuera del repositorio.
3. **Sin acceso desde la VPN.** El puerto `21` no tiene DNAT en `pvexx`; si se necesitara acceso externo habría que redirigirlo junto con el rango pasivo y ajustar `PASV_ADDRESS`.
4. **Pruebas sin ejecutar.** Las pruebas FTP-01 a FTP-07 están definidas pero pendientes de ejecución y evidencia.
