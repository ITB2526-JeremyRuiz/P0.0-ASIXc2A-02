# Documentación Técnica de Redes (Networks)

Este apartado detalla la arquitectura de red centralizada en el hipervisor **Proxmox VE (`pvexx`)**, la topología de subredes (DMZ, Intranet y Red de Clientes/VPN) y la configuración paso a paso de cada nodo, contenedor y cliente de la infraestructura.

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
