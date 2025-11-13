---
layout: post
title: WordPress Security Hardening - Defendiendo sitios de producción
author: Vicente José Moreno Escobar
categories: [WordPress, Seguridad, DevOps]
published: true
---

> Cuando tu sitio WordPress es atacado 1000+ veces al día

# El despertar

Logs del servidor mostraban:
- 5000+ intentos de login fallidos/día
- Requests a `/wp-admin` desde IPs sospechosas
- Escaneo de plugins vulnerables
- Intentos de SQL injection

Era hora de endurecer la seguridad seriamente.

## Vectores de ataque comunes

1. **Brute force en wp-login.php**
2. **Plugins/themes desactualizados**
3. **Usuarios con contraseñas débiles**
4. **File upload vulnerabilities**
5. **SQL injection**
6. **XSS en comentarios/campos de usuario**

## Hardening paso a paso

### 1. Forzar HTTPS everywhere

```php
// wp-config.php
define('FORCE_SSL_ADMIN', true);

// Redirigir todo a HTTPS
if (!isset($_SERVER['HTTPS']) || $_SERVER['HTTPS'] !== 'on') {
    $redirect = 'https://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
    header('HTTP/1.1 301 Moved Permanently');
    header('Location: ' . $redirect);
    exit();
}
```

### 2. Cambiar prefijo de DB

Por defecto: `wp_`

Atacantes asumen este prefijo para SQL injection.

```sql
-- Cambiar prefijo (hacer backup primero!)
RENAME TABLE wp_commentmeta TO empresa_commentmeta;
RENAME TABLE wp_comments TO empresa_comments;
-- ... repetir para todas las tablas
```

```php
// wp-config.php
$table_prefix = 'empresa_';
```

### 3. Deshabilitar file editing desde admin

```php
// wp-config.php
define('DISALLOW_FILE_EDIT', true);
define('DISALLOW_FILE_MODS', true); // También deshabilita install de plugins
```

Ahora atacantes no pueden editar files PHP desde wp-admin.

### 4. Limitar intentos de login

```php
// Plugin: Limit Login Attempts Reloaded
// O implementar manualmente:

// functions.php
function check_failed_login() {
    $ip = $_SERVER['REMOTE_ADDR'];
    $attempts = get_transient('failed_login_' . $ip);

    if ($attempts >= 5) {
        wp_die('Demasiados intentos fallidos. Intenta en 30 minutos.');
    }
}
add_action('wp_login_failed', 'log_failed_login');

function log_failed_login($username) {
    $ip = $_SERVER['REMOTE_ADDR'];
    $attempts = get_transient('failed_login_' . $ip) ?: 0;

    set_transient('failed_login_' . $ip, $attempts + 1, 30 * MINUTE_IN_SECONDS);
}
```

### 5. Ocultar versión de WordPress

```php
// functions.php
remove_action('wp_head', 'wp_generator');

// También remover de RSS feeds
add_filter('the_generator', '__return_empty_string');

// Remover de scripts/styles
function remove_version_from_assets($src) {
    if (strpos($src, 'ver=')) {
        $src = remove_query_arg('ver', $src);
    }
    return $src;
}
add_filter('style_loader_src', 'remove_version_from_assets', 9999);
add_filter('script_loader_src', 'remove_version_from_assets', 9999);
```

### 6. Deshabilitar XML-RPC

XML-RPC es usado para ataques DDoS y brute force.

```php
// functions.php
add_filter('xmlrpc_enabled', '__return_false');

// O bloquear en .htaccess
<Files xmlrpc.php>
    order deny,allow
    deny from all
</Files>
```

### 7. Proteger wp-config.php

```apache
# .htaccess
<files wp-config.php>
    order allow,deny
    deny from all
</files>
```

Mejor aún: **Mover wp-config.php fuera de web root**:

```
/var/www/
    html/           # Web root
        wp-admin/
        wp-content/
        index.php
    wp-config.php   # Un nivel arriba, inaccesible vía web
```

### 8. Restringir acceso a wp-admin por IP

