# 06. Monitorización de Infraestructura en Tiempo Real (Uptime Kuma)

## 1. Arquitectura y Diseño del Servicio

Para garantizar la observabilidad continua de los servicios críticos desplegados entre la **DMZ (`10.10.10.0/24`)** y la **Intranet (`10.10.20.0/24`)**, se ha implementado un sistema de monitorización activa basado en **Uptime Kuma (`v2.5.12`)**.

* **Nodo de ejecución (`B-NCC` — `10.10.20.10`):** El servicio corre sobre **Docker Engine** aprovechando la infraestructura de contenedores ya operativa en la subred interna, con almacenamiento persistente en **SQLite** para evitar sobrecargar el motor MySQL de producción.
* **Publicación segura (`WEB-NCC` — `10.10.10.20`):** En lugar de exponer el servicio en texto claro hacia la red de clientes, **Nginx** actúa como proxy inverso TLS (`https://educacion.local:3001`) reutilizando el certificado X.509 de la plataforma y negociando conexiones **WebSockets (Socket.IO)** en tiempo real.
* **Segmentación de red (`pvexx`):** El hipervisor Proxmox controla el tránsito entre subredes mediante reglas `FORWARD` con seguimiento de estado (`ESTABLISHED,RELATED`), restringiendo el acceso únicamente al puerto `3001/TCP` de `B-NCC`.

---

## 2. Despliegue del Contenedor y Resolución DNS (`B-NCC`)

### 2.1. Configuración DNS en Docker Engine
Para que los contenedores alojados en `B-NCC` puedan resolver los nombres internos del dominio `educacion.local` sin depender exclusivamente de servidores DNS externos, se configura el demonio de Docker apuntando al servidor DNS interno de Proxmox (`10.10.10.1`):

```bash
cat << 'EOF' > /etc/docker/daemon.json
{
  "dns": ["10.10.10.1", "8.8.8.8"]
}
EOF

systemctl reload docker
```

### 2.2. Lanzamiento del contenedor persistente
Se despliega la imagen oficial `louislam/uptime-kuma:2` con política de reinicio automático y un volumen dedicado (`/app/data`) donde reside la base de datos SQLite y la configuración de monitores:

```bash
docker run -d \
  --restart=always \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  --name uptime-kuma \
  louislam/uptime-kuma:2

# Inyectar resolución local directa del portal en /etc/hosts del contenedor
docker exec -u 0 uptime-kuma sh -c 'echo "10.10.10.20 educacion.local www.educacion.local" >> /etc/hosts'
```

* **Motor de almacenamiento:** `SQLite` (integrado en `/app/data/kuma.db`).
* **Credenciales de administración:** Usuario `bchecker` / Contraseña `bchecker121`.

---

## 3. Enrutamiento y Reglas de Cortafuegos (`pvexx`)

Para permitir que el proxy inverso de `WEB-NCC` (`10.10.10.20`) y las estaciones de administración accedan al puerto `3001/TCP` de `B-NCC` (`10.10.20.10`) manteniendo el aislamiento de la Intranet, se añaden las reglas de ida y retorno en `/etc/network/interfaces` de **`pvexx`** bajo la interfaz `vmbr1:0`:

```text
# Puerta de enlace para la subred Intranet (10.10.20.0/24)
auto vmbr1:0
iface vmbr1:0 inet static
    address 10.10.20.1/24
    # Permitir acceso al panel Uptime Kuma (puerto 3001) en B-NCC
    post-up iptables -I FORWARD 1 -p tcp -d 10.10.20.10 --dport 3001 -j ACCEPT
    post-up iptables -I FORWARD 1 -s 10.10.20.10 -p tcp --sport 3001 -m state --state ESTABLISHED,RELATED -j ACCEPT
    post-down iptables -D FORWARD -p tcp -d 10.10.20.10 --dport 3001 -j ACCEPT
    post-down iptables -D FORWARD -s 10.10.20.10 -p tcp --sport 3001 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

---

## 4. Proxy Inverso HTTPS y Soporte WebSockets (`WEB-NCC`)

Uptime Kuma utiliza **Socket.IO** para actualizar las métricas y latencias en pantalla sin recargar el navegador. En **`WEB-NCC` (`10.10.10.20`)** se define el bloque de servidor `/etc/nginx/conf.d/uptime-kuma.conf` con mapeo dinámico de cabeceras `Upgrade` / `Connection` y almacenamiento en búfer desactivado:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 3001 ssl;
    server_name educacion.local www.educacion.local web-ncc 10.10.10.20 _;

    ssl_certificate /etc/nginx/ssl/educacion.local.crt;
    ssl_certificate_key /etc/nginx/ssl/educacion.local.key;

    location / {
        proxy_pass [http://10.10.20.10:3001](http://10.10.20.10:3001);
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 86400s;
    }
}
```

Tras validar la sintaxis con `nginx -t && systemctl reload nginx`, el panel queda accesible de forma cifrada desde cualquier cliente de la red en **`https://educacion.local:3001`**.

---

## 5. Matriz de Monitores Activos y Página de Estado

Se han configurado **5 sondas de comprobación periódica** cada **60 segundos** que cubren todas las capas de la infraestructura:

| Monitor | Nodo Destino | Tipo de Sonda | Endpoint / Puerto | Parámetros Específicos |
| :--- | :--- | :--- | :--- | :--- |
| **`WEB-NCC` — Portal HTTPS** | `10.10.10.20` | `HTTP(s)` | `https://educacion.local` | Código esperado `200`, ignorar validación CA autofirmada |
| **`LDAP-NCC` — Directorio OpenLDAP** | `10.10.10.30` | `Puerto TCP` | `10.10.10.30:389` | Comprobación de socket LDAP activo |
| **`F-NCC` — Servidor FTP Pasivo** | `10.10.10.10` | `Puerto TCP` | `10.10.10.10:21` | Comprobación de canal de control `vsftpd` |
| **`B-NCC` — Base de Datos MySQL** | `10.10.20.10` | `Puerto TCP` | `10.10.20.10:3306` | Comprobación de instancia MySQL 8.0 |
| **`SSH` — Gestión Remota (`WEB-NCC`)** | `10.10.10.20` | `Puerto TCP` | `10.10.10.20:22` | Verificación de demonio OpenSSH operativo |

### Página de estado pública (Status Page)
* **URL de acceso:** `https://educacion.local:3001/status/educacion`
* **Agrupación:** `Servicios Core & DMZ`
* **Funcionalidad:** Proporciona un tablero visual de solo lectura sin autenticación previa con el porcentaje de disponibilidad histórica (24h / 30d), estado operativo en tiempo real y tiempo de respuesta en milisegundos de cada servicio.

---

## 6. Verificación y Pruebas de Resiliencia

| ID Prueba | Escenario Evaluado | Procedimiento de Verificación | Resultado Esperado | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **MON-01** | Acceso cifrado y sesión WebSocket | Acceder a `https://educacion.local:3001` desde `CLI-LINUX` / `CLI-WIN` e iniciar sesión con `bchecker`. | Carga del panel sin bloqueos de Socket.IO y actualización de métricas en tiempo real. | **Superada** |
| **MON-02** | Detección de caída y recuperación | Detener temporalmente `vsftpd` en `F-NCC` (`systemctl stop vsftpd`), esperar 60s y volver a iniciarlo. | El monitor `F-NCC` cambia a estado **Caído** en menos de 1 minuto y retorna a **Funcional** tras el arranque. | **Superada** |
