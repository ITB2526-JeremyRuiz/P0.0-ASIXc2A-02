# Arquitectura de Red Centralizada, Reglas Persistentes y Evolución desde `R-NCC`

Este documento especifica la topología de red desplegada sobre **Proxmox VE (`pvexx`)** en **IsardVDI**, explica la migración técnica desde el contenedor router inicial **`R-NCC`** hacia el hipervisor, y detalla la configuración persistente de interfaces, cortafuegos (`iptables`), DNS y DHCP (`dnsmasq`).

---

## 1. Evolución Arquitectónica: Del Router `R-NCC` al Enrutamiento en Proxmox

### 1.1. Diseño Inicial de 3 Patas (`CT 103 - R-NCC`)
En la primera fase del proyecto se implementó un contenedor LXC dedicado denominado **`R-NCC` (`CT 103`)** configurado con tres interfaces de red (`eth0` en `vmbr0` para WAN, `eth1` en `vmbr1` con IP `10.10.10.1/24` para la DMZ y `eth2` en `vmbr2` con IP `10.10.20.1/24` para la Intranet), ejecutando Netplan, enmascaramiento NAT y `dnsmasq`.

### 1.2. Limitación Técnica Detectada en IsardVDI (*Anti-MAC Spoofing* y VXLAN)
Al validar la entrega de direcciones IP por DHCP hacia las máquinas cliente conectadas a la red privada de IsardVDI (`enp3s0`), se detectaron dos comportamientos críticos de la plataforma de virtualización:
1. **Filtro de Seguridad de Capa 2 (*Port Security / Anti-MAC Spoofing*):** El switch virtual de IsardVDI registra únicamente la dirección MAC original de la interfaz `enp3s0` asignada a la máquina virtual de Proxmox. Cualquier trama Ethernet saliente con una MAC de origen distinta es descartada automáticamente. Como el contenedor `R-NCC` operaba en modo puente con su propia dirección MAC virtual, sus paquetes `DHCP OFFER` eran bloqueados por IsardVDI antes de llegar a los clientes.
2. **Encapsulación VXLAN (`MTU 1446`):** La red virtualizada de IsardVDI añade cabeceras VXLAN sobre las tramas Ethernet, reduciendo el MTU efectivo a **1446 bytes**. Sin ajuste de *Maximum Segment Size* (`TCPMSS`), las conexiones HTTP/TLS de gran tamaño sufren fragmentación y bloqueos.

### 1.3. Solución Arquitectónica Adoptada
Se detuvo el contenedor `R-NCC` (`CT 103`, conservado como referencia del diseño inicial) y se centralizaron las funciones de **enrutamiento IPv4, NAT/DNAT (`iptables`), ajuste `TCPMSS` y servidor DNS/DHCP (`dnsmasq`)** directamente en el hipervisor **Proxmox VE (`pvexx`)**. De esta manera, todo el tráfico hacia los clientes sale a través de `vmbr2` utilizando la dirección MAC autorizada de `enp3s0`, resolviendo el bloqueo de capa 2 de forma definitiva.

---

## 2. Diagrama de Topología Consolidada

```text
                       [ Internet / IsardVDI WAN ]
                                    │
                             enp1s0 / vmbr0 (DHCP)
                                    │
   [ Cliente VPN ] ──► enp2s0 (10.2.169.223 / MTU 1446)
                         │  (DNAT TCP 80/443 -> 10.10.10.20)
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

### 3.1. Reenvío de Paquetes en el Kernel (`/etc/sysctl.conf`)
```ini
net.ipv4.ip_forward=1
```

### 3.2. Interfaces, Puentes y Reglas `iptables` (`/etc/network/interfaces`)
Se incluyen eventos `post-up` y `post-down` para garantizar la persistencia tras reinicios y evitar reglas duplicadas al recargar con `ifreload -a`:

```bash
auto lo
iface lo inet loopback

iface enp1s0 inet manual

# 1. Salida a Internet (IsardVDI WAN)
auto vmbr0
iface vmbr0 inet dhcp
	bridge-ports enp1s0
	bridge-stp off
	bridge-fd 0

# 2. Interfaz VPN (IsardVDI) - Redirección de puertos hacia WEB-NCC
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

# 3. Zona DMZ - Servidores Internos (WEB-NCC, F-NCC, LDAP-NCC)
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

# 3.1. Zona Intranet - Pasarela para Base de Datos (B-NCC 10.10.20.10)
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

### 3.3. Servidor DNS y DHCP (`/etc/dnsmasq.conf`)
El servicio `dnsmasq` escucha en `enp2s0`, `vmbr2` y `vmbr1`, resuelve de forma autoritativa el dominio `educacion.local` y los nombres de host de la infraestructura, y entrega direccionamiento DHCP en `vmbr2` inyectando la ruta hacia `10.10.10.0/24` mediante la **Opción DHCP 121**:

```ini
interface=enp2s0
interface=vmbr2
interface=vmbr1

domain-needed
bogus-priv
no-resolv
no-poll

# Reenviadores DNS externos para navegación a Internet
server=1.1.1.1
server=8.8.8.8

# Dominio local autoritativo
domain=educacion.local
local=/educacion.local/
expand-hosts

# Resolución DNS interna
address=/educacion.local/10.10.10.20
address=/www.educacion.local/10.10.10.20
address=/web-ncc/10.10.10.20
address=/w-ncc/10.10.10.20
address=/f-ncc/10.10.10.10
address=/ldap-ncc/10.10.10.30
address=/b-ncc/10.10.20.10
addn-hosts=/etc/hosts

cache-size=1000
log-queries
log-facility=/var/log/dnsmasq.log
except-interface=lo

# Rango DHCP para clientes en vmbr2 + inyección de ruta estática (Opción 121)
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

### 4.4. Cliente Linux (`CLI-LINUX` — DHCP en `vmbr2`)
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: true
      mtu: 1446
      dhcp4-overrides:
        use-routes: true
        use-dns: true
```
