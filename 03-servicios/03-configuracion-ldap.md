# Configuración de Autenticación Centralizada OpenLDAP, `ppolicy` y LAM (`LDAP-NCC`)

Este documento recoge la arquitectura, el despliegue y la puesta en marcha del servidor de directorio **OpenLDAP (`LDAP-NCC` — `10.10.10.30`)**, la interfaz gráfica de administración **LDAP Account Manager (LAM)**, la política de bloqueo automático de cuentas (**`ppolicy`**) y su integración en el portal web **Educación Barcelona (`WEB-NCC` — `10.10.10.20`)**.

---

## 1. Arquitectura y Árbol del Directorio (`dc=educacion,dc=local`)

| Componente | Hostname / URL | Dirección IP / Puerto | Función Principal |
| :--- | :--- | :--- | :--- |
| **Servidor Directorio** | `LDAP-NCC` | `10.10.10.30` (TCP `389`) | Contenedor con OpenLDAP (`dc=educacion,dc=local`), módulo `ppolicy` y LAM (`/lam`). |
| **Servidor Web** | `WEB-NCC` (`[http://educacion.local](http://educacion.local)`) | `10.10.10.20` (TCP `80/443`) | Portal PHP 8.2-FPM + Nginx con autenticación LDAP, control de bloqueos y CRUD. |
| **Base de Datos** | `B-NCC` | `10.10.20.10` (TCP `3306`) | Base de datos MySQL `bcn_educacio` (tabla `equipaments`). |

```text
dc=educacion,dc=local
├── cn=admin                       (Administrador global del directorio)
├── ou=usuarios                    (Cuentas: posixAccount + inetOrgPerson + shadowAccount)
│   ├── uid=admin                  (Administrador web - GID primario 10000 / memberUid en admins)
│   └── uid=bchecker               (Usuario estándar de aplicación y auditoría)
├── ou=grupos                      (Grupos POSIX)
│   └── cn=admins                  (gidNumber: 10000 - Otorga permisos de desbloqueo en WEB-NCC)
└── ou=policies                    (Políticas de contraseñas ppolicy)
    └── cn=default                 (3 intentos fallidos, 15 minutos / 900s de bloqueo)
```

---

## 2. Despliegue de OpenLDAP y Política `ppolicy` en `LDAP-NCC`

### 2.1. Instalación y Unidades Organizativas (`estructura_base.ldif`)
```bash
apt update && apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```
Parámetros configurados: Dominio `educacion.local` (`dc=educacion,dc=local`), Organización `Educacion Barcelona`, Motor `MDB`.

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
```bash
ldapadd -x -D "cn=admin,dc=educacion,dc=local" -W -f estructura_base.ldif
```

