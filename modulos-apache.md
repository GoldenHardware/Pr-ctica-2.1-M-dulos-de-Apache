# Práctica 2.1 — Módulos de Apache

## 1. ¿Qué es un módulo de Apache?

Apache tiene una arquitectura **modular**: el núcleo (`core`) solo sabe aceptar conexiones y servir ficheros, y el resto de funcionalidades (HTTPS, proxy, reescritura de URLs, cabeceras, autenticación, compresión…) se añaden mediante **módulos** que se cargan con la directiva `LoadModule`.

En Debian/Ubuntu los módulos se gestionan así:

| Ruta / comando | Función |
|---|---|
| `/etc/apache2/mods-available/` | Todos los módulos instalados (`.load` + `.conf`) |
| `/etc/apache2/mods-enabled/` | Enlaces simbólicos a los módulos activos |
| `sudo a2enmod <módulo>` | Activa un módulo (crea el enlace) |
| `sudo a2dismod <módulo>` | Desactiva un módulo (borra el enlace) |
| `apache2ctl -M` | Lista los módulos cargados (`static` = compilados dentro, `shared` = cargados como .so) |

**¿Por qué hay que activarlos?** Por defecto solo se cargan los módulos imprescindibles. Cargar solo lo necesario reduce el consumo de memoria y la **superficie de ataque** (menos código expuesto = menos vulnerabilidades posibles). Por eso activamos únicamente los que el proyecto necesita.

## 2. Módulos activados

```bash
sudo a2enmod proxy proxy_http ssl rewrite headers
sudo systemctl restart apache2
```

> Nota: al activar `ssl`, `a2enmod` activa automáticamente su dependencia `socache_shmcb` (caché de sesiones SSL en memoria compartida).

### 2.1 `proxy`
**Propósito:** módulo base de proxy. Permite que Apache actúe como **proxy inverso** (o de reenvío): recibe la petición del cliente y la reenvía a otro servidor. Por sí solo no sabe hablar ningún protocolo; necesita submódulos como `proxy_http`.

```apache
# Nunca activar proxy de reenvío abierto
ProxyRequests Off
ProxyPreserveHost On
```

### 2.2 `proxy_http`
**Propósito:** submódulo de `proxy` que añade soporte para **HTTP/HTTPS**. Es el que permite redirigir peticiones a una aplicación backend (Node.js, Tomcat, Gunicorn, un contenedor Docker…) que escucha en otro puerto.

```apache
<VirtualHost *:80>
    ServerName app.local
    ProxyPreserveHost On
    ProxyPass        /  http://127.0.0.1:3000/
    ProxyPassReverse /  http://127.0.0.1:3000/
</VirtualHost>
```

### 2.3 `ssl`
**Propósito:** añade soporte de **HTTPS** (TLS) usando OpenSSL. Permite cifrar la comunicación entre cliente y servidor mediante certificados. Habilita el puerto 443 (`Listen 443` en `ports.conf`).

```apache
<VirtualHost *:443>
    ServerName app.local
    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
    SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
    DocumentRoot /var/www/html
</VirtualHost>
```

### 2.4 `rewrite`
**Propósito:** motor de **reescritura de URLs** basado en expresiones regulares. Sirve para URLs amigables, redirecciones (por ejemplo HTTP → HTTPS), front controllers de frameworks (Laravel, WordPress…) y reglas en `.htaccess`.

```apache
# Redirigir todo el tráfico HTTP a HTTPS
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
```

### 2.5 `headers`
**Propósito:** permite **añadir, modificar o eliminar cabeceras HTTP** de peticiones y respuestas. Se usa sobre todo para cabeceras de seguridad, CORS y control de caché.

```apache
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-Content-Type-Options "nosniff"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Header unset X-Powered-By
```

## 3. Verificación

```bash
# Comprobar sintaxis
sudo apache2ctl configtest
# -> Syntax OK

# Reiniciar y comprobar estado
sudo systemctl restart apache2
sudo systemctl status apache2 --no-pager

# Verificación filtrada de los 5 módulos
apache2ctl -M | grep -E 'proxy_module|proxy_http_module|ssl_module|rewrite_module|headers_module'
```

Salida esperada:

```
 headers_module (shared)
 proxy_module (shared)
 proxy_http_module (shared)
 rewrite_module (shared)
 ssl_module (shared)
```

> Para que el reinicio no muestre el aviso `AH00558: Could not reliably determine the server's fully qualified domain name` se ha añadido `ServerName localhost` en `/etc/apache2/conf-available/servername.conf` y se ha activado con `sudo a2enconf servername`.

## 4. Otros módulos relevantes

| Módulo | Para qué sirve |
|---|---|
| `proxy_balancer` + `lbmethod_byrequests` | Balanceo de carga entre varios backends |
| `proxy_wstunnel` | Proxy de WebSockets |
| `http2` | Soporte del protocolo HTTP/2 |
| `deflate` | Compresión gzip de las respuestas |
| `expires` | Cabeceras de caché (`Expires`, `Cache-Control`) |
| `security2` (ModSecurity) | Firewall de aplicaciones web (WAF) |
| `evasive` | Mitigación de ataques DoS/fuerza bruta |
| `auth_basic` / `authz_core` | Autenticación y control de acceso |
| `userdir` | Webs personales en `~/public_html` |
| `status` | Página `/server-status` con el estado del servidor |
