# Infraestructura de Autenticación Centralizada OpenLDAP, LAM y Aplicación Web (`educacion.local`)

En este documento recogemos la arquitectura, el despliegue y la puesta en marcha de nuestro servidor de directorio **OpenLDAP (`LDAP-NCC`)**, la interfaz de gestión web **LDAP Account Manager (LAM)**, la política de bloqueo automático de cuentas (**`ppolicy`**) y su integración en el portal web **Educación Barcelona (`WEB-NCC`)**.

---

## 1. Arquitectura y Topología del Sistema

Hemos diseñado una arquitectura distribuida en tres nodos independientes dentro de nuestra red interna para separar los servicios de directorio, aplicación web y persistencia de datos:

| Componente | Hostname / URL | Dirección IP / Puerto | Función Principal |
| :--- | :--- | :--- | :--- |
| **Servidor Directorio** | `LDAP-NCC` | `10.10.10.30` (TCP `389`) | Contenedor con OpenLDAP (`dc=educacion,dc=local`), módulo `ppolicy` y LAM (`/lam`). |
| **Servidor Web** | `WEB-NCC` (`https://educacion.local`) | `HTTPS` (`443`) | Portal PHP-FPM + servidor web con autenticación LDAP, control de bloqueos y gestión CRUD. |
| **Base de Datos** | `DB-NCC` | `10.10.20.10` (TCP `3306`) | Base de datos MySQL `bcn_educacio` (tabla `equipaments`). |

### Estructura del Árbol LDAP (`dc=educacion,dc=local`)

Organizamos nuestro directorio bajo el sufijo `dc=educacion,dc=local` dividiendo las cuentas, los grupos y las políticas de seguridad en tres unidades organizativas dedicadas:

```text
dc=educacion,dc=local
├── cn=admin                       (Administrador global del directorio)
├── ou=usuarios                    (Cuentas de usuario: posixAccount + inetOrgPerson + shadowAccount)
│   ├── uid=admin                  (Administrador web - GID primario 10000 / memberUid en admins)
│   └── uid=bchecker               (Usuario estándar de aplicación)
├── ou=grupos                      (Grupos POSIX)
│   └── cn=admins                  (gidNumber: 10000 - Otorga permisos de desbloqueo en WEB-NCC)
└── ou=policies                    (Políticas de contraseñas ppolicy)
    └── cn=default                 (3 intentos fallidos, 15 minutos / 900s de bloqueo)
```

---

## 2. Despliegue de OpenLDAP en `LDAP-NCC` (`10.10.10.30`)

### 2.1. Instalación de paquetes base

Partiendo de un contenedor limpio en `10.10.10.30`, instalamos el demonio `slapd` junto con las utilidades cliente de LDAP y lanzamos la configuración inicial del dominio:

```bash
apt update && apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```

Durante el asistente establecimos los siguientes parámetros:
* **Dominio DNS:** `educacion.local` (`dc=educacion,dc=local`)
* **Organización:** `Educacion Barcelona`
* **Motor de base de datos:** `MDB`
* **Eliminar base de datos al purgar:** `No`

### 2.2. Creación de Unidades Organizativas (`ou=usuarios`, `ou=grupos`, `ou=policies`)

A continuación, definimos el archivo `estructura_base.ldif` para crear las ramas donde alojaremos usuarios, grupos POSIX y políticas de contraseñas:

```ldif
dn: ou=usuarios,dc=educacion,dc=local
objectClass: organizationalUnit
ou: usuarios

dn: ou=grupos,dc=educacion,dc=local
objectClass: organizationalUnit
ou: grupos

dn: ou=policies,dc=educacion,dc=local
objectClass: organizationalUnit
ou: policies
```

Incorporamos estas entradas al directorio autenticándonos con el usuario administrador global:

```bash
ldapadd -x -D "cn=admin,dc=educacion,dc=local" -W -f estructura_base.ldif
```

---

## 3. Configuración de Política de Contraseñas y Bloqueo (`ppolicy`)

Con el objetivo de proteger el portal frente a ataques de fuerza bruta, activamos el *overlay* `ppolicy` en OpenLDAP, configurando el bloqueo automático tras **3 intentos fallidos** durante **15 minutos (900 segundos)**.

### 3.1. Carga del esquema y módulo `ppolicy`

Cargamos el esquema correspondiente en la configuración dinámica (`cn=config`) y habilitamos el módulo `ppolicy.la`:

```bash
# Cargamos el esquema ppolicy (si no está ya integrado en slapd)
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/ppolicy.ldif 2>/dev/null || true

# Activamos el módulo ppolicy.la
ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: ppolicy.la
EOF
```

### 3.2. Creación de la política por defecto (`cn=default,ou=policies,dc=educacion,dc=local`)

Creamos el objeto de política predeterminada que aplicará las reglas de intentos y tiempos de bloqueo sobre nuestras cuentas:

```bash
ldapadd -x -D "cn=admin,dc=educacion,dc=local" -W << 'EOF'
dn: cn=default,ou=policies,dc=educacion,dc=local
objectClass: top
objectClass: device
objectClass: pwdPolicy
cn: default
pwdAttribute: userPassword
pwdMaxFailure: 3
pwdFailureCountInterval: 300
pwdLockout: TRUE
pwdLockoutDuration: 900
pwdMustChange: FALSE
pwdAllowUserChange: TRUE
pwdSafeModify: FALSE
EOF
```

### 3.3. Vinculación del overlay `ppolicy` a la base de datos MDB

Finalmente, asociamos el *overlay* `ppolicy` a nuestra base de datos `olcDatabase={1}mdb,cn=config` para que OpenLDAP aplique la política `cn=default` de forma global:

```bash
ldapadd -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: olcOverlay=ppolicy,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcPPolicyConfig
olcOverlay: ppolicy
olcPPolicyDefault: cn=default,ou=policies,dc=educacion,dc=local
olcPPolicyHashCleartext: TRUE
olcPPolicyUseLockout: TRUE
EOF
```

---

## 4. Instalación y Configuración de LDAP Account Manager (LAM)

Para evitar depender exclusivamente de comandos LDIF en las tareas diarias de administración, desplegamos **LDAP Account Manager (LAM)** en el propio servidor `LDAP-NCC` (`http://10.10.10.30/lam`).

### 4.1. Instalación

Instalamos Apache, PHP con soporte LDAP y el paquete de LAM:

```bash
apt install -y apache2 php php-ldap php-xml php-mbstring php-zip php-gd ldap-account-manager
systemctl enable --now apache2
```

### 4.2. Configuración del Perfil del Servidor en LAM

Accedimos al panel de configuración de LAM (`http://10.10.10.30/lam` → **LAM configuration** → **Edit server profiles**) y adaptamos el perfil a nuestra estructura de directorio:

1. **General settings:**
   * **Server address:** `ldap://localhost:389`
   * **Tree suffix:** `dc=educacion,dc=local`
   * **Security settings → List of valid users:** `cn=admin,dc=educacion,dc=local`
2. **Account types:**
   * **Users → LDAP suffix:** `ou=usuarios,dc=educacion,dc=local`
   * **Groups → LDAP suffix:** `ou=grupos,dc=educacion,dc=local`
3. **Modules:**
   * **Users:** `Personal (inetOrgPerson)`, `Unix (posixAccount)`, `Shadow (shadowAccount)`.
   * **Groups:** `Unix (posixGroup)`.

### 4.3. Creación de Grupos y Usuarios

A través de la interfaz de LAM dimos de alta los grupos y usuarios necesarios para operar la plataforma:

* **Grupo `admins`:**
  * `cn`: `admins`
  * `gidNumber`: `10000`
  * `memberUid`: `admin`
* **Usuario Administrador (`admin`):**
  * `uid`: `admin`
  * `uidNumber`: `10002`
  * `gidNumber`: `10000` (Grupo primario `admins`)
  * `homeDirectory`: `/home/admin`
  * `loginShell`: `/bin/bash`
* **Usuario Estándar (`bchecker`):**
  * `uid`: `bchecker`
  * `uidNumber`: `10001`
  * `gidNumber`: `10001`

*(Nota: Dado que la versión gratuita de LAM permite inspeccionar los atributos operativos `pwdAccountLockedTime` y `pwdFailureTime` desde **Tools → Tree view → Show internal attributes** en modo lectura, implementamos el desbloqueo interactivo en un clic directamente en nuestro portal web `WEB-NCC`).*

---

## 5. Integración en el Portal Web `WEB-NCC` (`https://educacion.local`)

### 5.1. Requisitos en el servidor web

En el servidor `WEB-NCC` habilitamos las extensiones `php-ldap` y `php-mysql` para comunicar la aplicación tanto con OpenLDAP como con MySQL:

```bash
apt install -y php-ldap php-mysql
systemctl restart php8.2-fpm nginx   # Ajustar versión de PHP-FPM si aplica
```

### 5.2. Funcionalidades implementadas en `/var/www/html/index.php`

Desarrollamos el archivo `/var/www/html/index.php` integrando las siguientes capas de seguridad y gestión:

1. **Autenticación LDAP con patrón PRG (Post/Redirect/Get):** Evitamos el reenvío accidental del formulario al recargar el navegador y regeneramos el identificador de sesión (`session_regenerate_id(true)`) tras cada inicio de sesión válido.
2. **Diagnóstico inteligente de `ppolicy`:** Cuando `ldap_bind()` rechaza las credenciales del usuario, abrimos una conexión limpia independiente (`$diag_conn`) autenticada como `cn=admin` para consultar los atributos operativos del usuario:
   * Si existe `pwdAccountLockedTime`, informamos al usuario de que su **cuenta está bloqueada** durante 15 minutos o hasta que un administrador la libere.
   * Si existen marcas en `pwdFailureTime`, calculamos y mostramos en pantalla los **intentos restantes** (`2 intentos` / `¡Último intento!`).
