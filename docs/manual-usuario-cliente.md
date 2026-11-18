# Manual de Acceso, Uso y Soporte para Clientes (`educacion.local`)

Esta guía describe los requisitos de red, el procedimiento de conexión (en red local y mediante VPN), el uso funcional del portal **Educación Barcelona** y la resolución de incidencias frecuentes para usuarios finales y administradores.

---

## 1. Requisitos Previos del Equipo Cliente

* **Sistemas operativos soportados:** Windows 10/11 o distribuciones Linux modernas (Ubuntu 22.04/24.04 LTS o superior).
* **Navegador web:** Google Chrome, Mozilla Firefox o Microsoft Edge actualizados.
* **Conectividad de red:**
  * **Acceso Local (Red Interna):** Tarjeta de red virtual conectada a la red privada de clientes en IsardVDI (puente `vmbr2` / `enp3s0` de Proxmox).
  * **Acceso Remoto (VPN):** Cliente VPN de IsardVDI activo y conectado al entorno virtual.

---

## 2. Configuración de Red e Instalación en el Cliente

### 2.1. Acceso en Red Local (DHCP Automático)
El servidor Proxmox (`192.168.2.1`) entrega automáticamente mediante `dnsmasq` la dirección IP (`192.168.2.50` – `192.168.2.150`), el servidor DNS (`192.168.2.1`) y la ruta estática hacia la subred de servidores (`10.10.10.0/24` mediante la **Opción DHCP 121**).

* **En Clientes Windows:**
  1. Ir a **Panel de control > Redes e Internet > Cambiar opciones del adaptador**.
  2. Abrir **Propiedades** del adaptador de red privada > **Protocolo de Internet versión 4 (TCP/IPv4)**.
  3. Seleccionar **Obtener una dirección IP automáticamente** y **Obtener la dirección del servidor DNS automáticamente**.
  4. Para forzar la renovación inmediata desde CMD:
     ```cmd
     ipconfig /release
     ipconfig /renew
     ipconfig /flushdns
     ```

* **En Clientes Linux (Ubuntu / Netplan):**
  Configurar `/etc/netplan/01-netcfg.yaml` con DHCP activo y MTU `1446` (ajustado al túnel de IsardVDI):
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