```apache
# .htaccess en wp-admin/
<Limit GET POST>
    order deny,allow
    deny from all
    allow from 203.0.113.50    # IP oficina
    allow from 198.51.100.0/24 # Rango corporativo
</Limit>
```

### 9. Security keys

Generar nuevas desde https://api.wordpress.org/secret-key/1.1/salt/

```php
// wp-config.php
define('AUTH_KEY',         'valor-unico-generado');
define('SECURE_AUTH_KEY',  'valor-unico-generado');
define('LOGGED_IN_KEY',    'valor-unico-generado');
define('NONCE_KEY',        'valor-unico-generado');
define('AUTH_SALT',        'valor-unico-generado');
define('SECURE_AUTH_SALT', 'valor-unico-generado');
define('LOGGED_IN_SALT',   'valor-unico-generado');
define('NONCE_SALT',       'valor-unico-generado');
```

Cambiarlas fuerza re-login de todos los usuarios.

### 10. File permissions correctos

```bash
# Directories: 755
find /var/www/html -type d -exec chmod 755 {} \;

# Files: 644
find /var/www/html -type f -exec chmod 644 {} \;

# wp-config.php: 400 (solo lectura para owner)
chmod 400 wp-config.php

# .htaccess: 644
chmod 644 .htaccess
```

### 11. Deshabilitar directory browsing

```apache
# .htaccess
Options -Indexes

# Si alguien accede a /wp-content/uploads/
# No verá listado de archivos
```

### 12. Proteger contra SQL injection

```php
// NUNCA hacer queries directas sin sanitizar
// MAL:
$user_input = $_GET['id'];
$results = $wpdb->get_results("SELECT * FROM wp_posts WHERE ID = $user_input");

// BIEN:
$user_input = absint($_GET['id']); // Sanitizar
$results = $wpdb->get_results($wpdb->prepare(
    "SELECT * FROM wp_posts WHERE ID = %d",
    $user_input
));
```

### 13. Proteger uploads directory

```apache
# wp-content/uploads/.htaccess
# Prevenir ejecución de PHP en uploads
<FilesMatch "\.(php|phtml|php3|php4|php5|pl|py|jsp|asp|html|htm|shtml|sh|cgi)$">
    deny from all
</FilesMatch>
```

### 14. WAF (Web Application Firewall)

Cloudflare Free tier:

```
Settings:
- SSL: Full (strict)
- Always Use HTTPS: On
- Automatic HTTPS Rewrites: On
- Security Level: Medium
- Challenge Passage: 30 minutes
```

Firewall rules:

```
(http.request.uri.path contains "/wp-login.php") and
(ip.geoip.country ne "ES") and
(ip.geoip.country ne "US")
→ Challenge

(http.request.uri.path contains "wp-admin") and
(not ip.src in {203.0.113.50})
→ Block
```

### 15. Two-Factor Authentication (2FA)

Plugin: **Two-Factor** o **Wordfence Login Security**

```php
// Forzar 2FA para administrators
add_filter('two_factor_required_roles', function($roles) {
    return ['administrator', 'editor'];
});
```

## Plugins de seguridad

### Wordfence

```php
// Configuración recomendada
- Firewall: Extended Protection
- Malware scan: Daily
- Login security: Enable 2FA
- Rate limiting: 20 requests/minute
```

### iThemes Security

```
Configuración:
- Strong passwords: Enforced
- File change detection: Enabled
- 404 detection: Block after 20/10 minutes
- Lockout whitelist: IP oficina
```

## Monitoring y alertas

```php
// functions.php - Alert en logins sospechosos
add_action('wp_login', 'alert_admin_on_login', 10, 2);

function alert_admin_on_login($user_login, $user) {
    if (in_array('administrator', $user->roles)) {
        $ip = $_SERVER['REMOTE_ADDR'];
        $time = current_time('mysql');

        wp_mail(
            'security@empresa.com',
            'Admin Login Alert',
            "Administrator $user_login logged in from IP $ip at $time"
        );
    }
}

// Alert en cambios de archivos core
add_action('admin_init', 'check_core_integrity');

function check_core_integrity() {
    $core_checksums = get_core_checksums(get_bloginfo('version'), 'en_US');

    foreach ($core_checksums as $file => $checksum) {
        $file_path = ABSPATH . $file;

        if (file_exists($file_path)) {
            $file_md5 = md5_file($file_path);

            if ($file_md5 !== $checksum) {
                // Archivo modificado!
                wp_mail(
                    'security@empresa.com',
                    'Core File Modified',
                    "WordPress core file modified: $file"
                );
            }
        }
    }
}
```

