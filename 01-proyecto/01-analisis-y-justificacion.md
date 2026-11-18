# Análisis de Necesidades, Estudio de Mercado y Justificación Tecnológica

## 1. Planteamiento del Problema y Objetivos

### 1.1. Identificación de la Necesidad
Los departamentos de gestión educativa requieren consultar, filtrar y mantener actualizado el catálogo oficial de equipamientos educativos de la ciudad de Barcelona (**2.710 registros públicos** procedentes del portal *OpenData Ajuntament de Barcelona*). El fichero fuente original se distribuye en formato CSV codificado en **UTF-16 LE con marcas BOM (`\ufeff`)**, lo que provoca corrupción de caracteres (*mojibake*) al importarlo directamente en motores de bases de datos estándar.

Asimismo, la infraestructura debe garantizar:
1. **Segmentación estricta de red** para aislar el servidor de base de datos respecto a los servicios expuestos en la zona desmilitarizada.
2. **Autenticación centralizada** con protección activa frente a ataques de fuerza bruta y gestión delegada de bloqueos.
3. **Acceso transparente** tanto para puestos internos mediante asignación dinámica de red como para administradores conectados remotamente vía VPN sobre la plataforma **IsardVDI**.

### 1.2. Objetivos del Proyecto
* **Objetivo General:** Diseñar, desplegar, automatizar y documentar una arquitectura multicapa de alta disponibilidad sobre Proxmox VE que soporte una aplicación web segura de gestión CRUD de equipamientos educativos.
* **Objetivos Específicos:**
  * Implementar tres zonas de red diferenciadas (DMZ, Intranet y Red de Clientes/VPN) con enrutamiento, NAT/DNAT y resolución DNS local autoritativa (`educacion.local`).
  * Desplegar servicios portables mediante contenedores **Docker** y **Docker Compose** (`MySQL 8.0`, `PHP 8.2-FPM`, `Nginx Alpine`, `ProFTPD`).
  * Integrar un directorio **OpenLDAP** con el módulo de políticas `ppolicy` y gestión visual mediante **LDAP Account Manager (LAM)**.
  * Automatizar la ingesta limpia de datos desde Python asegurando codificación `utf8mb4` de extremo a extremo.

### 1.3. Público Objetivo
* **Personal Técnico y Administrativo (Usuarios Finales):** Consultan centros educativos por distrito, calle o categoría, y realizan altas, modificaciones o bajas desde estaciones Windows y Linux.
* **Administradores de Sistemas (Equipo IT):** Supervisan el hipervisor Proxmox, gestionan cuentas y desbloqueos en LDAP, administran copias de seguridad y auditan el tráfico de red.

---

## 2. Estudio de Mercado y Comparativa de Tecnologías

Para seleccionar la pila tecnológica óptima dentro de un entorno virtualizado anidado con recursos de CPU y memoria acotados (IsardVDI con encapsulación VXLAN), se realizó un análisis comparativo de las principales alternativas disponibles en el mercado:

### 2.1. Plataforma de Virtualización (Hipervisor)

| Criterio de Evaluación | Proxmox VE (Solución Elegida) | VMware ESXi (Broadcom) | VirtualBox / KVM puro |
| :--- | :--- | :--- | :--- |
| **Modelo de Licencia y Coste** | Software Libre (**AGPLv3**). Acceso completo a funcionalidades empresariales sin coste de licencia. | Propietario. Tras su adquisición por Broadcom, el licenciamiento por suscripción resulta costoso y la versión gratuita carece de APIs de respaldo. | Gratuito (GPL/PUEL), pero KVM puro carece de interfaz web centralizada nativa y VirtualBox no está orientado a servidores. |
| **Arquitectura de Virtualización** | **Híbrida (KVM + LXC):** Permite ejecutar máquinas virtuales completas y contenedores Linux ultracompactos sobre el mismo kernel. | **Monolítica (solo VMs):** Cada servicio requiere su propio sistema operativo completo y reserva dedicada de RAM. | **Solo VMs** en VirtualBox; gestión manual por CLI/libvirt en KVM puro. |
| **Flexibilidad de Red en IsardVDI** | Basado en Debian GNU/Linux: soporta puentes Linux (`vmbr`), ajuste fino de `MTU 1446`, `iptables` y servicios de red directamente en el host. | Switches virtuales (`vSwitch`) cerrados; requiere desplegar vCenter para funcionalidades avanzadas de red. | Limitado en VirtualBox para enrutamiento complejo inter-subred y control granular de `TCPMSS`. |
| **Copias de Seguridad y Snapshots** | Integrados de forma nativa (`vzdump`, snapshots instantáneos y programación desde la UI). | Requiere soluciones de terceros (como Veeam Backup) mediante licencias de pago. | Snapshots manuales básicos sin programación centralizada. |

**Conclusión y Justificación de Proxmox VE:**
La elección de **Proxmox VE (`pvexx`)** resulta determinante por dos factores técnicos:
1. **Eficiencia de recursos mediante LXC:** Al desplegar `WEB-NCC`, `F-NCC`, `LDAP-NCC` y `B-NCC` como contenedores LXC en lugar de máquinas virtuales completas, el consumo global de memoria RAM se reduce en más de un 65 %, manteniendo tiempos de arranque inferiores a 3 segundos.
2. **Resolución de restricciones de capa 2 en IsardVDI:** Gracias a su base Debian, Proxmox pudo asumir directamente las funciones de enrutamiento, cortafuegos (`iptables`) y servidor DNS/DHCP (`dnsmasq`), evitando el bloqueo *Anti-MAC Spoofing* que afectaba a un router virtualizado independiente (`R-NCC`).

---

### 2.2. Contenerización y Orquestación de Servicios

| Criterio | Docker + Docker Compose (Elegido) | Instalación Nativa en SO | Kubernetes (K8s / K3s) |
| :--- | :--- | :--- | :--- |
| **Aislamiento de Entorno** | Alto: encapsula `mysql:8.0`, `php:8.2-fpm`, `nginx:alpine` y `delfer/proftpd` con sus versiones exactas. | Bajo: conflictos potenciales entre extensiones PHP, librerías OpenSSL y paquetes del sistema. | Muy alto, diseñado para clústeres multi-nodo de gran escala. |
| **Automatización e Idempotencia** | Completa mediante ficheros declarativos (`docker-compose.yml` y `Dockerfile`) integrados en `deploy.sh`. | Requiere scripts imperativos complejos y difíciles de revertir. | Sobredimensionado para una infraestructura de nodo único en laboratorio/pyme. |

---

### 2.3. Directorio de Identidades y Motor de Base de Datos

* **OpenLDAP + LAM frente a Active Directory / FreeIPA:** **OpenLDAP** ofrece un consumo mínimo de memoria (inferior a 50 MB), integración directa con PHP mediante `php-ldap` y soporte nativo del *overlay* `ppolicy` para el bloqueo automático de cuentas por intentos fallidos. La incorporación de **LDAP Account Manager (LAM)** proporciona una interfaz web intuitiva para la administración de usuarios y grupos POSIX.
* **MySQL 8.0 frente a PostgreSQL / MariaDB:** **MySQL 8.0** garantiza compatibilidad nativa con `utf8mb4_unicode_ci` para almacenar caracteres catalanes y castellanos sin pérdida, integración nativa con `mysqli` (consultas preparadas) e inicialización automática de esquemas mediante `/docker-entrypoint-initdb.d/`.
