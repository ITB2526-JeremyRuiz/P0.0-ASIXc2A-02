# Configuración de Cifrado HTTPS (TLS) en el Servidor Web (`WEB-NCC`)

Este documento detalla la implementación del protocolo **HTTPS (TLS 1.2 / TLS 1.3)** en el servidor **`WEB-NCC`** (`CT 105` · `10.10.10.20`), la generación del certificado X.509 con extensión **SAN (*Subject Alternative Name*)**, la redirección automática desde HTTP (`80`) hacia HTTPS (`443`) y la importación del certificado en los equipos cliente.

---

## 1. Justificación de Seguridad

En la arquitectura del proyecto, el portal **Educación Barcelona (`educacion.local`)** transmite credenciales de usuario hacia el directorio **OpenLDAP (`10.10.10.30`)** y gestiona registros de la base de datos **`B-NCC` (`10.10.20.10`)**. Habilitar **HTTPS** elimina el envío de contraseñas y cookies de sesión en texto plano tanto en la red local (`vmbr2`) como en el acceso remoto a través de la VPN de IsardVDI (`enp2s0`).

| Parámetro | Configuración Aplicada |
| :--- | :--- |
| **Servidor Web** | `WEB-NCC` (`10.10.10.20/24` · DMZ `vmbr1`) |
| **Virtual Host Activo** | `/etc/nginx/sites-available/default` |
| **Socket Backend PHP** | `unix:/run/php/php8.3-fpm.sock` |
| **Certificado Público** | `/etc/nginx/ssl/educacion.local.crt` (X.509 v3, validez 365 días) |
| **Clave Privada** | `/etc/nginx/ssl/educacion.local.key` (RSA 2048 bits, permisos `600`) |
| **Extensiones SAN** | `DNS:educacion.local`, `DNS:www.educacion.local`, `DNS:web-ncc`, `IP:10.10.10.20`, `IP:10.2.169.223` |
| **Protocolos Cifrados** | `TLSv1.2`, `TLSv1.3` con redirección permanente `301` desde el puerto `80` |

---

## 2. Generación del Certificado TLS con OpenSSL (`WEB-NCC`)

En el contenedor `WEB-NCC` se creó el directorio `/etc/nginx/ssl` y se generó un certificado autofirmado con nombres alternativos del sujeto (**SAN**) para cubrir todos los dominios locales y direcciones IP internas y VPN:

```bash
mkdir -p /etc/nginx/ssl

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/educacion.local.key \
  -out /etc/nginx/ssl/educacion.local.crt \
  -subj "/C=ES/ST=Barcelona/L=Barcelona/O=Educacion Barcelona/OU=ASIX/CN=educacion.local" \
  -addext "subjectAltName=DNS:educacion.local,DNS:www.educacion.local,DNS:web-ncc,IP:10.10.10.20,IP:10.2.169.223"

chmod 600 /etc/nginx/ssl/educacion.local.key
```

### Verificación de Metadatos y Extensiones SAN
```bash
openssl x509 -in /etc/nginx/ssl/educacion.local.crt -noout -subject -dates -ext subjectAltName
```

---

## 3. Configuración del Virtual Host Nginx (`/etc/nginx/sites-available/default`)

Se configuraron dos bloques `server` en Nginx: el primero escucha en el puerto `80` y fuerza una redirección permanente (`301`) hacia `https://`, mientras que el segundo sirve la aplicación PHP 8.3-FPM cifrada en el puerto `443` incorporando cabeceras de endurecimiento (`X-Frame-Options` y `X-Content-Type-Options`):

```nginx
# 1. Redirección permanente de HTTP (80) a HTTPS (443)
server {
    listen 80;
    server_name educacion.local www.educacion.local web-ncc 10.10.10.20 _;
    return 301 https://$host$request_uri;
}

# 2. Servidor seguro HTTPS (443)
server {
    listen 443 ssl;
    server_name educacion.local www.educacion.local web-ncc 10.10.10.20 _;

    ssl_certificate /etc/nginx/ssl/educacion.local.crt;
    ssl_certificate_key /etc/nginx/ssl/educacion.local.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Cabeceras de seguridad web
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri$uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

### Validación de Sintaxis y Recarga del Servicio
```bash
nginx -t && systemctl reload nginx
ss -tlnp | grep -E ':80|:443'
```

---

## 4. Enrutamiento VPN en Proxmox VE (`pvexx`)

El nodo hipervisor Proxmox (`pvexx`) dispone en `/etc/network/interfaces` de la regla `DNAT` persistente en la interfaz VPN `enp2s0` (`10.2.169.223`) para reenviar el tráfico entrante HTTPS directamente hacia `WEB-NCC`:

```bash
post-up iptables -t nat -A PREROUTING -i enp2s0 -p tcp --dport 443 -j DNAT --to-destination 10.10.10.20:443
```

---

## 5. Confianza del Certificado en Equipos Cliente (Opcional)

Al tratarse de un certificado autofirmado para un dominio privado (`.local`), los navegadores muestran inicialmente el aviso `MOZILLA_PKIX_ERROR_SELF_SIGNED_CERT` / `SEC_ERROR_UNKNOWN_ISSUER`. Para eliminar dicha advertencia en los clientes autorizados:

* **En Clientes Linux (`CLI-LINUX`):**
  ```bash
  scp bchecker@10.10.10.20:/etc/nginx/ssl/educacion.local.crt /tmp/
  sudo cp /tmp/educacion.local.crt /usr/local/share/ca-certificates/
  sudo update-ca-certificates
  ```
* **En Navegadores (Firefox / Windows):**
  Importar `educacion.local.crt` en **Ajustes $\rightarrow$ Privacidad y seguridad $\rightarrow$ Certificados $\rightarrow$ Ver certificados $\rightarrow$ Autoridades** marcando *"Confiar en esta CA para identificar sitios web"*.

---

## 6. Pruebas de Verificación

```bash
# 1. Comprobar redirección HTTP -> HTTPS (debe devolver HTTP/1.1 301 Moved Permanently)
curl -I [http://educacion.local](http://educacion.local)

# 2. Comprobar respuesta cifrada HTTPS (debe devolver HTTP/1.1 200 OK)
curl -Ik [https://educacion.local](https://educacion.local)
```
