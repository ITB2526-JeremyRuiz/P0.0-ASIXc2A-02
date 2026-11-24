# Configuración del Servicio de Administración Remota SSH

Este documento especifica la arquitectura, el aprovisionamiento automatizado, el endurecimiento de seguridad (`hardening`) y la configuración de autenticación por clave pública del servicio **OpenSSH (`sshd`)** desplegado de forma transversal en los nodos Linux de la infraestructura (`F-NCC`, `WEB-NCC`, `LDAP-NCC`, `B-NCC` y `CLI-LINUX`).

---

## 1. Arquitectura y Alcance del Servicio SSH

Para garantizar una administración remota segura y auditada sin exponer cuentas privilegiadas (`root`), todos los contenedores Linux ejecutan el demonio `openssh-server` bajo un perfil unificado vinculado al usuario estándar de administración **`bchecker`**:

| Nodo / Hostname | Rol en la Infraestructura | Dirección IP | Puerto SSH | Métodos de Autenticación Habilitados |
| :--- | :--- | :--- | :--- | :--- |
| **`F-NCC` (`Servidor-FTP`)** | Servidor FTP pasivo (DMZ) | `10.10.10.10/24` | TCP `22` | Clave pública Ed25519 + Contraseña (`bchecker121`) |
| **`WEB-NCC` (`W-NCC`)** | Servidor Web CRUD (DMZ) | `10.10.10.20/24` | TCP `22` (DNAT VPN `2220`) | Clave pública Ed25519 + Contraseña (`bchecker121`) |
| **`LDAP-NCC`** | Directorio OpenLDAP y LAM (DMZ) | `10.10.10.30/24` | TCP `22` | Clave pública Ed25519 + Contraseña (`bchecker121`) |
| **`B-NCC`** | Base de Datos MySQL 8.0 (Intranet) | `10.10.20.10/24` | TCP `22` | Clave pública Ed25519 + Contraseña (`bchecker121`) |
| **`CLI-LINUX`** | Estación cliente y consola de gestión | `192.168.2.50` – `.150` | TCP `22` | Cliente SSH (par de claves Ed25519) + Servidor SSH |

---

## 2. Aprovisionamiento Automatizado de `openssh-server` y Usuario `bchecker`

En cada contenedor LXC se ejecutó el siguiente script idempotente de aprovisionamiento, encargado de instalar `openssh-server` y `sudo`, verificar la existencia previa de `bchecker` antes de crearlo y aplicar la directiva de seguridad de `sshd`:

```bash
#!/bin/bash
set -e

# 1. Instalar servidor SSH y sudo
apt-get update && apt-get install -y openssh-server sudo
systemctl enable --now ssh

# 2. Comprobar si existe el usuario 'bchecker' y crearlo si no está
if ! id -u bchecker >/dev/null 2>&1; then
    useradd -m -s /bin/bash bchecker
    echo "bchecker:bchecker121" | chpasswd
    echo "[OK] Usuario 'bchecker' creado."
else
    echo "[INFO] El usuario 'bchecker' ya existe, no se vuelve a crear."
fi

# 3. Garantizar permisos de administrador (grupo sudo)
usermod -aG sudo bchecker

# 4. Aplicar configuración de seguridad SSH y reiniciar
mkdir -p /etc/ssh/sshd_config.d
cat << 'EOF' > /etc/ssh/sshd_config.d/99-proyecto-p0.conf
Port 22
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
MaxAuthTries 3
UseDNS no
AllowUsers bchecker
EOF

sshd -t && systemctl restart ssh
echo "[OK] Servidor SSH operativo y configurado."
```

---

## 3. Endurecimiento de Seguridad (`/etc/ssh/sshd_config.d/99-proyecto-p0.conf`)

La configuración modular en `/etc/ssh/sshd_config.d/99-proyecto-p0.conf` sobrescribe los valores por defecto de las plantillas LXC de Ubuntu aplicando los siguientes controles:

