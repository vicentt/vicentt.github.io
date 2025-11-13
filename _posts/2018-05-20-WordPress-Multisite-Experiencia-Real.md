---
layout: post
title: WordPress Multisite - Cuando un cliente quiere 12 sitios en uno
author: Vicente José Moreno Escobar
categories: [WordPress, PHP, Arquitectura]
published: true
---

> La experiencia de gestionar 12 sitios WordPress como si fueran uno solo

# El reto 📋

Un cliente con presencia en 12 países nos pidió: "Queremos un sitio web por país, pero que podamos gestionar todo desde un solo lugar".

Opciones:
1. **12 instalaciones WordPress separadas**: Pesadilla de mantenimiento
2. **Un sitio multiidioma**: No servía, cada país quería personalización
3. **WordPress Multisite**: La opción que elegimos

Tres años después, puedo decir que fue la decisión correcta (con matices).

## ¿Qué es WordPress Multisite?

WordPress Multisite (WPMU) te permite gestionar múltiples sitios WordPress desde una sola instalación:
- Una base de datos
- Un conjunto de archivos PHP
- Pero N sitios independientes

Piensa en ello como "hosting compartido dentro de WordPress".

## La arquitectura que implementamos

### Estructura de URLs

Teníamos dos opciones:
- **Subdominios**: spain.cliente.com, france.cliente.com
- **Subdirectorios**: cliente.com/spain, cliente.com/france

Elegimos subdominios porque cada país quería sentir que tenía "su" sitio.

### wp-config.php

```php
define('WP_ALLOW_MULTISITE', true);
define('MULTISITE', true);
define('SUBDOMAIN_INSTALL', true);
define('DOMAIN_CURRENT_SITE', 'cliente.com');
define('PATH_CURRENT_SITE', '/');
define('SITE_ID_CURRENT_SITE', 1);
define('BLOG_ID_CURRENT_SITE', 1);
```

### .htaccess

```apache
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]

# add a trailing slash to /wp-admin
RewriteRule ^([_0-9a-zA-Z-]+/)?wp-admin$ $1wp-admin/ [R=301,L]

RewriteCond %{REQUEST_FILENAME} -f [OR]
RewriteCond %{REQUEST_FILENAME} -d
RewriteRule ^ - [L]
RewriteRule ^([_0-9a-zA-Z-]+/)?(wp-(content|admin|includes).*) $2 [L]
RewriteRule ^([_0-9a-zA-Z-]+/)?(.*\.php)$ $2 [L]
RewriteRule . index.php [L]
```

## Desafíos encontrados

### 1. Plugins que no son "multisite-aware"

Algunos plugins no funcionaban bien en multisite. Por ejemplo, un plugin de caché guardaba configuración global afectando a todos los sitios.

**Solución**: Crear un "must-use plugin" que forzaba configuración por sitio:

```php
// wp-content/mu-plugins/multisite-fixes.php
add_filter('plugin_action_links', function($links, $file) {
    if (is_network_admin()) {
        // Deshabilitar ciertos plugins en network admin
        if (in_array($file, ['problematic-plugin/plugin.php'])) {
            unset($links['activate']);
        }
    }
    return $links;
}, 10, 2);
```

### 2. Temas y personalizaciones

Cada país quería su tema personalizado, pero con elementos comunes (header, footer).

**Solución**: Tema padre común + child themes por país:

```
wp-content/themes/
├── cliente-base/           # Tema padre
├── cliente-spain/          # Child theme España
├── cliente-france/         # Child theme Francia
└── ...
```

Child theme mínimo:

```php
// style.css
/*
Theme Name: Cliente Spain
Template: cliente-base
*/

// functions.php
<?php
add_action('wp_enqueue_scripts', function() {
    wp_enqueue_style('parent-style',
        get_template_directory_uri() . '/style.css');
    wp_enqueue_style('child-style',
        get_stylesheet_directory_uri() . '/style.css',
        array('parent-style')
    );
});
```

### 3. Media library compartida vs. separada

Por defecto, cada sitio tiene su propia librería de medios. Pero teníamos imágenes corporativas que todos los sitios necesitaban.

**Solución**: Plugin "Multisite Global Media":

