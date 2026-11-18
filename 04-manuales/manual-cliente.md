# Manual de Acceso, Uso y Soporte para Clientes (`educacion.local`)

Esta guía proporciona las instrucciones paso a paso para conectar estaciones cliente (Windows y Linux) a la infraestructura, acceder tanto en red local como de forma remota mediante VPN, operar sobre el portal web **Educación Barcelona**, utilizar el servicio de transferencia FTP y resolver incidencias habituales.

---

## 1. Requisitos Previos del Equipo Cliente

* **Sistemas operativos soportados:** Windows 10/11 o distribuciones Linux modernas (Ubuntu 22.04/24.04 LTS o superior).
* **Navegador web:** Google Chrome, Mozilla Firefox o Microsoft Edge.
* **Conectividad de red en IsardVDI:**
  * **En Red Local (LAN Privada):** Tarjeta de red asociada a la misma red privada que la interfaz `enp3s0` (`vmbr2`) de Proxmox.
  * **En Acceso Remoto (VPN):** Cliente VPN de IsardVDI activo y conectado.

---

## 2. Configuración de Red e Instalación en el Cliente

### 2.1. Acceso en Red Local (DHCP Automático + Opción 121)
El servidor Proxmox (`192.168.2.1`) entrega de forma totalmente automática mediante `dnsmasq` la IP (`192.168.2.50` – `192.168.2.150`), el servidor DNS (`192.168.2.1`) y la ruta estática hacia la red de servidores (`10.10.10.0/24` mediante la **Opción DHCP 121**).

* **Configuración en Clientes Windows:**
  1. Abrir **Panel de control > Redes e Internet > Centro de redes y recursos compartidos > Cambiar configuración del adaptador**.
  2. Hacer clic derecho en el adaptador de red privada > **Propiedades** > **Protocolo de Internet versión 4 (TCP/IPv4)**.
  3. Marcar **Obtener una dirección IP automáticamente** y **Obtener la dirección del servidor DNS automáticamente**.
  4. Para renovar la concesión IP y limpiar la caché DNS desde CMD:
     ```cmd
     ipconfig /release
     ipconfig /renew
     ipconfig /flushdns
     ```

* **Configuración en Clientes Linux (Ubuntu / Netplan):**
  Editar `/etc/netplan/01-netcfg.yaml` con `dhcp4: true` y `mtu: 1446` (necesario por la encapsulación VXLAN de IsardVDI):
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
  Aplicar la configuración:
  ```bash
  sudo netplan apply
  ```

### 2.2. Acceso Remoto por VPN (IsardVDI)
Cuando el usuario accede desde su equipo anfitrión mediante la VPN de IsardVDI, el tráfico entra por la interfaz `enp2s0` de Proxmox (`10.2.169.223`), donde el hipervisor redirige automáticamente los puertos web (`80` y `443`) hacia `WEB-NCC` (`10.10.10.20`).
* **URL de acceso por VPN:** `http://<IP_VPN_PROXMOX>` (ej. `[http://10.2.169.223](http://10.2.169.223)`) o `[http://educacion.local](http://educacion.local)` (si el cliente VPN utiliza `10.2.169.223` como DNS).

---

## 3. Guía de Uso del Portal Web (`[http://educacion.local](http://educacion.local)`)

### 3.1. Inicio de Sesión con Credenciales LDAP
1. Abrir el navegador e introducir **`[http://educacion.local](http://educacion.local)`**.
2. En la pantalla de **Autenticación Institucional**, introducir el identificador de usuario (**UID**, por ejemplo `bchecker` o `admin`) y su contraseña.
3. Pulsar **Iniciar Sesión**.

> **Aviso de Seguridad (`ppolicy`):** El sistema admite un máximo de **3 intentos fallidos**. La pantalla indica los intentos restantes tras cada error. Al tercer fallo consecutivo, la cuenta queda **bloqueada automáticamente durante 15 minutos** (o hasta que un administrador la desbloquee).

### 3.2. Búsqueda y Filtrado de Centros Educativos
* **Búsqueda libre:** Escribir en el campo de búsqueda para filtrar por nombre del centro, calle o tipo de enseñanza.
* **Filtro por distrito:** Seleccionar cualquier distrito municipal de Barcelona en el desplegable y pulsar **Filtrar**.
* **Restaurar listado:** Pulsar el botón **`X`** para limpiar los criterios de filtrado.

### 3.3. Gestión CRUD de Equipamientos
* **Añadir Equipamiento:** Pulsar **`+ Añadir Equipamiento`**, completar los campos (*Reg. ID*, *Nombre*, *Calle*, *Número*, *Barrio*, *Distrito*, *Categoría*) y pulsar **Crear Equipamiento**.
* **Editar Equipamiento:** Pulsar **`Editar`** en la fila del centro deseado, actualizar los datos y confirmar.
* **Eliminar Equipamiento:** Pulsar **`Eliminar`** y aceptar el cuadro de confirmación.

### 3.4. Panel de Administración y Desbloqueo (Grupo `admins`)
Los usuarios pertenecientes al grupo `admins` (distinguidos con la etiqueta amarilla **`Admin`** junto a su nombre):
* Visualizan un **banner superior** con todas las cuentas LDAP bloqueadas en ese momento y pueden liberarlas al instante pulsando **`Desbloquear`**.
* Pueden gestionar altas, bajas y grupos desde la interfaz **LAM** en `[http://10.10.10.30/lam](http://10.10.10.30/lam)`.

---

## 4. Acceso al Servidor de Archivos FTP (`F-NCC`)

Para subir o descargar ficheros de intercambio desde un cliente Windows o Linux:
```bash
ftp 10.10.10.10
# Usuario: bchecker
# Contraseña: bchecker121
```

---

## 5. Mantenimiento Preventivo y Resolución de Problemas

| Incidencia / Síntoma | Causa Probable | Solución |
| :--- | :--- | :--- |
| **El navegador no encuentra `educacion.local`** | El equipo cliente tiene una IP o DNS antiguo en caché. | Ejecutar `ipconfig /renew && ipconfig /flushdns` (Windows) o `sudo resolvectl flush-caches` (Linux) y verificar `nslookup educacion.local` (debe responder `192.168.2.1`). |
| **La página se queda cargando sin terminar** | Desajuste de MTU en el túnel VPN de IsardVDI o IP VPN renovada tras reiniciar Proxmox. | Verificar `mtu 1446` en el cliente. Si accedes por VPN, entrar mediante `[http://10.2.169.223](http://10.2.169.223)`. |
| **Mensaje *"Cuenta bloqueada"* al iniciar sesión** | Se han superado los 3 intentos fallidos configurados en OpenLDAP `ppolicy`. | Esperar 15 minutos o solicitar a un usuario `admins` que pulse **Desbloquear** en el portal web. |
| **El portal carga pero se queda esperando tras pulsar *Iniciar Sesión*** | El servicio LDAP (`10.10.10.30`) o MySQL (`10.10.20.10`) está detenido, o falta la ruta Intranet (`10.10.20.1`). | Avisar al administrador para revisar `pct list` y verificar la interfaz `vmbr1:0` (`10.10.20.1`) en Proxmox. |