* **`PermitRootLogin no`:** Bloquea cualquier intento de inicio de sesión directo como superusuario `root`, obligando a autenticarse con una cuenta nominal (`bchecker`) y escalar privilegios mediante `sudo` para mantener la trazabilidad.
* **`AllowUsers bchecker`:** Restringe el acceso SSH exclusivamente al usuario autorizado del proyecto, rechazando cualquier otro identificador del sistema.
* **`MaxAuthTries 3`:** Limita a tres los intentos de autenticación por conexión para mitigar ataques de fuerza bruta automatizados.
* **`PubkeyAuthentication yes` / `PasswordAuthentication yes`:** Habilita la autenticación asimétrica mediante claves criptográficas sin contraseña para administración diaria desde `CLI-LINUX`, manteniendo activa la contraseña `bchecker121` para tareas de evaluación y auditoría.
* **`UseDNS no`:** Desactiva la resolución DNS inversa de las IP cliente durante el *handshake* SSH, eliminando retardos en el establecimiento de sesión.

---

## 4. Configuración de Autenticación por Clave Pública desde `CLI-LINUX`

Desde la estación de trabajo **`CLI-LINUX`** se generó un par de claves criptográficas **Ed25519** y se distribuyó la clave pública (`~/.ssh/id_ed25519.pub`) a todos los servidores internos:

### 4.1. Generación y Distribución de Claves
```bash
# Generar par de claves Ed25519 en CLI-LINUX
ssh-keygen -t ed25519 -C "admin@educacion.local"

# Copiar clave pública a los cuatro servidores internos
ssh-copy-id bchecker@10.10.10.10   # F-NCC
ssh-copy-id bchecker@10.10.10.20   # WEB-NCC
ssh-copy-id bchecker@10.10.10.30   # LDAP-NCC
ssh-copy-id bchecker@10.10.20.10   # B-NCC
```

### 4.2. Configuración de Alias de Acceso Rápido (`~/.ssh/config`)
```ssh-config
Host f-ncc
    HostName 10.10.10.10
    User bchecker
    IdentityFile ~/.ssh/id_ed25519

Host web-ncc
    HostName 10.10.10.20
    User bchecker
    IdentityFile ~/.ssh/id_ed25519

Host ldap-ncc
    HostName 10.10.10.30
    User bchecker
    IdentityFile ~/.ssh/id_ed25519

Host b-ncc
    HostName 10.10.20.10
    User bchecker
    IdentityFile ~/.ssh/id_ed25519
```

---

## 5. Acceso Remoto desde VPN vía DNAT en Proxmox (`pvexx`)

Dado que el hipervisor Proxmox (`pvexx`) utiliza el puerto `22` en su interfaz VPN (`enp2s0` — `10.2.169.223`), el acceso administrativo directo hacia el servidor web interno (`WEB-NCC` — `10.10.10.20`) desde fuera de la VPN se canaliza mediante redirección de puerto (`DNAT` TCP `2220` $\rightarrow$ `10.10.10.20:22`) en `/etc/network/interfaces`:

```bash
post-up iptables -t nat -A PREROUTING -i enp2s0 -p tcp --dport 2220 -j DNAT --to-destination 10.10.10.20:22
post-down iptables -t nat -D PREROUTING -i enp2s0 -p tcp --dport 2220 -j DNAT --to-destination 10.10.10.20:22
```

Comando de conexión remota desde la red VPN:
```bash
ssh -p 2220 bchecker@10.2.169.223
```

---

## 6. Pruebas de Verificación y Auditoría

```bash
# 1. Comprobar en cada servidor que sshd escucha en el puerto 22 y validar directivas activas
ss -tlnp | grep :22
sshd -T | grep -E "permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers"

# 2. Verificar acceso sin contraseña (BatchMode) desde CLI-LINUX a toda la infraestructura
for host in 10.10.10.10 10.10.10.20 10.10.10.30 10.10.20.10; do
    ssh -o BatchMode=yes bchecker@$host "echo -n '$host -> '; hostname"
done

# 3. Verificar rechazo de conexión directa como root (debe devolver Permission denied)
ssh -o PubkeyAuthentication=no root@10.10.10.20
```
