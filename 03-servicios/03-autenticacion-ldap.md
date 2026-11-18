# Autenticació centralitzada LDAP

## Components
| Component | Dada documentada | Funció |
|---|---|---|
| LDAP-NCC | 10.10.10.30:389 | OpenLDAP |
| Domini LDAP | `dc=educacion,dc=local` | Sufix del directori |
| LAM | `http://10.10.10.30/lam` | Administració web |
| WEB-NCC | `educacion.local` | Aplicació integrada |
| B-NCC | 10.10.20.10:3306 | MySQL |

## Estructura del directori
El directori es divideix en les unitats organitzatives:
- `ou=usuarios`: comptes d'usuari.
- `ou=grupos`: grups POSIX.
- `ou=policies`: polítiques de contrasenya.

S'hi documenten els comptes `admin` i `bchecker`, i el grup `admins`.

## Instal·lació d'OpenLDAP
Paquets documentats: `slapd` i `ldap-utils`. La configuració inicial utilitza el domini `educacion.local`, l'organització `Educacion Barcelona` i el motor MDB.

## Política ppolicy
La política descrita bloqueja un compte després de 3 intents fallits durant 900 segons (15 minuts). L'overlay `ppolicy` s'associa a la base MDB i utilitza la política predeterminada de `ou=policies`.

## LDAP Account Manager
LAM s'instal·la amb Apache i les extensions PHP necessàries. El perfil del servidor utilitza:
- Servidor LDAP: `ldap://localhost:389`.
- Sufix: `dc=educacion,dc=local`.
- Sufix d'usuaris: `ou=usuarios,dc=educacion,dc=local`.
- Sufix de grups: `ou=grupos,dc=educacion,dc=local`.

## Integració amb WEB-NCC
La documentació del portal descriu:
- Autenticació d'usuaris mitjançant LDAP.
- Regeneració de la sessió després d'un inici de sessió correcte.
- Consulta dels atributs operatius de `ppolicy` per detectar comptes bloquejats.
- Permisos d'administració associats al grup `admins`.
- Desbloqueig d'usuaris eliminant els atributs operatius de bloqueig.

## Consultes de diagnòstic
Consultar els atributs de bloqueig d'un usuari:

```bash
ldapsearch -x -D "cn=admin,dc=educacion,dc=local" -W \
  -b "uid=USUARI,ou=usuarios,dc=educacion,dc=local" \
  pwdAccountLockedTime pwdFailureTime
```

Les credencials no s'inclouen al repositori. Els comandos que requereixen autenticació demanen la contrasenya de manera interactiva.