3. **Panel de desbloqueo en 1 clic para administradores:**
   * Verificamos si el usuario autenticado forma parte del grupo `admins`, comprobando tanto su presencia en `memberUid` dentro de `ou=grupos` como su `gidNumber=10000` en `ou=usuarios`.
   * Mostramos la insignia **`Admin`** junto a su nombre de sesión.
   * Cuando detectamos cuentas con `pwdAccountLockedTime=*`, desplegamos un banner superior con el listado de usuarios bloqueados y un botón **Desbloquear** que elimina `pwdAccountLockedTime` y `pwdFailureTime` mediante `ldap_mod_del()`.
4. **Gestión CRUD de Equipamientos (`bcn_educacio.equipaments`):** Permitimos buscar, filtrar por distrito, crear, editar y eliminar equipamientos educativos utilizando sentencias preparadas (`mysqli::prepare`).

### 5.3. Código completo de `/var/www/html/index.php`

```php
<?php
session_start();
ini_set('display_errors', 1);
error_reporting(E_ALL);
header('Content-Type: text/html; charset=utf-8');

// ==========================================
// CONFIGURACIÓN LDAP (ppolicy y desbloqueo admin)
// ==========================================
define('LDAP_URI',          'ldap://10.10.10.30:389');
define('LDAP_USERS_OU',     'ou=usuarios,dc=educacion,dc=local');
define('LDAP_GROUPS_OU',    'ou=grupos,dc=educacion,dc=local');
define('LDAP_READ_DN',      'cn=admin,dc=educacion,dc=local');
define('LDAP_READ_PASS',    'Admin1234'); // Sustituir por la contraseña real de cn=admin
define('LDAP_MAX_FAILURES', 3);           // Debe coincidir con pwdMaxFailure en ppolicy
define('LDAP_ADMINS_GID',   10000);       // GID del grupo primario admins

// ==========================================
// 1. GESTIÓN DE SESIÓN Y AUTENTICACIÓN LDAP (PRG + ppolicy)
// ==========================================
if (isset($_GET['logout'])) {
    session_unset();
    session_destroy();
    header("Location: index.php");
    exit;
}

if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['action']) && $_POST['action'] === 'ldap_login') {
    $ldap_user = trim($_POST['auth_uid_field'] ?? '');
    $ldap_pass = $_POST['auth_pwd_field'] ?? '';

    if ($ldap_user !== '' && $ldap_pass !== '') {
        if (!function_exists('ldap_connect')) {
            $_SESSION['ldap_error'] = ['type' => 'danger', 'msg' => "Error: la extensión php-ldap no está activa en PHP-FPM."];
            header("Location: index.php");
            exit;
        }

        $safe_user = ldap_escape($ldap_user, "", LDAP_ESCAPE_DN);
        $user_dn   = "uid={$safe_user}," . LDAP_USERS_OU;

        $ldap_conn = @ldap_connect(LDAP_URI);
        if ($ldap_conn) {
            ldap_set_option($ldap_conn, LDAP_OPT_PROTOCOL_VERSION, 3);
            ldap_set_option($ldap_conn, LDAP_OPT_REFERRALS, 0);
            ldap_set_option($ldap_conn, LDAP_OPT_NETWORK_TIMEOUT, 5);

            // Intento de autenticación del usuario
            if (@ldap_bind($ldap_conn, $user_dn, $ldap_pass)) {
                session_regenerate_id(true);
                $_SESSION['ldap_user'] = $ldap_user;
                unset($_SESSION['ldap_error']);
                @ldap_unbind($ldap_conn);
                header("Location: index.php");
                exit;
            } else {
                // El bind falló: abrimos una conexión limpia independiente para consultar estado ppolicy
                $mensaje_error = "Usuario o contraseña incorrectos.";
                $tipo_alerta   = "danger";

                $diag_conn = @ldap_connect(LDAP_URI);
                if ($diag_conn) {
                    ldap_set_option($diag_conn, LDAP_OPT_PROTOCOL_VERSION, 3);
                    ldap_set_option($diag_conn, LDAP_OPT_REFERRALS, 0);
                    ldap_set_option($diag_conn, LDAP_OPT_NETWORK_TIMEOUT, 5);

                    if (@ldap_bind($diag_conn, LDAP_READ_DN, LDAP_READ_PASS)) {
                        $sr = @ldap_read($diag_conn, $user_dn, "(objectClass=*)", ["pwdAccountLockedTime", "pwdFailureTime", "uid"]);
                        if ($sr) {
                            $entries = ldap_get_entries($diag_conn, $sr);
                            if ($entries['count'] > 0) {
                                $info = $entries[0];

                                // 1. Comprobamos si la cuenta ya tiene marca de bloqueo
                                if (isset($info['pwdaccountlockedtime']) && $info['pwdaccountlockedtime']['count'] > 0) {
                                    $mensaje_error = "<strong>Cuenta bloqueada:</strong> Has superado el límite de intentos fallidos. Espera 15 minutos o contacta con un administrador para desbloquearla.";
                                    $tipo_alerta   = "dark";
                                } else {
                                    // 2. Contamos fallos recientes en la ventana de tiempo activa
                                    $fallos_actuales = isset($info['pwdfailuretime']) ? intval($info['pwdfailuretime']['count']) : 0;
                                    $restantes       = max(0, LDAP_MAX_FAILURES - $fallos_actuales);

                                    if ($restantes === 0) {
                                        $mensaje_error = "<strong>Cuenta bloqueada:</strong> Has agotado todos los intentos permitidos. La cuenta ha quedado bloqueada temporalmente.";
                                        $tipo_alerta   = "dark";
                                    } elseif ($restantes === 1) {
                                        $mensaje_error = "Contraseña incorrecta. <strong>¡Último intento!</strong> Si vuelves a fallar, tu cuenta quedará bloqueada.";
                                        $tipo_alerta   = "warning";
                                    } else {
                                        $mensaje_error = "Contraseña incorrecta. Te quedan <strong>{$restantes} intentos</strong> antes de que se bloquee tu cuenta.";
                                        $tipo_alerta   = "danger";
                                    }
                                }
                            } else {
                                $mensaje_error = "El usuario <strong>" . htmlspecialchars($ldap_user, ENT_QUOTES, 'UTF-8') . "</strong> no existe en el directorio.";
                            }
                        } else {
                            $mensaje_error = "El usuario <strong>" . htmlspecialchars($ldap_user, ENT_QUOTES, 'UTF-8') . "</strong> no existe en el directorio.";
                        }
                    } else {
                        $mensaje_error = "Error interno de diagnóstico LDAP: " . htmlspecialchars(ldap_error($diag_conn), ENT_QUOTES, 'UTF-8');
                    }
                    @ldap_unbind($diag_conn);
                }

                $_SESSION['ldap_error'] = ['type' => $tipo_alerta, 'msg' => $mensaje_error];
                @ldap_unbind($ldap_conn);
                header("Location: index.php");
                exit;
            }
        } else {
            $_SESSION['ldap_error'] = ['type' => 'danger', 'msg' => "No se pudo conectar con el servidor de autenticación (" . LDAP_URI . ")."];
            header("Location: index.php");
            exit;
        }
    } else {
        $_SESSION['ldap_error'] = ['type' => 'warning', 'msg' => "Por favor, introduce tu usuario y contraseña."];
        header("Location: index.php");
        exit;
    }
}

$ldap_error     = $_SESSION['ldap_error'] ?? null;
unset($_SESSION['ldap_error']);
$usuario_actual = $_SESSION['ldap_user'] ?? null;

// ==========================================
// 2. SI NO ESTÁ AUTENTICADO -> PANTALLA DE LOGIN
// ==========================================
if (!$usuario_actual):
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Acceso Restringido - Educación Barcelona</title>
    <link href="[https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css](https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css)" rel="stylesheet">
    <link href="[https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css](https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css)" rel="stylesheet">
</head>
<body class="bg-light d-flex align-items-center justify-content-center vh-100">
    <div class="card shadow-sm border-0" style="max-width: 440px; width: 100%;">
        <div class="card-header bg-primary text-white text-center py-3">
            <h4 class="mb-0"><i class="bi bi-shield-lock-fill"></i> Educación Barcelona</h4>
            <small class="opacity-75">Autenticación Institucional (educacion.local)</small>
        </div>
        <div class="card-body p-4">
            <?php if ($ldap_error !== null): ?>
                <div class="alert alert-<?= htmlspecialchars($ldap_error['type'], ENT_QUOTES, 'UTF-8') ?> py-2 small" role="alert">
                    <?php if ($ldap_error['type'] === 'dark'): ?>
                        <i class="bi bi-lock-fill"></i>
                    <?php else: ?>
                        <i class="bi bi-exclamation-triangle-fill"></i>
                    <?php endif; ?>
                    <?= $ldap_error['msg'] ?>
                </div>
            <?php endif; ?>

            <form method="POST" action="index.php" autocomplete="off">
                <input type="hidden" name="action" value="ldap_login">
                <div class="mb-3">
                    <label class="form-label fw-semibold">Usuario (UID)</label>
                    <div class="input-group">
                        <span class="input-group-text"><i class="bi bi-person-fill"></i></span>
                        <input type="text" name="auth_uid_field" class="form-control" required placeholder="Ej: bchecker" autocomplete="off" autofocus>
                    </div>
                </div>
                <div class="mb-4">
                    <label class="form-label fw-semibold">Contraseña</label>
                    <div class="input-group">
                        <span class="input-group-text"><i class="bi bi-key-fill"></i></span>
                        <input type="password" name="auth_pwd_field" id="ldap_pass" class="form-control" required placeholder="••••••••" autocomplete="new-password">
                        <button class="btn btn-outline-secondary" type="button" onclick="const p = document.getElementById('ldap_pass'); p.type = p.type === 'password' ? 'text' : 'password';" title="Mostrar/ocultar contraseña">
                            <i class="bi bi-eye"></i>
                        </button>
                    </div>
                </div>
                <button type="submit" class="btn btn-primary w-100 py-2 fw-semibold">
                    <i class="bi bi-box-arrow-in-right"></i> Iniciar Sesión
                </button>
            </form>
        </div>
        <div class="card-footer text-center text-muted small py-2 bg-white">
            Acceso exclusivo para personal autorizado
        </div>
    </div>
</body>
</html>
<?php
exit;
endif;

// ==========================================
// 3. CONTROL DE BLOQUEOS LDAP (SOLO GRUPO ADMINS)
// ==========================================
$es_admin_ldap = false;
$usuarios_bloqueados = [];

$lconn_adm = @ldap_connect(LDAP_URI);
if ($lconn_adm) {
    ldap_set_option($lconn_adm, LDAP_OPT_PROTOCOL_VERSION, 3);
    ldap_set_option($lconn_adm, LDAP_OPT_REFERRALS, 0);
    ldap_set_option($lconn_adm, LDAP_OPT_NETWORK_TIMEOUT, 5);

    if (@ldap_bind($lconn_adm, LDAP_READ_DN, LDAP_READ_PASS)) {
        // Comprobamos si el usuario pertenece al grupo admins (por memberUid o por gidNumber primario)
        $safe_u = ldap_escape($usuario_actual, "", LDAP_ESCAPE_FILTER);
        $sr_grp = @ldap_search($lconn_adm, LDAP_GROUPS_OU, "(&(cn=admins)(memberUid={$safe_u}))", ["cn"]);
        $sr_gid = @ldap_search($lconn_adm, LDAP_USERS_OU, "(&(uid={$safe_u})(gidNumber=" . LDAP_ADMINS_GID . "))", ["uid"]);

        if (($sr_grp && ldap_count_entries($lconn_adm, $sr_grp) > 0) || ($sr_gid && ldap_count_entries($lconn_adm, $sr_gid) > 0)) {
            $es_admin_ldap = true;
        }

        // Acción: desbloquear usuario con un clic
        if ($es_admin_ldap && isset($_GET['unlock_uid'])) {
            $uid_unlock = trim($_GET['unlock_uid']);
            $dn_unlock  = "uid=" . ldap_escape($uid_unlock, "", LDAP_ESCAPE_DN) . "," . LDAP_USERS_OU;
            @ldap_mod_del($lconn_adm, $dn_unlock, ['pwdAccountLockedTime' => []]);
            @ldap_mod_del($lconn_adm, $dn_unlock, ['pwdFailureTime' => []]);
            @ldap_unbind($lconn_adm);
            header("Location: index.php?unlocked=" . urlencode($uid_unlock));
            exit;
        }

        // Listamos todos los usuarios actualmente bloqueados
        if ($es_admin_ldap) {
            $sr_lock = @ldap_search($lconn_adm, LDAP_USERS_OU, "(pwdAccountLockedTime=*)", ["uid", "cn", "pwdAccountLockedTime"]);
            if ($sr_lock) {
                $entries_lock = ldap_get_entries($lconn_adm, $sr_lock);
                for ($i = 0; $i < $entries_lock['count']; $i++) {
                    $usuarios_bloqueados[] = [
                        'uid'  => $entries_lock[$i]['uid'][0] ?? '',
                        'cn'   => $entries_lock[$i]['cn'][0] ?? '',
                        'time' => $entries_lock[$i]['pwdaccountlockedtime'][0] ?? ''
                    ];
                }
            }
        }
    }
    @ldap_unbind($lconn_adm);
}

// ==========================================
// 4. USUARIO AUTENTICADO -> CONEXIÓN Y CRUD MYSQL
// ==========================================
$mysqli = new mysqli("10.10.20.10", "bchecker", "bchecker121", "bcn_educacio");

if ($mysqli->connect_error) {
    die("Error de conexión: " . htmlspecialchars($mysqli->connect_error, ENT_QUOTES, 'UTF-8'));
}

$mysqli->set_charset("utf8mb4");

function normalizar_texto($texto) {
    if ($texto === null || $texto === '') {
        return '';
    }
    if (!mb_check_encoding($texto, 'UTF-8')) {
        $texto = mb_convert_encoding($texto, 'UTF-8', 'UTF-16LE');
    }
    $texto = preg_replace('/\x{FEFF}/u', '', $texto);
    return trim($texto);
}

// ACCIÓN: ELIMINAR REGISTRO
if (isset($_GET['action']) && $_GET['action'] === 'delete' && isset($_GET['id'])) {
    $id_del = intval($_GET['id']);
    $stmt = $mysqli->prepare("DELETE FROM equipaments WHERE id = ?");
    $stmt->bind_param("i", $id_del);
    $stmt->execute();
    $stmt->close();

    header("Location: index.php");
    exit;
}

// ACCIÓN: CREAR O EDITAR REGISTRO (POST)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $id             = !empty($_POST['id']) ? intval($_POST['id']) : null;
    $register_id    = normalizar_texto($_POST['register_id'] ?? '');
    $name           = normalizar_texto($_POST['name'] ?? '');
    $address_road   = normalizar_texto($_POST['address_road'] ?? '');
    $address_number = normalizar_texto($_POST['address_number'] ?? '');
    $neighborhood   = normalizar_texto($_POST['neighborhood'] ?? '');
    $district       = normalizar_texto($_POST['district'] ?? '');
    $category       = normalizar_texto($_POST['category'] ?? '');

    if ($id) {
        $stmt = $mysqli->prepare("UPDATE equipaments SET register_id=?, name=?, address_road=?, address_number=?, neighborhood=?, district=?, category=? WHERE id=?");
        $stmt->bind_param("sssssssi", $register_id, $name, $address_road, $address_number, $neighborhood, $district, $category, $id);
    } else {
        $stmt = $mysqli->prepare("INSERT INTO equipaments (register_id, name, address_road, address_number, neighborhood, district, category) VALUES (?, ?, ?, ?, ?, ?, ?)");
        $stmt->bind_param("sssssss", $register_id, $name, $address_road, $address_number, $neighborhood, $district, $category);
    }

    $stmt->execute();
    $stmt->close();

    header("Location: index.php");
    exit;
}

// CONSULTA Y FILTRADO
$busqueda = isset($_GET['q']) ? normalizar_texto($_GET['q']) : '';
$distrito = isset($_GET['distrito']) ? normalizar_texto($_GET['distrito']) : '';

$distritos_query = $mysqli->query("SELECT DISTINCT district FROM equipaments WHERE district IS NOT NULL AND district != '' ORDER BY district ASC");

$sql = "SELECT id, register_id, name, address_road, address_number, neighborhood, district, category FROM equipaments WHERE 1=1";
$params = [];
$types = "";

if ($busqueda !== '') {
    $sql .= " AND (name LIKE ? OR address_road LIKE ? OR category LIKE ?)";
    $like_busqueda = "%" . $busqueda . "%";
    $params[] = $like_busqueda;
    $params[] = $like_busqueda;
    $params[] = $like_busqueda;
    $types .= "sss";
}

if ($distrito !== '') {
    $sql .= " AND district = ?";
    $params[] = $distrito;
    $types .= "s";
}

$sql .= " ORDER BY id DESC LIMIT 50";

if (!empty($params)) {
    $stmt = $mysqli->prepare($sql);
    $stmt->bind_param($types, ...$params);
    $stmt->execute();
    $resultado = $stmt->get_result();
} else {
    $resultado = $mysqli->query($sql);
}

$total_query = $mysqli->query("SELECT COUNT(*) AS total FROM equipaments");
$total_registros = $total_query->fetch_assoc()['total'];
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BCN Educación - Buscador y Gestión CRUD</title>
    <link href="[https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css](https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css)" rel="stylesheet">
    <link href="[https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css](https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css)" rel="stylesheet">
</head>
<body class="bg-light">
    <div class="container py-4">

        <!-- Alertas de Desbloqueo LDAP (Solo Admins) -->
        <?php if (isset($_GET['unlocked'])): ?>
            <div class="alert alert-success alert-dismissible fade show" role="alert">
                <i class="bi bi-unlock-fill"></i> Usuario <strong><?= htmlspecialchars($_GET['unlocked'], ENT_QUOTES, 'UTF-8') ?></strong> desbloqueado correctamente.
                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Cerrar"></button>
            </div>
        <?php endif; ?>

        <?php if ($es_admin_ldap && !empty($usuarios_bloqueados)): ?>
            <div class="alert alert-warning shadow-sm d-flex flex-wrap justify-content-between align-items-center gap-2" role="alert">
                <div>
                    <i class="bi bi-shield-exclamation"></i> <strong>Cuentas LDAP bloqueadas actualmente:</strong>
                    <?php foreach ($usuarios_bloqueados as $ub): ?>
                        <span class="badge bg-danger ms-1 py-1 px-2">
                            <?= htmlspecialchars($ub['uid'], ENT_QUOTES, 'UTF-8') ?>
                            <a href="?unlock_uid=<?= urlencode($ub['uid']) ?>" class="btn btn-light btn-sm py-0 px-1 ms-1 fw-bold text-danger" onclick="return confirm('¿Desbloquear la cuenta <?= htmlspecialchars($ub['uid'], ENT_QUOTES, 'UTF-8') ?>?')">Desbloquear</a>
                        </span>
                    <?php endforeach; ?>
                </div>
            </div>
        <?php endif; ?>

        <!-- Encabezado -->
        <div class="d-flex flex-wrap justify-content-between align-items-center mb-4 gap-3">
            <div>
                <h1 class="h2 text-primary m-0"><i class="bi bi-building"></i> Educación Barcelona</h1>
                <p class="text-muted m-0">Mostrando resultados sobre un total de <strong><?= number_format($total_registros) ?></strong> equipamientos.</p>
            </div>

            <div class="d-flex align-items-center gap-2">
                <span class="badge bg-primary-subtle text-primary-emphasis border border-primary-subtle px-3 py-2 fs-6">
                    <i class="bi bi-person-check-fill"></i> <?= htmlspecialchars($usuario_actual, ENT_QUOTES, 'UTF-8') ?>
                    <?php if ($es_admin_ldap): ?>
                        <span class="badge bg-warning text-dark ms-1">Admin</span>
                    <?php endif; ?>
                </span>
                <button class="btn btn-success" onclick="abrirModalCrear()">
                    <i class="bi bi-plus-lg"></i> Añadir Equipamiento
                </button>
                <a href="?logout=1" class="btn btn-outline-danger" title="Cerrar sesión">
                    <i class="bi bi-box-arrow-right"></i> Cerrar Sesión
                </a>
            </div>
        </div>

        <!-- Formulario de Filtros -->
        <form method="GET" class="card card-body mb-4 shadow-sm">
            <div class="row g-3">
                <div class="col-md-6">
                    <input type="text" name="q" class="form-control" placeholder="Buscar por nombre, calle o categoría..." value="<?= htmlspecialchars($busqueda, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>">
                </div>
                <div class="col-md-4">
                    <select name="distrito" class="form-select">
                        <option value="">-- Todos los distritos --</option>
                        <?php while ($d = $distritos_query->fetch_assoc()): ?>
                            <?php $dist_val = normalizar_texto($d['district']); ?>
                            <option value="<?= htmlspecialchars($dist_val, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>" <?= $distrito === $dist_val ? 'selected' : '' ?>>
                                <?= htmlspecialchars($dist_val, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
                            </option>
                        <?php endwhile; ?>
                    </select>
                </div>
                <div class="col-md-2 d-flex gap-2">
                    <button type="submit" class="btn btn-primary w-100">Filtrar</button>
                    <?php if ($busqueda !== '' || $distrito !== ''): ?>
                        <a href="index.php" class="btn btn-outline-secondary" title="Limpiar filtro"><i class="bi bi-x-lg"></i></a>
                    <?php endif; ?>
                </div>
            </div>
        </form>

        <!-- Tabla de Datos -->
        <div class="card shadow-sm">
            <div class="table-responsive">
                <table class="table table-hover table-striped align-middle mb-0">
                    <thead class="table-dark">
                        <tr>
                            <th>Reg. ID</th>
                            <th>Nombre</th>
                            <th>Dirección</th>
                            <th>Barrio</th>
                            <th>Distrito</th>
                            <th>Categoría</th>
                            <th class="text-center">Acciones</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php if ($resultado && $resultado->num_rows > 0): ?>
                            <?php while ($row = $resultado->fetch_assoc()): ?>
                                <?php
                                    $reg_id    = normalizar_texto($row['register_id']);
                                    $nombre    = normalizar_texto($row['name']);
                                    $calle     = normalizar_texto($row['address_road']);
                                    $numero    = normalizar_texto($row['address_number']);
                                    $barrio    = normalizar_texto($row['neighborhood']);
                                    $dist      = normalizar_texto($row['district']);
                                    $categoria = normalizar_texto($row['category']);
                                    $direccion = trim($calle . ' ' . $numero);
                                ?>
                                <tr>
                                    <td><code><?= htmlspecialchars($reg_id, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></code></td>
                                    <td><strong><?= htmlspecialchars($nombre, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></strong></td>
                                    <td><?= htmlspecialchars($direccion !== '' ? $direccion : '-', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></td>
                                    <td><?= htmlspecialchars($barrio !== '' ? $barrio : '-', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></td>
                                    <td><?= htmlspecialchars($dist !== '' ? $dist : '-', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></td>
                                    <td><span class="badge bg-info text-dark"><?= htmlspecialchars($categoria !== '' ? $categoria : 'General', ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></span></td>
                                    <td class="text-center text-nowrap">
                                        <button class="btn btn-sm btn-outline-warning me-1" onclick='abrirModalEditar(<?= json_encode([
                                            'id' => $row['id'],
                                            'register_id' => $reg_id,
                                            'name' => $nombre,
                                            'address_road' => $calle,
                                            'address_number' => $numero,
                                            'neighborhood' => $barrio,
                                            'district' => $dist,
                                            'category' => $categoria
                                        ], JSON_HEX_APOS | JSON_HEX_QUOT | JSON_UNESCAPED_UNICODE) ?>)'>
                                            <i class="bi bi-pencil-fill"></i> Editar
                                        </button>
                                        <a href="?action=delete&id=<?= $row['id'] ?>" class="btn btn-sm btn-outline-danger" onclick="return confirm('¿Estás seguro de que deseas eliminar este equipamiento?')">
                                            <i class="bi bi-trash-fill"></i> Eliminar
                                        </a>
                                    </td>
                                </tr>
                            <?php endwhile; ?>
                        <?php else: ?>
                            <tr>
                                <td colspan="7" class="text-center py-4 text-muted">No se encontraron registros que coincidan con la búsqueda.</td>
                            </tr>
                        <?php endif; ?>
                    </tbody>
                </table>
            </div>
        </div>
    </div>

    <!-- Modal Formulario CRUD -->
    <div class="modal fade" id="modalEquipamiento" tabindex="-1" aria-hidden="true">
        <div class="modal-dialog modal-lg">
            <div class="modal-content">
                <form method="POST" action="index.php">
                    <div class="modal-header">
                        <h5 class="modal-title" id="modalTitulo">Añadir Equipamiento</h5>
                        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Cerrar"></button>
                    </div>
                    <div class="modal-body">
                        <input type="hidden" name="id" id="form_id">
                        <div class="row g-3">
                            <div class="col-md-4">
                                <label class="form-label">Reg. ID</label>
                                <input type="text" name="register_id" id="form_register_id" class="form-control" required placeholder="Ej: 9940001">
                            </div>
                            <div class="col-md-8">
                                <label class="form-label">Nombre del centro</label>
                                <input type="text" name="name" id="form_name" class="form-control" required placeholder="Ej: Escola BCN">
                            </div>
                            <div class="col-md-8">
                                <label class="form-label">Calle / Dirección</label>
                                <input type="text" name="address_road" id="form_address_road" class="form-control" placeholder="Ej: Av. Diagonal">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Número</label>
                                <input type="text" name="address_number" id="form_address_number" class="form-control" placeholder="Ej: 123">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Barrio</label>
                                <input type="text" name="neighborhood" id="form_neighborhood" class="form-control" placeholder="Ej: la Vila de Gràcia">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Distrito</label>
                                <input type="text" name="district" id="form_district" class="form-control" placeholder="Ej: Gràcia">
                            </div>
                            <div class="col-md-4">
                                <label class="form-label">Categoría</label>
                                <input type="text" name="category" id="form_category" class="form-control" placeholder="Ej: Educació primària">
                            </div>
                        </div>
                    </div>
                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancelar</button>
                        <button type="submit" class="btn btn-primary" id="btnGuardar">Guardar</button>
                    </div>
                </form>
            </div>
        </div>
    </div>

    <script src="[https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js](https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js)"></script>
    <script>
        const modalElement = document.getElementById('modalEquipamiento');
        const bsModal = new bootstrap.Modal(modalElement);

        function abrirModalCrear() {
            document.getElementById('modalTitulo').innerText = 'Añadir Nuevo Equipamiento';
            document.getElementById('form_id').value = '';
            document.getElementById('form_register_id').value = '';
            document.getElementById('form_name').value = '';
            document.getElementById('form_address_road').value = '';
            document.getElementById('form_address_number').value = '';
            document.getElementById('form_neighborhood').value = '';
            document.getElementById('form_district').value = '';
            document.getElementById('form_category').value = '';
            document.getElementById('btnGuardar').innerText = 'Crear Equipamiento';
            bsModal.show();
        }

        function abrirModalEditar(data) {
            document.getElementById('modalTitulo').innerText = 'Editar Equipamiento #' + data.id;
            document.getElementById('form_id').value = data.id;
            document.getElementById('form_register_id').value = data.register_id || '';
            document.getElementById('form_name').value = data.name || '';
            document.getElementById('form_address_road').value = data.address_road || '';
            document.getElementById('form_address_number').value = data.address_number || '';
            document.getElementById('form_neighborhood').value = data.neighborhood || '';
            document.getElementById('form_district').value = data.district || '';
            document.getElementById('form_category').value = data.category || '';
            document.getElementById('btnGuardar').innerText = 'Actualizar Cambios';
            bsModal.show();
        }
    </script>
</body>
</html>
```

---

## 6. Comandos de Verificación y Mantenimiento

Para comprobar el correcto funcionamiento de nuestras políticas y realizar tareas de auditoría directa desde la consola de `LDAP-NCC`, disponemos de los siguientes comandos:

### Consultar estado de políticas e intentos fallidos de un usuario

```bash
ldapsearch -x -D "cn=admin,dc=educacion,dc=local" -W \
  -b "uid=bchecker,ou=usuarios,dc=educacion,dc=local" \
  pwdAccountLockedTime pwdFailureTime
```

### Desbloqueo manual desde consola en `LDAP-NCC`

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: uid=bchecker,ou=usuarios,dc=educacion,dc=local
changetype: modify
delete: pwdAccountLockedTime
-
delete: pwdFailureTime
EOF
```

### Comprobar miembros del grupo `admins`

```bash
ldapsearch -x -D "cn=admin,dc=educacion,dc=local" -W \
  -b "cn=admins,ou=grupos,dc=educacion,dc=local"
```
