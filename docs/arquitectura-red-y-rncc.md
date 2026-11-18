# Arquitectura de Red, Enrutamiento Centralizado y Evolución desde `R-NCC`

Este documento detalla la topología de red definitiva desplegada sobre **Proxmox VE (`pvexx`)** en **IsardVDI**, la justificación técnica de la migración del router inicial **`R-NCC`** hacia el hipervisor, y la configuración persistente de interfaces, cortafuegos (`iptables`), DNS y DHCP (`dnsmasq`).

---

## 1. Evolución Arquitectónica: De `R-NCC` al Enrutamiento en Proxmox

### 1.1. Diseño Inicial (`CT 103 - R-NCC`)
En la fase inicial del proyecto se diseñó un contenedor LXC dedicado llamado **`R-NCC`** (con tres interfaces virtuales: `eth0` hacia `vmbr0` WAN, `eth1` hacia `vmbr1` DMZ `10.10.10.1/24` y `eth2` hacia `vmbr2` Intranet `10.10.20.1/24`) encargado de ejecutar Netplan, reglas NAT de `iptables` y el servidor DNS/DHCP (`dnsmasq`).

### 1.2. Incidencia Detectada en IsardVDI (*Anti-MAC Spoofing*)
Durante las pruebas de integración en el entorno virtualizado de **IsardVDI**, se detectó que los clientes conectados a la red privada (`enp3s0`) no recibían dirección IP por DHCP desde `R-NCC`, a pesar de que el demonio `dnsmasq` procesaba correctamente las peticiones `DHCP DISCOVER` y emitía las respuestas `DHCP OFFER`.

* **Causa raíz:** El switch virtual de IsardVDI aplica una política de seguridad de capa 2 (**Port Security / Anti-MAC Spoofing**) que registra únicamente la dirección MAC original de la interfaz `enp3s0` asignada a la máquina virtual Proxmox, descartando cualquier trama Ethernet saliente cuya MAC de origen sea distinta. Como el contenedor LXC `R-NCC` operaba en modo puente con su propia dirección MAC virtual, IsardVDI bloqueaba sus respuestas DHCP antes de que alcanzaran a los clientes.

### 1.3. Solución Adoptada (Centralización en Proxmox `pvexx`)
Se retiró el contenedor `R-NCC` (`CT 103` detenido como referencia histórica) y se trasladaron las funciones de **enrutamiento IPv4, NAT (`iptables`), cortafuegos y servidor DNS/DHCP (`dnsmasq`)** directamente al nodo anfitrión **Proxmox VE (`pvexx`)**. De este modo, todo el tráfico hacia la red de clientes sale por `vmbr2` utilizando la dirección MAC autorizada de `enp3s0`, resolviendo el bloqueo de capa 2 y reduciendo la latencia entre subredes.

---

## 2. Topología de Red y Segmentación

```text
                       [ Internet / IsardVDI WAN ]
                                    │
                             enp1s0 / vmbr0 (DHCP)
                                    │
   [ Cliente VPN ] ──► enp2s0 (10.2.169.223 / MTU 1446)
                         │  (DNAT 80/443 -> 10.10.10.20)
                         ▼
              ┌─────────────────────────────────────┐
              │       PROXMOX VE (pvexx)            │
              │  Router · Gateway · NAT · dnsmasq   │
              └───┬──────────────┬──────────────┬───┘
                  │              │              │
       vmbr1 (10.10.10.1)   vmbr1:0 (10.10.20.1)   vmbr2 + enp3s0 (192.168.2.1)
       Zona DMZ             Zona Intranet          Red Clientes Locales (DHCP)
         │                    │                      │
         ├──► 10.10.10.10     └──► 10.10.20.10       └──► 192.168.2.50 - .150
         │    F-NCC (FTP)          B-NCC (MySQL)          CLI-LINUX / CLI-WIN
         ├──► 10.10.10.20
         │    WEB-NCC (Web)
         └──► 10.10.10.30
              LDAP-NCC (LDAP)
```

---

## 3. Configuración Persistente en Proxmox VE (`pvexx`)

### 3.1. Habilitación de Reenvío IP en el Kernel (`/etc/sysctl.conf`)
```ini
net.ipv4.ip_forward=1
```

### 3.2. Interfaces, Puentes y Reglas `iptables` (`/etc/network/interfaces`)
La configuración incluye la redirección de puertos (`DNAT`) para acceso desde la VPN (`enp2s0`), las reglas `FORWARD` en primera posición (`-I FORWARD 1`), el ajuste `TCPMSS` para evitar bloqueos por fragmentación MTU (`1446`) y el alias `vmbr1:0` (`10.10.20.1/24`) para comunicar `WEB-NCC` con `B-NCC`:

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

