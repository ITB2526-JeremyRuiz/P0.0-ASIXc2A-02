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
  ```
  Aplicar con:
  ```bash
  sudo netplan apply
  ```

### 2.2. Acceso Remoto desde Fuera de la Red (VPN IsardVDI)
Cuando el usuario se conecta desde su equipo anfitrión mediante la VPN de IsardVDI, el tráfico entra a través de la interfaz `enp2s0` de Proxmox (por ejemplo, `10.2.169.223`), donde el hipervisor redirige automáticamente los puertos web (`80` y `443`) hacia `WEB-NCC` (`10.10.10.20`).
* **URL de acceso por VPN:** `http://<IP_VPN_PROXMOX>` (ej. `[http://10.2.169.223](http://10.2.169.223)`) o `[http://educacion.local](http://educacion.local)` (si el DNS de la VPN apunta a Proxmox).

---

## 3. Guía de Uso del Portal Web (`[http://educacion.local](http://educacion.local)`)

### 3.1. Inicio de Sesión (Autenticación Centralizada LDAP)
1. Abrir el navegador e introducir **`[http://educacion.local](http://educacion.local)`**.
2. En la pantalla de **Autenticación Institucional**, introducir:
   * **Usuario (UID):** Nombre de usuario registrado en OpenLDAP (ej. `bchecker` o `admin`).
   * **Contraseña:** Contraseña asignada en el directorio.
3. Pulsar **Iniciar Sesión**.

> **Política de Seguridad de Contraseñas (`ppolicy`):** Por seguridad, el sistema permite un máximo de **3 intentos fallidos**. La interfaz informa de los intentos restantes en pantalla. Al tercer fallo consecutivo, la cuenta queda **bloqueada automáticamente durante 15 minutos** (o hasta que un administrador la desbloquee).

### 3.2. Consulta y Filtrado de Equipamientos Educativos
Una vez autenticado, el panel principal muestra el catálogo de **2.710 equipamientos educativos de Barcelona**:
* **Búsqueda por texto libre:** Escribir en el cuadro superior para filtrar en tiempo real por nombre del centro, calle o categoría.
* **Filtro por distrito:** Seleccionar uno de los 10 distritos municipales en el desplegable y pulsar **Filtrar**.
* **Limpiar filtros:** Pulsar el botón **`X`** junto al botón de filtrado para restaurar la vista general.

### 3.3. Gestión de Registros (Operaciones CRUD)
* **Añadir Equipamiento:** Pulsar el botón verde **`+ Añadir Equipamiento`**, rellenar los campos (*Reg. ID*, *Nombre*, *Calle*, *Número*, *Barrio*, *Distrito*, *Categoría*) y pulsar **Crear Equipamiento**.
* **Editar Equipamiento:** Pulsar el botón amarillo **`Editar`** en la fila del centro deseado, modificar los datos en la ventana modal y pulsar **Actualizar Cambios**.
* **Eliminar Equipamiento:** Pulsar el botón rojo **`Eliminar`** y confirmar la acción en el cuadro de diálogo.

### 3.4. Funciones Exclusivas para Administradores (Grupo `admins`)
Los usuarios pertenecientes al grupo LDAP `admins` (identificados con la insignia **`Admin`** junto a su nombre de usuario):
* Visualizan automáticamente un **banner superior de alerta** cuando existe alguna cuenta bloqueada por intentos fallidos.
* Pueden liberar cualquier cuenta con un solo clic pulsando el botón **`Desbloquear`** junto al UID bloqueado.
* Disponen de acceso a la consola de administración de directorio **LDAP Account Manager (LAM)** en `[http://10.10.10.30/lam](http://10.10.10.30/lam)` para crear nuevos usuarios o restablecer contraseñas.

---

## 4. Mantenimiento Básico del Puesto Cliente

* **Renovación de concesión DHCP y caché DNS:** Ejecutar `ipconfig /renew && ipconfig /flushdns` (Windows) o `sudo resolvectl flush-caches` (Linux) si se realizan cambios en el direccionamiento interno.
* **Cierre de sesión seguro:** Utilizar siempre el botón **`Cerrar Sesión`** en la esquina superior derecha al finalizar el trabajo en equipos compartidos.

---

## 5. Resolución de Problemas Frecuentes (Troubleshooting)

| Síntoma | Causa Probable | Solución Paso a Paso |
| :--- | :--- | :--- |
| **El navegador no resuelve `educacion.local`** | El cliente no está usando el DNS de Proxmox (`192.168.2.1`) o tiene caché antigua. | Ejecutar `nslookup educacion.local`. Si no responde `192.168.2.1`, renovar IP por DHCP (`ipconfig /renew`) y vaciar caché DNS (`ipconfig /flushdns`). |
| **La página se queda cargando indefinidamente** | Desajuste de MTU en el túnel VPN de IsardVDI o reglas de enrutamiento pendientes tras reiniciar el servidor. | Comprobar que la interfaz del cliente tiene `mtu 1446`. En acceso por VPN, acceder mediante la IP de `enp2s0` de Proxmox (`[http://10.2.169.223](http://10.2.169.223)`). |
| **Mensaje *"Cuenta bloqueada"* al iniciar sesión** | Se han superado los 3 intentos fallidos permitidos por la política `ppolicy` de OpenLDAP. | Esperar 15 minutos a que expire el bloqueo automático o solicitar a un usuario del grupo `admins` que pulse **Desbloquear** en el panel web. |
| **El portal carga pero se queda esperando al pulsar *Iniciar Sesión*** | El contenedor `LDAP-NCC` (`10.10.10.30`) o `B-NCC` (`10.10.20.10`) está detenido o falta la ruta Intranet (`10.10.20.1`). | Contactar con el administrador para verificar el estado de los contenedores (`pct list`) y la pasarela `vmbr1:0` (`10.10.20.1`) en Proxmox. |
