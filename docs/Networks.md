# Documentación Técnica de Redes (Networks)

Este apartado detalta la arquitectura de red centralizada en el hipervisor **Proxmox VE (`pvexx`)**, la topología de subredes (DMZ, Intranet y Red de Clientes/VPN) y la configuración paso a paso de cada nodo, contenedor y cliente de la infraestructura.

---

## 1. Topología y Descripción Arquitectónica

La infraestructura se despliega en la plataforma **Isard VDI** utilizando **Proxmox VE (`pvexx`)** como servidor único de virtualización, enrutador centralizado, gateway, firewall, servidor DHCP y servidor DNS. La topología está dividida en tres zonas lógicas:

- **Red de Clientes y VPN (`192.168.2.0/24`):** Gestionada a través del puente `vmbr2` asociando físicamente la interfaz `enp3s0`. Proxmox actúa como pasarela predeterminada (`192.168.2.1`), servidor DHCP dinámico y servidor DNS local para resolver los dominios internos.
- **Zona Desmilitarizada (DMZ / Servidores Internos - `10.10.10.0/24`):** Alojada en `vmbr1`, contiene los contenedores expuestos como el servidor web (`WEB-NCC` / `10.10.10.20`) y el servidor FTP (`Servidor-FTP` / `F-NCC` / `10.10.10.10`).
- **Zona Intranet Privada (`10.10.20.0/24`):** Segmento aislado para servicios internos como la base de datos MySQL (`B-NCC` / `10.10.20.10`).

---

## 2. Hipervisor Proxmox VE (`pvexx`) - Enrutador & Gateway Central

### Descripción de la Configuración

El archivo `/etc/network/interfaces` configura el puente WAN (`vmbr0`), la interfaz para el tráfico VPN (`enp2s0`), el puente para la red de servidores (`vmbr1` / `10.10.10.1`) y el puente para los clientes locales/VPN (`vmbr2` con `enp3s0` / `192.168.2.1`).

Además, implementa mediante reglas de `iptables` en los eventos `post-up` el reenvío de tráfico entre la VPN (`enp2s0`) y la red interna (`vmbr1`), el enmascaramiento NAT (`MASQUERADE`) para `vmbr1`, `vmbr2` y la salida a Internet por `vmbr0`, así como el ajuste de MSS TCP (`TCPMSS --clamp-mss-to-pmtu`) para prevenir bloqueos por fragmentación MTU.

### Archivo `/etc/network/interfaces`

```bash
auto lo
iface lo inet loopback

iface enp1s0 inet manual

# 1. Salida a Internet (IsardVDI)
auto vmbr0
iface vmbr0 inet dhcp
	bridge-ports enp1s0
	bridge-stp off
	bridge-fd 0

# 2. Interfaz VPN
auto enp2s0
iface enp2s0 inet dhcp
	mtu 1446

iface enp3s0 inet manual
	mtu 1446

# 3. Red Servidores Interna (WEB-NCC 10.10.10.20)
auto vmbr1
iface vmbr1 inet static
	address 10.10.10.1/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0
	post-up iptables -A FORWARD -i enp2s0 -o vmbr1 -j ACCEPT
	post-up iptables -A FORWARD -i vmbr1 -o enp2s0 -m state --state RELATED,ESTABLISHED -j ACCEPT
	post-up iptables -t nat -A POSTROUTING -o vmbr1 -j MASQUERADE

# 4. Red Clientes (Conectada a enp3s0 - Pasarela 192.168.2.1)
auto vmbr2
iface vmbr2 inet static
	address 192.168.2.1/24
	bridge-ports enp3s0
	bridge-stp off
	bridge-fd 0
	mtu 1446
	# NAT para enrutamiento local entre clientes y servidores
	post-up iptables -t nat -A POSTROUTING -o vmbr2 -j MASQUERADE
	# Dar salida a Internet a los clientes a través de vmbr0/enp1s0
	post-up iptables -t nat -A POSTROUTING -o vmbr0 -j MASQUERADE
	# Evitar bloqueos TLS por fragmentación MTU
	post-up iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
	post-down iptables -t nat -D POSTROUTING -o vmbr2 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -o vmbr0 -j MASQUERADE

source /etc/network/interfaces.d/*
```

### Servidor DNS & DHCP (`/etc/dnsmasq.conf`)

El servicio `dnsmasq` en Proxmox gestiona las peticiones en las interfaces `enp2s0`, `vmbr2` y `vmbr1`. Redirige las consultas externas a los DNS públicos (`1.1.1.1`, `8.8.8.8`), resuelve de manera autoritativa los dominios locales `educacion.local` y `www.educacion.local` a la IP `10.10.10.20`, y entrega por DHCP direccionamiento dinámico en la red `vmbr2` inyectando la ruta estática a `10.10.10.0/24` a través de la **Opción DHCP 121**.

### Archivo `/etc/dnsmasq.conf`