# 2. Interfaz VPN (IsardVDI)
auto enp2s0
iface enp2s0 inet dhcp
	mtu 1446
	post-up sysctl -w net.ipv4.ip_forward=1
	post-up iptables -t nat -A PREROUTING -i enp2s0 -p tcp --dport 80 -j DNAT --to-destination 10.10.10.20:80
	post-up iptables -t nat -A PREROUTING -i enp2s0 -p tcp --dport 443 -j DNAT --to-destination 10.10.10.20:443
	post-up iptables -I FORWARD 1 -i enp2s0 -o vmbr1 -j ACCEPT
	post-up iptables -I FORWARD 1 -i vmbr1 -o enp2s0 -m state --state RELATED,ESTABLISHED -j ACCEPT
	post-up iptables -I FORWARD 1 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
	post-down iptables -t nat -D PREROUTING -i enp2s0 -p tcp --dport 80 -j DNAT --to-destination 10.10.10.20:80
	post-down iptables -t nat -D PREROUTING -i enp2s0 -p tcp --dport 443 -j DNAT --to-destination 10.10.10.20:443
	post-down iptables -D FORWARD -i enp2s0 -o vmbr1 -j ACCEPT
	post-down iptables -D FORWARD -i vmbr1 -o enp2s0 -m state --state RELATED,ESTABLISHED -j ACCEPT
	post-down iptables -D FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

iface enp3s0 inet manual
	mtu 1446

# 3. Red Servidores Interna / DMZ (WEB-NCC, F-NCC, LDAP-NCC)
auto vmbr1
iface vmbr1 inet static
	address 10.10.10.1/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0
	post-up iptables -A FORWARD -i vmbr1 -o vmbr1 -j ACCEPT
	post-up iptables -t nat -A POSTROUTING -o vmbr1 -j MASQUERADE
	post-down iptables -D FORWARD -i vmbr1 -o vmbr1 -j ACCEPT
	post-down iptables -t nat -D POSTROUTING -o vmbr1 -j MASQUERADE

# 3.1. Pasarela Subred Intranet (B-NCC 10.10.20.10)
auto vmbr1:0
iface vmbr1:0 inet static
	address 10.10.20.1/24

# 4. Red Clientes Locales (Conectada a enp3s0 - Pasarela 192.168.2.1)
auto vmbr2
iface vmbr2 inet static
	address 192.168.2.1/24
	bridge-ports enp3s0
	bridge-stp off
	bridge-fd 0
	mtu 1446
	post-up iptables -t nat -A POSTROUTING -o vmbr2 -j MASQUERADE
	post-up iptables -t nat -A POSTROUTING -o vmbr0 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -o vmbr2 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -o vmbr0 -j MASQUERADE

source /etc/network/interfaces.d/*
```

### 3.3. Servicio DNS y DHCP (`/etc/dnsmasq.conf`)
```ini
interface=enp2s0
interface=vmbr2
interface=vmbr1

domain-needed
bogus-priv
no-resolv
no-poll

server=1.1.1.1
server=8.8.8.8

domain=educacion.local
local=/educacion.local/
expand-hosts

# Resolución DNS interna
address=/educacion.local/10.10.10.20
address=/www.educacion.local/10.10.10.20
address=/web-ncc/10.10.10.20
address=/f-ncc/10.10.10.10
address=/ldap-ncc/10.10.10.30
address=/b-ncc/10.10.20.10
addn-hosts=/etc/hosts

cache-size=1000
log-queries
log-facility=/var/log/dnsmasq.log
except-interface=lo

# DHCP para clientes en vmbr2 + inyección de ruta estática (Opción 121)
dhcp-range=interface:vmbr2,192.168.2.50,192.168.2.150,255.255.255.0,12h
dhcp-option=option:router,192.168.2.1
dhcp-option=option:dns-server,192.168.2.1
dhcp-option=option:domain-name,"educacion.local"
dhcp-option=121,10.10.10.0/24,192.168.2.1
```

---

## 4. Configuración de Red de los Nodos Internos (Netplan)

### 4.1. Servidor Web (`WEB-NCC` — `10.10.10.20`)
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

### 4.2. Servidor FTP (`F-NCC` — `10.10.10.10`)
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

### 4.3. Servidor de Base de Datos (`B-NCC` — `10.10.20.10`)
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
