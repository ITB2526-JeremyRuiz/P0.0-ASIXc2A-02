# Análisis de Necesidades, Estudio de Mercado y Justificación Tecnológica

## 1. Planteamiento del Problema y Objetivos

### 1.1. Problema Detectado
La gestión y consulta de los equipamientos educativos de la ciudad de Barcelona requiere centralizar un catálogo público extenso (2.710 registros oficiales procedentes del portal OpenData Barcelona) en una plataforma interna segura, accesible tanto desde puestos locales como de forma remota mediante VPN, garantizando la integridad de los datos (codificación UTF-8) y el control de acceso centralizado.

### 1.2. Objetivos del Proyecto
* **Objetivo General:** Diseñar, desplegar y documentar una infraestructura multicapa segmentada (DMZ, Intranet y Red de Clientes/VPN) que soporte una aplicación web de gestión CRUD sobre los equipamientos educativos de Barcelona.
* **Objetivos Específicos:**
  1. Garantizar el aislamiento de servicios críticos (base de datos en Intranet frente a servicios expuestos en DMZ).
  2. Centralizar la autenticación y las políticas de seguridad de contraseñas mediante directorio LDAP.
  3. Automatizar la conectividad de clientes internos mediante servicios DHCP y DNS integrados.
  4. Asegurar la portabilidad y reproducibilidad de los servicios de datos y transferencia de ficheros mediante contenedores Docker.

### 1.3. Público Objetivo
* **Usuarios Finales (Personal Administrativo y Técnico):** Consultan, filtran, dan de alta y actualizan registros de centros educativos desde estaciones Windows y Linux.
* **Administradores de Sistemas (Equipo IT):** Gestionan la infraestructura virtualizada, políticas de bloqueo de cuentas, copias de seguridad y enrutamiento.

---

## 2. Estudio de Mercado y Comparativa de Tecnologías

Para seleccionar la pila tecnológica más adecuada al entorno académico y empresarial de recursos acotados (despliegue anidado sobre IsardVDI), se evaluaron las principales soluciones disponibles en el mercado:

### 2.1. Plataforma de Virtualización (Hipervisor)

| Criterio Técnico | Proxmox VE (Elegido) | VMware ESXi | VirtualBox / KVM puro |
| :--- | :--- | :--- | :--- |
| **Licenciamiento y Coste** | Software Libre (AGPLv3), sin coste de licencia para todas las funciones empresariales. | Propietario (modelo de suscripción tras adquisición por Broadcom; versión gratuita limitada). | Gratuito (GPL), pero sin gestión web centralizada nativa en KVM puro. |
| **Tecnología de Virtualización** | Híbrida: Máquinas Virtuales completas (**KVM**) + Contenedores ligeros (**LXC**). | Únicamente Máquinas Virtuales completas (mayor consumo de RAM/CPU por nodo). | Únicamente Máquinas Virtuales completas (VirtualBox). |
| **Gestión de Redes Virtuales** | Soporta múltiples puentes Linux (`vmbr`), VLANs, NAT e `iptables` directamente desde el host. | Switches virtuales avanzados (vSwitch), pero rígidos en entornos anidados sin vCenter. | Limitado en VirtualBox para escenarios complejos con múltiples subredes y MTU personalizado. |
| **Copias de Seguridad y Snapshots** | Integrados de serie (`vzdump`, snapshots instantáneos en LXC/KVM) sin software adicional. | Requiere soluciones de terceros (Veeam) mediante APIs de pago. | Snapshots básicos sin programación centralizada. |

**Justificación de la elección de Proxmox VE:**
Elegimos **Proxmox VE** porque permite desplegar cada servicio de red en su propio contenedor **LXC** (`WEB-NCC`, `LDAP-NCC`, `F-NCC`, `B-NCC`) consumiendo una fracción mínima de memoria RAM y CPU en comparación con máquinas virtuales completas. Además, su base Debian GNU/Linux nos permitió absorber directamente en el hipervisor (`pvexx`) las funciones de enrutamiento, cortafuegos (`iptables`) y servidor DNS/DHCP (`dnsmasq`), superando las restricciones de filtrado de direcciones MAC de la plataforma IsardVDI.

---

### 2.2. Contenerización y Despliegue de Servicios

| Criterio | Docker + Docker Compose (Elegido) | Instalación Nativa sobre SO | Kubernetes (K8s / K3s) |
| :--- | :--- | :--- | :--- |
| **Aislamiento de Dependencias** | Alto: empaqueta MySQL 8.0, PHP 8.2-FPM y Nginx Alpine sin conflictos de librerías. | Bajo: actualizaciones del sistema pueden romper versiones de PHP o conectores. | Muy alto, pero excesivo para un despliegue de nodo único. |
| **Reproducibilidad** | Total mediante ficheros declarativos (`docker-compose.yml` y `Dockerfile`). | Manual y propensa a errores humanos entre entornos. | Alta complejidad de manifiestos YAML y mayor consumo base. |

---

### 2.3. Directorio de Identidades y Base de Datos

* **OpenLDAP + LAM frente a Active Directory / FreeIPA:** Se seleccionó **OpenLDAP** por su ligereza en contenedores Linux, su compatibilidad nativa con el módulo PHP (`php-ldap`) y la posibilidad de aplicar políticas granulares de bloqueo contra fuerza bruta mediante el *overlay* `ppolicy`. Se complementó con **LDAP Account Manager (LAM)** para facilitar la administración visual.
* **MySQL 8.0 frente a PostgreSQL / MariaDB:** Se optó por **MySQL 8.0** por su soporte nativo del conjunto de caracteres `utf8mb4_unicode_ci` (indispensable para corregir la codificación UTF-16 LE con BOM del dataset de OpenData Barcelona) y su integración directa mediante sentencias preparadas (`mysqli`) en PHP 8.2.
