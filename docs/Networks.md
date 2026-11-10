# Documentación Técnica de Redes (Networks)

Este apartado detalla la arquitectura de red, la topología de subredes (DMZ e Intranet) y la configuración paso a paso de cada nodo, hipervisor y contenedor del proyecto.

## 1. Topología y Descripción Arquitectónica

La infraestructura se despliega en la plataforma **Isard VDI** sobre un hipervisor **Proxmox VE (`pvexx`)**. La comunicación se divide en dos zonas lógicas aisladas interconectadas por un router virtual (`R-NCC`):

- **Zona Desmilitarizada (DMZ - `10.10.10.0/24`):** Contiene los servicios accesibles o expuestos al exterior a través de la infraestructura perimetral, específicamente el servidor web (`WEB-NCC`) y el servidor de archivos FTP (`Servidor-FTP` / `F-NCC`).

- **Zona Intranet Privada (`10.10.20.0/24`):** Contiene los componentes sensibles y no expuestos directamente al exterior, como la base de datos MySQL (`B-NCC`) y los clientes de gestión (`CLI-LINUX`, `CLI-WIN`).

---

## 2. Hipervisor Proxmox VE (`pvexx`)

### Descripción de la Configuración

El archivo `/etc/network/interfaces` del nodo Proxmox administra la interfaz WAN asignada por Isard (`enp2s0`), la interfaz privada dedicada a la VPN (`enp3s0`) y los puentes virtuales (`vmbr1` para DMZ y `vmbr2` para Intranet). Asimismo, habilita el reenvío de paquetes IPv4 (`ip_forward`) y la traducción de direcciones de red (NAT/MASQUERADE) para dar salida a los contenedores.

### Archivo `/etc/network/interfaces`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```bash
auto lo
iface lo inet loopback

iface enp1s0 inet manual

auto enp2s0
iface enp2s0 inet dhcp
	mtu 1446

auto enp3s0
iface enp3s0 inet static
	address 192.168.2.2/24
	mtu 1446

auto vmbr0
iface vmbr0 inet dhcp
	bridge-ports enp1s0
	bridge-stp off
	bridge-fd 0

auto vmbr1
iface vmbr1 inet static
	address 10.10.10.1/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0
	post-up sysctl -w net.ipv4.ip_forward=1
	post-up iptables -t nat -A POSTROUTING -o enp3s0 -j MASQUERADE
	post-up iptables -t nat -A POSTROUTING -o vmbr1 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -o enp3s0 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -o vmbr1 -j MASQUERADE
	mtu 1446

auto vmbr2
iface vmbr2 inet static
	address 10.10.20.1/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0
	post-up iptables -t nat -A POSTROUTING -o vmbr2 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -o vmbr2 -j MASQUERADE
	mtu 1446

source /etc/network/interfaces.d/*
```

</div>

---

## 3. Router Virtual Inter-VLAN (`R-NCC`)

### Descripción de la Configuración

El nodo `R-NCC` actúa como la pasarela de enrutamiento centralizada entre el segmento DMZ (`10.10.10.0/24`) y la Intranet (`10.10.20.0/24`). Su fichero `/etc/hosts` mantiene la tabla de resolución local de todos los elementos de la topología corporativa.

### Archivo `/etc/netplan/50-cloud-init.yaml`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      dhcp4: no
      addresses:
        - 10.10.10.1/24
    eth2:
      dhcp4: no
      addresses:
        - 10.10.20.1/24
```

</div>

### Archivo `/etc/hosts`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```hosts
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

# --- BEGIN PVE ---
127.0.1.1       R-NCC
# --- END PVE ---

10.10.10.1      R-NCC R
10.10.10.10     F-NCC
10.10.10.20     W-NCC
10.10.20.10     B-NCC
10.10.20.50     CLI-LINUX
10.10.20.51     CLI-WIN
```

</div>

---

## 4. Servidor Web (`WEB-NCC`)

### Descripción de la Configuración

Ubicado en la DMZ (`10.10.10.20`), el contenedor `WEB-NCC` aloja los servicios de aplicación web. En `/etc/hosts`, mapea directamente el dominio `educacion.local` y vincula el hostname `mysql_bcn_db` hacia la IP interna de la base de datos (`10.10.20.10`).

### Archivo `/etc/netplan/50-cloud-init.yaml`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      dhcp4: no
      addresses:
        - 10.10.10.20/24
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses:
          - 10.10.10.1
```

</div>

### Archivo `/etc/hosts`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```hosts
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

# --- BEGIN PVE ---
127.0.1.1       WEB-NCC W-NCC
# --- END PVE ---

10.10.10.1      R-NCC R
10.10.10.10     F-NCC
10.10.10.20     WEB-NCC W-NCC educacion.local
10.10.20.10     B-NCC mysql_bcn_db
10.10.20.50     CLI-LINUX
10.10.20.51     CLI-WIN
```

</div>

---

## 5. Servidor FTP (`Servidor-FTP` / `F-NCC`)

### Descripción de la Configuración

Ubicado en la zona DMZ (`10.10.10.10`), este contenedor proporciona el servicio de transferencia de archivos dentro del perímetro perimetral. Su configuración de Netplan enruta su tráfico predeterminado a través del router `10.10.10.1`, y su tabla local de hosts reconoce la pasarela `R-NCC` y su propio alias de servicio `F-NCC`.

### Archivo `/etc/netplan/50-cloud-init.yaml`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: yes
    eth1:
      dhcp4: no
      addresses:
        - 10.10.10.10/24
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses:
          - 10.10.10.1
```

</div>

### Archivo `/etc/hosts`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```hosts
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

# --- BEGIN PVE ---
10.10.10.10     Servidor-FTP
# --- END PVE ---

10.10.10.1      R-NCC R
10.10.10.10     F-NCC
```

</div>

---

## 6. Servidor de Base de Datos (`B-NCC`)

### Descripción de la Configuración

Alojado de forma aislada dentro de la subred Intranet (`10.10.20.10`), este contenedor almacena el motor de base de datos MySQL. Cuenta con salida hacia la pasarela privada `10.10.20.1` (`R-NCC`), limitando las referencias externas en su archivo `/etc/hosts` únicamente a los nodos perimetrales de la DMZ autorizados (`R-NCC` y `F-NCC`).

### Archivo `/etc/netplan/50-cloud-init.yaml`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```yaml
network:
  version: 2
  renderer: networkd
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

</div>

### Archivo `/etc/hosts`

<div style="background-color: #1e1e1e; border-radius: 8px; padding: 15px; margin: 10px 0;">

```hosts
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

# --- BEGIN PVE ---
192.168.121.3   B-NCC
# --- END PVE ---

10.10.10.1      R-NCC R
10.10.10.10     F-NCC
```

</div>
