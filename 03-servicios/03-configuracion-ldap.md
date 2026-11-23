# Configuración de Autenticación Centralizada OpenLDAP, `ppolicy` y LAM (`LDAP-NCC`)

Este documento recoge la arquitectura, el despliegue, la verificación y el mantenimiento del servidor de directorio **OpenLDAP (`LDAP-NCC` — `10.10.10.30`)**, la interfaz de administración **LDAP Account Manager (LAM)**, la política de bloqueo automático de cuentas (**`ppolicy`**) y su integración en el portal web **Educación Barcelona (`WEB-NCC` — `10.10.10.20`)**.

**Índice**

1. [Arquitectura y árbol del directorio](#1-arquitectura-y-árbol-del-directorio-dceducaciondclocal)
2. [Despliegue de OpenLDAP y política `ppolicy`](#2-despliegue-de-openldap-y-política-ppolicy-en-ldap-ncc)
3. [Instalación de LDAP Account Manager (LAM)](#3-instalación-de-ldap-account-manager-lam)
4. [Integración en el portal web](#4-integración-en-el-portal-web)
5. [Auditoría y desbloqueo manual](#5-comandos-de-auditoría-y-desbloqueo-manual-ldap-ncc)
6. [Pruebas del servicio](#6-pruebas-del-servicio)
7. [Mantenimiento y resolución de problemas](#7-mantenimiento-y-resolución-de-problemas)
8. [Limitaciones conocidas y mejoras previstas](#8-limitaciones-conocidas-y-mejoras-previstas)

---

## 1. Arquitectura y Árbol del Directorio (`dc=educacion,dc=local`)

| Componente | Hostname / URL | Dirección IP / Puerto | Función principal |
| :--- | :--- | :--- | :--- |
| **Servidor de directorio** | `LDAP-NCC` | `10.10.10.30` (TCP `389`) | OpenLDAP (`dc=educacion,dc=local`), módulo `ppolicy` y LAM (`/lam`). |
| **Servidor web** | `WEB-NCC` (`http://educacion.local`) | `10.10.10.20` (TCP `80`) | Portal PHP 8.2-FPM + Nginx con autenticación LDAP, control de bloqueos y CRUD. |
| **Base de datos** | `B-NCC` | `10.10.20.10` (TCP `3306`) | MySQL `bcn_educacio` (tabla `equipaments`). |

La configuración de red de `LDAP-NCC` está en `02-Redes-y-seguridad/01-arquitectura-red.md`, sección 4.5.

<!-- CONFIRMAR: ID del contenedor LXC de LDAP-NCC y su sistema operativo (Debian o Ubuntu) para completar el inventario. -->

```text
dc=educacion,dc=local
├── cn=admin                       (Administrador global del directorio)
├── ou=usuarios                    (Cuentas: posixAccount + inetOrgPerson + shadowAccount)
│   ├── uid=admin                  (Administrador web - GID primario 10000 / memberUid en admins)
│   └── uid=bchecker               (Usuario estándar de aplicación y auditoría)
├── ou=grupos                      (Grupos POSIX)
│   └── cn=admins                  (gidNumber: 10000 - Otorga permisos de desbloqueo en WEB-NCC)
└── ou=policies                    (Políticas de contraseñas ppolicy)
    └── cn=default                 (3 intentos fallidos, 15 minutos / 900 s de bloqueo)
```

---

## 2. Despliegue de OpenLDAP y Política `ppolicy` en `LDAP-NCC`

### 2.1. Instalación y unidades organizativas (`estructura_base.ldif`)

```bash
apt update && apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```

Parámetros configurados: dominio `educacion.local` (`dc=educacion,dc=local`), organización `Educacion Barcelona`, motor `MDB`.

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

### 2.2. Usuarios y grupo de administradores (`usuarios.ldif`)

Las contraseñas se almacenan como hash. Se generan con `slappasswd` y se copian al LDIF:

```bash
slappasswd -s bchecker121     # hash para bchecker (contraseña exigida por el enunciado)
slappasswd                    # hash para admin (pide la contraseña por teclado)
```

```ldif
dn: cn=admins,ou=grupos,dc=educacion,dc=local
objectClass: posixGroup
cn: admins
gidNumber: 10000
memberUid: admin

dn: uid=admin,ou=usuarios,dc=educacion,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: admin
sn: admin
uid: admin
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/admin
loginShell: /bin/bash
userPassword: <hash_SSHA_de_admin>

dn: uid=bchecker,ou=usuarios,dc=educacion,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: bchecker
sn: bchecker
uid: bchecker
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/bchecker
loginShell: /bin/bash
userPassword: <hash_SSHA_de_bchecker>
```

> Los valores `uidNumber` y `gidNumber` de este LDIF son de ejemplo (el árbol solo fija `gidNumber: 10000` para `admins`). Sustitúyelos por los reales del directorio.

```bash
ldapadd -x -D "cn=admin,dc=educacion,dc=local" -W -f usuarios.ldif
```

### 2.3. Activación de `ppolicy` (bloqueo tras 3 fallos durante 15 minutos)

```bash
# Cargar el esquema de ppolicy (si ya está cargado, ldapadd devolverá un error que se puede ignorar)
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/ppolicy.ldif

# Cargar el módulo ppolicy.la
ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: ppolicy.la
EOF

# Crear la política cn=default (3 intentos fallidos, bloqueo de 900 segundos)
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

# Vincular el overlay ppolicy a la base de datos MDB
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

**Parámetros de la política:**

| Atributo | Valor | Significado |
| :--- | :--- | :--- |
| `pwdMaxFailure` | `3` | Intentos fallidos permitidos antes del bloqueo. |
| `pwdFailureCountInterval` | `300` | Los fallos se cuentan en una ventana de 5 minutos: pasado ese tiempo sin nuevos fallos, el contador se reinicia. |
| `pwdLockout` | `TRUE` | Activa el bloqueo de la cuenta. |
| `pwdLockoutDuration` | `900` | La cuenta queda bloqueada 15 minutos y se desbloquea sola. |

### 2.4. Verificación de la configuración

```bash
# Política cargada
ldapsearch -x -D "cn=admin,dc=educacion,dc=local" -W -LLL -s base \
  -b "cn=default,ou=policies,dc=educacion,dc=local"

# Overlay ppolicy vinculado a la base de datos
ldapsearch -Y EXTERNAL -H ldapi:/// -LLL \
  -b "olcDatabase={1}mdb,cn=config" "(objectClass=olcPPolicyConfig)"

# Autenticación correcta de bchecker (desde WEB-NCC o LDAP-NCC)
ldapwhoami -x -H ldap://10.10.10.30 \
  -D "uid=bchecker,ou=usuarios,dc=educacion,dc=local" -w bchecker121
```

---

## 3. Instalación de LDAP Account Manager (LAM)

```bash
apt install -y apache2 php php-ldap php-xml php-mbstring php-zip php-gd ldap-account-manager
systemctl enable --now apache2
```

Acceso web en **`http://10.10.10.30/lam`**, configurado con sufijo `dc=educacion,dc=local`, usuarios en `ou=usuarios,dc=educacion,dc=local` (`inetOrgPerson`, `posixAccount`, `shadowAccount`) y grupos en `ou=grupos,dc=educacion,dc=local` (`posixGroup`).

**Contraseña de configuración de LAM.** LAM instala por defecto una contraseña maestra de configuración conocida (`lam`). Debe cambiarse desde *LAM configuration → Edit server profiles / Edit general settings*, antes de dejar el servicio accesible desde la red.

---

## 4. Integración en el Portal Web

El portal (`/var/www/html/index.php` en `WEB-NCC`) integra la autenticación LDAP con el patrón **PRG (Post/Redirect/Get)**, el diagnóstico de intentos restantes y de bloqueo `ppolicy`, el panel de desbloqueo en 1 clic para los miembros del grupo `admins` (`gidNumber=10000`) y la gestión CRUD sobre MySQL (`10.10.20.10`). El fragmento siguiente corresponde al **inicio de sesión**; el CRUD está descrito en `03-servicios/01-configuracion-servidor-web.md`.

**Credenciales fuera del código.** La contraseña de la cuenta de lectura no se escribe en `index.php`. Se guarda en un fichero fuera del directorio web, legible solo por `root` y por el usuario del servidor web:

```bash
# En WEB-NCC
mkdir -p /etc/educacion
cat > /etc/educacion/ldap.php << 'EOF'
<?php
return [
    'read_dn'   => 'cn=admin,dc=educacion,dc=local',
    'read_pass' => '<contraseña_de_la_cuenta_de_lectura>',
];
EOF
chown root:www-data /etc/educacion/ldap.php
chmod 640 /etc/educacion/ldap.php
```

```php
<?php
session_start();
header('Content-Type: text/html; charset=utf-8');

// ==========================================
// CONFIGURACIÓN LDAP (ppolicy y desbloqueo admin)
// ==========================================
$ldap_cfg = require '/etc/educacion/ldap.php';

define('LDAP_URI',          'ldap://10.10.10.30:389');
define('LDAP_USERS_OU',     'ou=usuarios,dc=educacion,dc=local');
define('LDAP_GROUPS_OU',    'ou=grupos,dc=educacion,dc=local');
define('LDAP_READ_DN',      $ldap_cfg['read_dn']);
define('LDAP_READ_PASS',    $ldap_cfg['read_pass']);
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

> **Nota:** el cambio respecto a la versión anterior se limita a leer la contraseña de `/etc/educacion/ldap.php` en lugar de escribirla en el código. Está revisado pero **no se ha ejecutado**: debe repetirse SEG-01 y SEG-02 tras aplicarlo.

**Funcionamiento del diagnóstico.** Tras un inicio de sesión fallido, el portal consulta con la cuenta de lectura los atributos operativos `pwdAccountLockedTime` y `pwdFailureTime` del usuario. El número de valores de `pwdFailureTime` indica los fallos acumulados dentro de la ventana de 300 s, y de ahí se calculan los intentos restantes (3 menos los fallos).

---

## 5. Comandos de Auditoría y Desbloqueo Manual (`LDAP-NCC`)

```bash
# Consultar el estado de bloqueo e intentos fallidos de un usuario
ldapsearch -x -D "cn=admin,dc=educacion,dc=local" -W -s base \
  -b "uid=bchecker,ou=usuarios,dc=educacion,dc=local" \
  pwdAccountLockedTime pwdFailureTime

# Desbloquear una cuenta manualmente desde la consola de LDAP-NCC
ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: uid=bchecker,ou=usuarios,dc=educacion,dc=local
changetype: modify
delete: pwdAccountLockedTime
-
delete: pwdFailureTime
EOF
```

---

## 6. Pruebas del servicio

Complementan las pruebas SEG-01, SEG-02 y MON-03 de `02-Redes-y-seguridad/Contingencia-y-pruebas.md`, a donde deben incorporarse. Cada una se documenta con evidencia en `evidencias/`.

| ID | Prueba | Procedimiento | Resultado esperado | Evidencia | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **LDP-01** | Política cargada | `ldapsearch` sobre `cn=default,ou=policies,...` (sección 2.4) | `pwdMaxFailure: 3`, `pwdLockoutDuration: 900` | `evidencias/LDP-01.png` | **Pendiente de ejecutar** |
| **LDP-02** | Autenticación desde `WEB-NCC` | `ldapwhoami -x -H ldap://10.10.10.30 -D "uid=bchecker,..." -w bchecker121` | Devuelve `dn:uid=bchecker,ou=usuarios,dc=educacion,dc=local` | `evidencias/LDP-02.png` | **Pendiente de ejecutar** |
| **LDP-03** | Bloqueo por línea de comandos | Tres `ldapwhoami` con contraseña errónea y uno con la correcta; consultar `pwdAccountLockedTime` | El cuarto intento también falla y aparece `pwdAccountLockedTime` | `evidencias/LDP-03.png` | **Pendiente de ejecutar** |
| **LDP-04** | Desbloqueo automático | Tras LDP-03, esperar 15 minutos (900 s) y repetir `ldapwhoami` con la contraseña correcta | Autenticación correcta sin intervención | `evidencias/LDP-04.png` | **Pendiente de ejecutar** |
| **LDP-05** | Acceso a LAM | Abrir `http://10.10.10.30/lam` e iniciar sesión | Se muestra el árbol `dc=educacion,dc=local` con los usuarios y el grupo `admins` | `evidencias/LDP-05.png` | **Pendiente de ejecutar** |
| **LDP-06** | Copia y restauración del directorio | `slapcat` y restauración en un entorno de pruebas (sección 7.1) | El directorio restaurado contiene los mismos usuarios, grupos y política | `evidencias/LDP-06.png` | **Pendiente de ejecutar** |

---

## 7. Mantenimiento y Resolución de Problemas

### 7.1. Copia de seguridad y restauración

El contenedor `LDAP-NCC` debe incluirse en la copia `vzdump` (ver `Contingencia-y-pruebas.md`). Además, se recomienda una copia lógica del directorio:

```bash
# Copia de la configuración (base 0) y de los datos (base 1)
slapcat -n 0 -l /root/backup-ldap/config-$(date +%F).ldif
slapcat -n 1 -l /root/backup-ldap/datos-$(date +%F).ldif
```

Restauración de los datos (con `slapd` detenido y la base vacía):

```bash
systemctl stop slapd
slapadd -n 1 -l /root/backup-ldap/datos-<FECHA>.ldif
chown -R openldap:openldap /var/lib/ldap
systemctl start slapd
```

### 7.2. Operaciones habituales

| Tarea | Comando |
| :--- | :--- |
| Estado del servicio | `systemctl status slapd` |
| Registro de `slapd` | `journalctl -u slapd -f` |
| Reiniciar el directorio | `systemctl restart slapd` |
| Registro de LAM | `tail -f /var/log/apache2/error.log` |
| Copia de seguridad del contenedor (desde `pvexx`) | `vzdump <ID_LDAP> --mode snapshot --compress zstd --storage local` |

### 7.3. Resolución de problemas

| Síntoma | Causa probable | Comprobación y solución |
| :--- | :--- | :--- |
| El portal siempre dice «Usuario o contraseña incorrectos» | `LDAP-NCC` inaccesible desde `WEB-NCC` | `nc -zv 10.10.10.30 389` desde `WEB-NCC`; `systemctl status slapd`. |
| Error `Call to undefined function ldap_connect()` | Falta la extensión `php-ldap` | `apt install php8.2-ldap && systemctl restart php8.2-fpm`. |
| El usuario no se bloquea tras fallar varias veces | Overlay `ppolicy` no vinculado | Ejecutar la segunda comprobación de la sección 2.4. |
| El contador de intentos restantes no coincide | Pasaron más de 300 s entre fallos | Es el comportamiento de `pwdFailureCountInterval`: el contador se reinicia. |
| `ldap_bind(): Unable to bind ... Invalid credentials (49)` | Contraseña errónea o cuenta bloqueada | Consultar `pwdAccountLockedTime` (sección 5) y desbloquear si procede. |
| El portal no muestra el panel de desbloqueo | El usuario no pertenece a `admins` | `ldapsearch ... "(cn=admins)" memberUid` y comprobar que aparece el usuario. |
| LAM no carga | Apache o `php-ldap` | `systemctl status apache2` y revisar `/var/log/apache2/error.log`. |

---

## 8. Limitaciones Conocidas y Mejoras Previstas

1. **Tráfico LDAP sin cifrar.** El portal usa `ldap://` en el puerto `389`, de modo que la contraseña de cada usuario viaja en claro entre `WEB-NCC` y `LDAP-NCC`. Mejora: activar StartTLS o LDAPS con un certificado y usar `ldap_start_tls()` en el portal. Coincide con la propuesta de certificados del último MVP.
2. **Cuenta de lectura con privilegios de administrador.** El portal consulta los atributos de bloqueo con `cn=admin`, la cuenta de administración global del directorio. Mejora: crear una cuenta de solo lectura (por ejemplo `cn=portal,dc=educacion,dc=local`) y restringir con ACL los atributos que puede ver.
3. **ACL por defecto.** Las ACL por defecto de Debian y Ubuntu permiten leer casi todo el directorio, salvo las contraseñas, incluso sin autenticarse. Se pueden comprobar con:

   ```bash
   ldapsearch -Y EXTERNAL -H ldapi:/// -LLL -b "olcDatabase={1}mdb,cn=config" olcAccess
   ```

   Mejora: sustituir `by * read` por `by users read` y limitar el acceso por red.
4. **Mensajes que revelan información.** El portal distingue «usuario no existe», «te quedan N intentos» y «cuenta bloqueada». Es útil para el usuario, pero permite enumerar cuentas válidas. En un entorno real se usaría un mensaje genérico único.
5. **Acceso a LDAP y LAM desde la red de clientes.** El puerto `389` y LAM son alcanzables desde `192.168.2.0/24`. Mejora: filtrar en `pvexx` con una regla análoga a la de MySQL, por ejemplo:

   ```bash
   iptables -I FORWARD 1 -i vmbr2 -d 10.10.10.30 -p tcp --dport 389 -j DROP
   ```

   Estado: **pendiente de aplicar y probar**.
6. **Política de contraseñas sin requisitos de calidad.** La política `cn=default` no exige longitud ni complejidad. Se pueden añadir `pwdMinLength` y `pwdCheckQuality`.
7. **Servidor único.** `LDAP-NCC` es un punto único de fallo: si cae, nadie puede iniciar sesión en el portal. La contingencia actual es la restauración desde `vzdump` (CON-05); una réplica con `syncrepl` sería la solución completa.
8. **LAM sin HTTPS** y con la contraseña de configuración por defecto (ver sección 3).