### 2.2. Activación de `ppolicy` (Bloqueo tras 3 fallos durante 15 minutos)
```bash
# Cargar esquema y módulo ppolicy.la
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/ppolicy.ldif 2>/dev/null || true

ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: ppolicy.la
EOF

# Crear política cn=default (3 intentos fallidos, bloqueo de 900 segundos)
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

# Vincular overlay ppolicy a la base de datos MDB
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

## 3. Instalación de LDAP Account Manager (LAM)

```bash
apt install -y apache2 php php-ldap php-xml php-mbstring php-zip php-gd ldap-account-manager
systemctl enable --now apache2
```
Acceso web en **`[http://10.10.10.30/lam](http://10.10.10.30/lam)`**, configurado con sufijo `dc=educacion,dc=local`, usuarios en `ou=usuarios,dc=educacion,dc=local` (`inetOrgPerson`, `posixAccount`, `shadowAccount`) y grupos en `ou=grupos,dc=educacion,dc=local` (`posixGroup`).

---

## 4. Integración en el Portal Web (`/var/www/html/index.php` en `WEB-NCC`)

Integra autenticación LDAP con patrón **PRG (Post/Redirect/Get)**, diagnóstico en tiempo real de intentos restantes y bloqueo `ppolicy`, panel de desbloqueo en 1 clic para miembros del grupo `admins` (`gidNumber=10000`) y gestión CRUD segura sobre MySQL (`10.10.20.10`):

```php
<?php
session_start();
header('Content-Type: text/html; charset=utf-8');

// ==========================================
// CONFIGURACIÓN LDAP (ppolicy y desbloqueo admin)
// ==========================================
define('LDAP_URI',          'ldap://10.10.10.30:389');
define('LDAP_USERS_OU',     'ou=usuarios,dc=educacion,dc=local');
define('LDAP_GROUPS_OU',    'ou=grupos,dc=educacion,dc=local');
define('LDAP_READ_DN',      'cn=admin,dc=educacion,dc=local');
define('LDAP_READ_PASS',    'Admin1234');
define('LDAP_MAX_FAILURES', 3);
define('LDAP_ADMINS_GID',   10000);

if (isset($_GET['logout'])) {
    session_unset();
    session_destroy();
    header("Location: index.php");
    exit;
}

if ($_SERVER['REQUEST_METHOD'] === 'POST' && ($_POST['action'] ?? '') === 'ldap_login') {
    $ldap_user = trim($_POST['auth_uid_field'] ?? '');
    $ldap_pass = $_POST['auth_pwd_field'] ?? '';

    if ($ldap_user !== '' && $ldap_pass !== '') {
        $safe_user = ldap_escape($ldap_user, "", LDAP_ESCAPE_DN);
        $user_dn   = "uid={$safe_user}," . LDAP_USERS_OU;
        $ldap_conn = @ldap_connect(LDAP_URI);

        if ($ldap_conn) {
            ldap_set_option($ldap_conn, LDAP_OPT_PROTOCOL_VERSION, 3);
            ldap_set_option($ldap_conn, LDAP_OPT_REFERRALS, 0);
            ldap_set_option($ldap_conn, LDAP_OPT_NETWORK_TIMEOUT, 5);

            if (@ldap_bind($ldap_conn, $user_dn, $ldap_pass)) {
                session_regenerate_id(true);
                $_SESSION['ldap_user'] = $ldap_user;
                unset($_SESSION['ldap_error']);
                @ldap_unbind($ldap_conn);
                header("Location: index.php");
                exit;
            } else {
                $mensaje_error = "Usuario o contraseña incorrectos.";
                $tipo_alerta   = "danger";
                $diag_conn = @ldap_connect(LDAP_URI);

                if ($diag_conn) {
                    ldap_set_option($diag_conn, LDAP_OPT_PROTOCOL_VERSION, 3);
                    ldap_set_option($diag_conn, LDAP_OPT_REFERRALS, 0);
                    ldap_set_option($diag_conn, LDAP_OPT_NETWORK_TIMEOUT, 5);

                    if (@ldap_bind($diag_conn, LDAP_READ_DN, LDAP_READ_PASS)) {
                        $sr = @ldap_read($diag_conn, $user_dn, "(objectClass=*)", ["pwdAccountLockedTime", "pwdFailureTime", "uid"]);
                        if ($sr && ($entries = ldap_get_entries($diag_conn, $sr)) && $entries['count'] > 0) {
                            $info = $entries[0];
                            if (isset($info['pwdaccountlockedtime']) && $info['pwdaccountlockedtime']['count'] > 0) {
                                $mensaje_error = "<strong>Cuenta bloqueada:</strong> Has superado el límite de intentos fallidos. Espera 15 minutos o contacta con un administrador.";
                                $tipo_alerta   = "dark";
                            } else {
                                $fallos = isset($info['pwdfailuretime']) ? intval($info['pwdfailuretime']['count']) : 0;
                                $restantes = max(0, LDAP_MAX_FAILURES - $fallos);
                                if ($restantes === 0) {
                                    $mensaje_error = "<strong>Cuenta bloqueada:</strong> Has agotado todos los intentos permitidos.";
                                    $tipo_alerta   = "dark";
                                } elseif ($restantes === 1) {
                                    $mensaje_error = "Contraseña incorrecta. <strong>¡Último intento!</strong> Si vuelves a fallar, tu cuenta quedará bloqueada.";
                                    $tipo_alerta   = "warning";
                                } else {
                                    $mensaje_error = "Contraseña incorrecta. Te quedan <strong>{$restantes} intentos</strong> antes del bloqueo.";
                                    $tipo_alerta   = "danger";
                                }
                            }
                        } else {
                            $mensaje_error = "El usuario <strong>" . htmlspecialchars($ldap_user, ENT_QUOTES, 'UTF-8') . "</strong> no existe en el directorio.";
                        }
                    }
                    @ldap_unbind($diag_conn);
                }
                $_SESSION['ldap_error'] = ['type' => $tipo_alerta, 'msg' => $mensaje_error];
                @ldap_unbind($ldap_conn);
                header("Location: index.php");
                exit;
            }
        }
    }
}
?>
```

---

## 5. Comandos de Auditoría y Desbloqueo Manual (`LDAP-NCC`)

```bash
# Consultar estado de bloqueo e intentos fallidos de un usuario
ldapsearch -x -D "cn=admin,dc=educacion,dc=local" -W \
  -b "uid=bchecker,ou=usuarios,dc=educacion,dc=local" \
  pwdAccountLockedTime pwdFailureTime

# Desbloquear cuenta manualmente desde la consola de LDAP-NCC
ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: uid=bchecker,ou=usuarios,dc=educacion,dc=local
changetype: modify
delete: pwdAccountLockedTime
-
delete: pwdFailureTime
EOF
```