```ini
# ===================================================================
# CONFIGURACIÓN DE INTERFACES Y ESCUCHA
# ===================================================================
# Escuchar en el bridge de clientes (vmbr2), el bridge interno (vmbr1) y loopback

interface=enp2s0
interface=vmbr2
interface=vmbr1
#listen-address=127.0.0.1,192.168.2.1,10.10.10.1
#bind-interfaces

# ===================================================================
# FILTRADO Y SEGURIDAD DE CONSULTAS DNS
# ===================================================================
# No reenviar nombres sin punto o dominio (ej. "host" en lugar de "host.local")
domain-needed

# No reenviar peticiones de búsqueda inversa para rangos IP privados no asignados
bogus-priv

# Ignorar el archivo /etc/resolv.conf local para evitar bucles de resolución
no-resolv
no-poll

# ===================================================================
# REENVIADORES EXTERNOS (SERVIDORES DNS PARA SALIDA A INTERNET)
# ===================================================================
server=1.1.1.1
server=8.8.8.8

# ===================================================================
# DOMINIO LOCAL Y MAPEADO DE HOSTS
# ===================================================================
# Definir el dominio local autoritativo
domain=educacion.local
local=/educacion.local/
expand-hosts

# Mapeado directo del dominio local al servidor web WEB-NCC
address=/educacion.local/10.10.10.20
address=/www.educacion.local/10.10.10.20

# Leer también las entradas estáticas del archivo /etc/hosts de Proxmox
addn-hosts=/etc/hosts

# ===================================================================
# CONFIGURACIÓN DHCP (OPCIONAL / SI EL CLIENTE REQUIERE IP DINÁMICA)
# ===================================================================
# Descomentar si se desea repartir direccionamiento en la red de clientes:
# dhcp-range=192.168.2.50,192.168.2.150,255.255.255.0,12h
# dhcp-option=option:router,192.168.2.1
# dhcp-option=option:dns-server,192.168.2.1

# ===================================================================
# CACHÉ Y REGISTROS (LOGS)
# ===================================================================
cache-size=1000
log-queries
log-facility=/var/log/dnsmasq.log
except-interface=lo

# Habilitar DHCP para la interfaz del bridge de clientes
dhcp-range=interface:vmbr2,192.168.2.50,192.168.2.150,255.255.255.0,12h

# Opciones DHCP entregadas al cliente
dhcp-option=option:router,192.168.2.1
dhcp-option=option:dns-server,192.168.2.1
dhcp-option=option:domain-name,"educacion.local"

# Entregar automáticamente la ruta a 10.10.10.0/24 usando 192.168.2.1 como pasarela
dhcp-option=121,10.10.10.0/24,192.168.2.1
```

---

## 3. Servidor Web (`WEB-NCC`)

### Descripción de la Configuración

Ubicado en la DMZ (`10.10.10.20`), el contenedor `WEB-NCC` obtiene IP por DHCP en su interfaz `eth0` y tiene asignada la IP estática `10.10.10.20/24` en `eth1`. Cuenta con rutas declaradas explícitamente para responder a las subredes de Intranet (`10.10.20.0/24`) y Clientes/VPN (`192.168.2.0/24`) pasando por el nodo Proxmox (`10.10.10.1`).

### Archivo `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      addresses:
        - 10.10.10.20/24
      routes:
        - to: 10.10.20.0/24
          via: 10.10.10.1
        - to: 192.168.2.0/24
          via: 10.10.10.1
```

---

## 4. Servidor FTP (`Servidor-FTP` / `F-NCC`)

### Descripción de la Configuración

Contenedor alojado en la subred de servidores (`10.10.10.10/24`). Mantiene en la interfaz `eth1` el direccionamiento estático y declara las rutas hacia `10.10.20.0/24` y `192.168.2.0/24` a través de Proxmox (`10.10.10.1`).

### Archivo `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      addresses:
        - 10.10.10.10/24
      routes:
        - to: 10.10.20.0/24
          via: 10.10.10.1
        - to: 192.168.2.0/24
          via: 10.10.10.1
```

---

## 5. Servidor de Base de Datos (`B-NCC`)

### Descripción de la Configuración

Alojado en el segmento Intranet (`10.10.20.10/24`) en la interfaz `net1`. Configura su salida predeterminada (`default`) y su servidor de nombres apuntando al nodo Proxmox (`10.10.20.1`).

### Archivo `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    net0:
      dhcp4: true
    net1:
      dhcp4: no
      addresses:
        - 10.10.20.10/24
      routes:
        - to: default
          via: 10.10.20.1
      nameservers:
        addresses:
          - 10.10.20.1
```

---

## 6. Cliente de Gestión (`CLI-LINUX` / `isard@ubuntu`)

### Descripción de la Configuración

El cliente obtiene la IP dinámica en su interfaz `enp3s0` asignada por `dnsmasq` (`192.168.2.111`). Al conectarse, la **Opción DHCP 121** o la configuración local de Netplan le inyectan automáticamente la ruta estática para alcanzar los servidores internos `10.10.10.0/24` utilizando Proxmox (`192.168.2.1`) como pasarela.

### Archivo `/etc/netplan/01-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: true
      routes:
        - to: 10.10.10.0/24
          via: 192.168.2.1
```

---

## 7. Enrutamiento y Acceso desde la VPN

### Funcionamiento del Tráfico VPN

El acceso remoto mediante la plataforma IsardVDI introduce la conexión del cliente a través de la interfaz VPN `enp2s0` de Proxmox. El flujo del tráfico funciona de la siguiente manera:

1. **Reenvío de Paquetes (`iptables FORWARD`):** Las reglas de Proxmox autorizan el paso de datos entre `enp2s0` y la red de servidores `vmbr1` (`10.10.10.0/24`).
2. **Respuesta de Servidores:** Los contenedores `WEB-NCC` (`10.10.10.20`) y `F-NCC` (`10.10.10.10`) cuentan con rutas específicas configuradas en Netplan que envían las respuestas destinadas a `192.168.2.0/24` a través de Proxmox (`10.10.10.1`).
3. **Traducción de Red (`MASQUERADE`):** Proxmox enmascara el tráfico para asegurar la transmisión de paquetes sobre los puentes locales y la salida a Internet por `vmbr0`.
4. **Resolución de Nombres DNS:** `dnsmasq` responde las peticiones DNS sobre la interfaz VPN `enp2s0`, resolviendo `educacion.local` a `10.10.10.20` de forma directa sin requerir edición del archivo `/etc/hosts` en los clientes.