## Backups automáticos

```bash
#!/bin/bash
# /root/backup-wordpress.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/wordpress"
WP_DIR="/var/www/html"

# Backup files
tar -czf $BACKUP_DIR/files_$DATE.tar.gz $WP_DIR

# Backup database
mysqldump -u root -p'password' wordpress_db | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Subir a S3
aws s3 cp $BACKUP_DIR/files_$DATE.tar.gz s3://backups-empresa/wordpress/
aws s3 cp $BACKUP_DIR/db_$DATE.sql.gz s3://backups-empresa/wordpress/

# Borrar backups locales >7 días
find $BACKUP_DIR -type f -mtime +7 -delete
```

Cron job:

```bash
# crontab -e
0 2 * * * /root/backup-wordpress.sh
```

## Escaneo de malware

```bash
# Instalar ClamAV
sudo apt-get install clamav clamav-daemon

# Escanear WordPress
clamscan -r -i /var/www/html --log=/var/log/clamscan.log

# Cron diario
0 3 * * * clamscan -r -i /var/www/html --log=/var/log/clamscan_$(date +\%Y\%m\%d).log
```

## Headers de seguridad

```php
// functions.php
add_action('send_headers', 'add_security_headers');

function add_security_headers() {
    header('X-Frame-Options: SAMEORIGIN');
    header('X-Content-Type-Options: nosniff');
    header('X-XSS-Protection: 1; mode=block');
    header('Referrer-Policy: strict-origin-when-cross-origin');
    header("Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://www.google-analytics.com; style-src 'self' 'unsafe-inline';");
    header('Strict-Transport-Security: max-age=31536000; includeSubDomains');
}
```

## Incident response plan

Cuando detectas compromiso:

```bash
# 1. Modo mantenimiento inmediato
echo "<?php \$upgrading = time(); ?>" > .maintenance

# 2. Cambiar todas las passwords
# Via WP-CLI
wp user list --role=administrator --field=ID | xargs -I % wp user update % --user_pass=$(openssl rand -base64 24)

# 3. Regenerar security keys
# Copiar nuevas de api.wordpress.org/secret-key/ a wp-config.php

# 4. Escanear malware
clamscan -r -i /var/www/html

# 5. Revisar usuarios sospechosos
wp user list

# 6. Revisar archivos modificados recientemente
find /var/www/html -type f -mtime -1

# 7. Restaurar desde backup si es necesario
# 8. Remover .maintenance cuando esté limpio
```

## Resultados

**Antes del hardening**:
- 5000+ ataques/día
- 2 compromises en 6 meses
- Downtime: 4 horas/mes

**Después**:
- Ataques bloqueados automáticamente
- 0 compromises en 18+ meses
- Downtime: 0 horas
- Scanning diario, backups automáticos

## Checklist de seguridad WordPress

- [ ] HTTPS forzado
- [ ] Prefijo de DB cambiado
- [ ] File editing deshabilitado
- [ ] Login attempts limitados
- [ ] Versión de WP oculta
- [ ] XML-RPC deshabilitado
- [ ] wp-config.php protegido
- [ ] wp-admin restringido por IP
- [ ] Security keys generadas
- [ ] File permissions correctos
- [ ] Directory browsing deshabilitado
- [ ] Uploads directory protegido
- [ ] WAF configurado (Cloudflare)
- [ ] 2FA habilitado para admins
- [ ] Plugin de seguridad instalado
- [ ] Monitoring configurado
- [ ] Backups automáticos
- [ ] Escaneo de malware programado
- [ ] Security headers configurados
- [ ] Incident response plan documentado

¿Qué medidas de seguridad aplicas a tus sitios WordPress?