```php
// Permite seleccionar imágenes del sitio principal desde cualquier sitio
add_action('admin_menu', function() {
    add_media_page(
        'Global Media',
        'Global Media',
        'upload_files',
        'global-media',
        'render_global_media_page'
    );
});
```

### 4. Backups complejos

Con 12 sitios, los backups eran enormes. No podíamos hacer backup completo cada noche.

**Solución**: Backup incremental por sitio:

```bash
#!/bin/bash
# Backup individual por sitio
for SITE_ID in {1..12}
do
  wp db export \
    --path=/var/www/html \
    --url=site-${SITE_ID}.cliente.com \
    site-${SITE_ID}-$(date +%Y%m%d).sql

  # Solo subir a S3 si cambió algo
  if ! diff site-${SITE_ID}-$(date +%Y%m%d).sql backup-anterior.sql > /dev/null
  then
    aws s3 cp site-${SITE_ID}-$(date +%Y%m%d).sql s3://backups/
  fi
done
```

### 5. Performance con 12 sitios

Con un solo sitio, WordPress vuela. Con 12, no tanto.

**Optimizaciones aplicadas**:

1. **Object cache compartido** (Redis):

```php
// wp-config.php
define('WP_REDIS_DATABASE', 0);
define('WP_CACHE_KEY_SALT', 'cliente_');
```

2. **CDN para assets estáticos**:

```php
define('WP_CONTENT_URL', 'https://cdn.cliente.com/wp-content');
```

3. **Lazy loading de sitios no activos**:

```php
// Solo cargar configuración del sitio actual
add_filter('ms_site_check', function($site_id) {
    if ($site_id !== get_current_blog_id()) {
        return false; // No cargar otros sitios
    }
    return true;
});
```

## Plugins indispensables para Multisite

1. **WP CLI**: Automatización salvaje
2. **Multisite Enhancements**: Mejoras en el Network Admin
3. **Multisite Plugin Manager**: Control granular de plugins por sitio
4. **Multisite User Management**: Gestión centralizada de usuarios

## Gestión diaria

### Crear nuevo sitio (automatizado)

```bash
wp site create \
  --slug=germany \
  --title="Cliente Germany" \
  --email=admin@cliente.com \
  --network-id=1

wp theme activate cliente-germany --url=germany.cliente.com
wp plugin activate woocommerce --url=germany.cliente.com
```

### Actualizar todos los sitios

```bash
# Actualizar WordPress core en todos
wp core update --network

# Actualizar plugins en todos los sitios
wp plugin update --all --network

# O site por site si da problemas
for site in $(wp site list --field=url)
do
  wp plugin update --all --url=$site
done
```

## Cuando NO usar Multisite

Después de 3 años, aprendí cuándo multisite NO es buena idea:

❌ Los sitios tienen requisitos MUY diferentes
❌ Necesitas aislar completamente los sitios (seguridad)
❌ Los sitios tienen tráfico extremadamente variable
❌ Vas a revender hosting (mejor WP tradicional por cliente)

✅ Control centralizado es prioridad
✅ Sitios con estructura similar
✅ Equipo técnico puede gestionar la complejidad
✅ Ahorro en costes de hosting importa

## Resultados después de 3 años

**Pros reales**:
- Actualizar 12 sitios en 5 minutos vs. 2 horas
- Un solo punto de backup
- Hosting: $100/mes vs. $800/mes (12 sitios separados)
- Gestión de usuarios centralizada

**Contras reales**:
- Depuración más compleja
- Algunos plugins simplemente no funcionan
- Si un sitio tiene un bug grave, puede afectar a otros
- Onboarding de desarrolladores nuevos toma más tiempo

## Consejos para quien empiece con Multisite

1. **Empieza pequeño**: No migres 12 sitios de golpe. Empieza con 2-3
2. **Documenta TODO**: Los problemas serán únicos, necesitas saber qué hiciste
3. **WP CLI es tu amigo**: Aprende a usarlo antes de escalar
4. **Ten un entorno de staging**: NO pruebes en producción con multisite
5. **Monitoriza por sitio**: New Relic o similar que pueda segmentar por subdominio

## Conclusión

WordPress Multisite no es para todos, pero para nuestro caso (12 sitios similares, gestión centralizada) fue perfecto.

¿Usas o has usado Multisite? ¿Qué desafíos encontraste?
