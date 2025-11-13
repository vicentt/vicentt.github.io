---
layout: post
title: WooCommerce a escala - Optimizar tienda con 10,000 productos
author: Vicente José Moreno Escobar
categories: [WordPress, WooCommerce, Performance]
published: true
---

> Cómo convertir una tienda WooCommerce lenta en una máquina de ventas

# El problema

Cliente con tienda WooCommerce:
- 10,000 productos
- 500 pedidos/día
- Página de producto: **8 segundos** de carga
- Checkout: **12 segundos**

Resultado: 40% de carritos abandonados.

## Diagnóstico inicial

```bash
# Query Monitor plugin reveló el horror
- 247 queries en homepage
- 89 queries en página de producto
- Queries N+1 por todos lados
- 15 MB de CSS/JS sin minificar
```

## Optimización 1: Queries N+1

El problema clásico de WooCommerce:

```php
// Código problemático en el tema
foreach ($products as $product) {
    $product_obj = wc_get_product($product->ID); // Query!
    $categories = $product_obj->get_categories(); // Query!
    $image = $product_obj->get_image(); // Query!
}
```

**247 queries** para mostrar 20 productos.

### Solución: Pre-cargar todo

```php
// functions.php - Pre-cargar productos con relaciones
function optimize_product_query($query) {
    if (!is_admin() && $query->is_main_query() && is_shop()) {
        // Pre-cargar meta
        $query->set('meta_query', [
            'relation' => 'AND',
            [
                'key' => '_stock_status',
                'value' => 'instock'
            ]
        ]);

        // Pre-cargar imágenes
        $query->set('update_post_meta_cache', true);
        $query->set('update_post_term_cache', true);
    }
}
add_action('pre_get_posts', 'optimize_product_query');

// Cachear term meta (categorías, tags)
function cache_product_terms($posts) {
    if (empty($posts)) return $posts;

    $post_ids = wp_list_pluck($posts, 'ID');

    // Pre-cargar todos los terms de una vez
    $terms = wp_get_object_terms($post_ids, ['product_cat', 'product_tag'], [
        'fields' => 'all_with_object_id'
    ]);

    // Cachear en objeto transitorio
    foreach ($terms as $term) {
        wp_cache_add($term->object_id . '_terms', $term, 'product_terms');
    }

    return $posts;
}
add_filter('the_posts', 'cache_product_terms');
```

**Resultado**: 247 queries → 28 queries

## Optimización 2: Object Cache persistente

WooCommerce sin object cache es un desastre:

```bash
# Instalar Redis
sudo apt-get install redis-server php-redis
sudo systemctl enable redis-server
```

```php
// wp-config.php
define('WP_REDIS_HOST', 'localhost');
define('WP_REDIS_PORT', 6379);
define('WP_CACHE_KEY_SALT', 'tienda_');
define('WP_CACHE', true);

// Instalar plugin: Redis Object Cache
```

Cachear agresivamente:

```php
// Cachear productos en transients largos
function get_cached_product($product_id) {
    $cache_key = 'product_' . $product_id;
    $product = wp_cache_get($cache_key, 'products');

    if (false === $product) {
        $product = wc_get_product($product_id);
        wp_cache_set($cache_key, $product, 'products', DAY_IN_SECONDS);
    }

    return $product;
}

// Invalidar cache al actualizar producto
function invalidate_product_cache($product_id) {
    wp_cache_delete('product_' . $product_id, 'products');
}
add_action('woocommerce_update_product', 'invalidate_product_cache');
add_action('woocommerce_new_product', 'invalidate_product_cache');
```

**Resultado**: Tiempo de respuesta -60%

## Optimización 3: Lazy load de imágenes de productos

10,000 productos = muchas imágenes:

```php
// functions.php
function lazy_load_product_images($html, $post_id) {
    if (!is_admin() && get_post_type($post_id) === 'product') {
        // Reemplazar src con data-src
        $html = preg_replace(
            '/<img(.*)src=/i',
            '<img$1loading="lazy" src=',
            $html
        );
    }
    return $html;
}
add_filter('post_thumbnail_html', 'lazy_load_product_images', 10, 2);

// Usar thumbnails apropiados
add_image_size('product-thumbnail', 300, 300, true);
add_image_size('product-catalog', 600, 600, true);

// Servir imágenes optimizadas
function optimize_product_image_sizes($sizes, $size) {
    if (is_product()) {
        unset($sizes['medium_large']); // No necesario
        unset($sizes['large']); // No necesario
    }
    return $sizes;
}
add_filter('intermediate_image_sizes_advanced', 'optimize_product_image_sizes', 10, 2);
```

Compresión de imágenes con plugin:

```bash
# Instalar Imagify o ShortPixel
# Optimizar todas las imágenes existentes
wp media regenerate --yes
```

**Resultado**: Peso página -4.2MB

## Optimización 4: Deshabilitar funciones WooCommerce innecesarias

WooCommerce carga scripts en TODAS las páginas:

```php
// functions.php - Solo cargar scripts donde se necesitan
function disable_woocommerce_scripts() {
    // Deshabilitar en páginas que no son tienda
    if (!is_shop() && !is_product() && !is_cart() && !is_checkout()) {
        // Deshabilitar CSS
        wp_dequeue_style('woocommerce-general');
        wp_dequeue_style('woocommerce-layout');
        wp_dequeue_style('woocommerce-smallscreen');

        // Deshabilitar JS
        wp_dequeue_script('wc-cart-fragments');
        wp_dequeue_script('woocommerce');
        wp_dequeue_script('wc-add-to-cart');
    }
}
add_action('wp_enqueue_scripts', 'disable_woocommerce_scripts', 99);

// Deshabilitar heartbeat API (no necesario para tienda)
add_action('init', function() {
    wp_deregister_script('heartbeat');
}, 1);

// Deshabilitar embeds
function disable_embeds() {
    wp_deregister_script('wp-embed');
}
add_action('wp_footer', 'disable_embeds');
```

**Resultado**: -800KB de scripts innecesarios

## Optimización 5: Checkout optimizado

El checkout es crítico:

```php
// Simplificar checkout - menos campos
add_filter('woocommerce_checkout_fields', function($fields) {
    // Hacer opcionales campos no críticos
    unset($fields['billing']['billing_company']);
    unset($fields['billing']['billing_address_2']);
    unset($fields['order']['order_comments']);

    return $fields;
});

// AJAX checkout para no recargar página
add_action('wp_enqueue_scripts', function() {
    if (is_checkout()) {
        wp_enqueue_script('custom-checkout',
            get_template_directory_uri() . '/js/checkout.js',
            ['jquery', 'woocommerce'],
            '1.0',
            true
        );
    }
});
```

```javascript
// checkout.js - Actualizar totales sin recargar
jQuery(function($) {
    $('body').on('change', 'input[name="payment_method"]', function() {
        // Actualizar via AJAX en lugar de recargar página
        $.ajax({
            type: 'POST',
            url: wc_checkout_params.ajax_url,
            data: {
                action: 'update_order_review',
                payment_method: $(this).val()
            },
            success: function(response) {
                $('.order-total .amount').html(response.total);
            }
        });
    });
});
```

**Resultado**: Checkout 12s → 2.8s

## Optimización 6: Database optimizations

```sql
-- Índices en tablas WooCommerce
ALTER TABLE wp_postmeta
ADD INDEX meta_key_value (meta_key, meta_value(10));

ALTER TABLE wp_woocommerce_order_items
ADD INDEX order_id_type (order_id, order_item_type);

-- Limpiar data órfana
DELETE pm FROM wp_postmeta pm
LEFT JOIN wp_posts wp ON wp.ID = pm.post_id
WHERE wp.ID IS NULL;

-- Limpiar transients expirados
DELETE FROM wp_options
WHERE option_name LIKE '%_transient_%'
AND option_value < UNIX_TIMESTAMP();
```

Automatizar limpieza:

```php
// wp-config.php
define('WP_CRON_LOCK_TIMEOUT', 60);

// functions.php
add_action('wp_scheduled_delete', function() {
    global $wpdb;

    // Limpiar sessions viejas
    $wpdb->query("
        DELETE FROM {$wpdb->prefix}woocommerce_sessions
        WHERE session_expiry < UNIX_TIMESTAMP()
    ");

    // Limpiar carritos abandonados (>7 días)
    $wpdb->query("
        DELETE FROM {$wpdb->usermeta}
        WHERE meta_key = '_woocommerce_persistent_cart_1'
        AND meta_value < DATE_SUB(NOW(), INTERVAL 7 DAY)
    ");
});
```

**Resultado**: Queries -25% más rápidas

## Optimización 7: CDN para assets estáticos

```php
// wp-config.php
define('WP_CONTENT_URL', 'https://cdn.tienda.com/wp-content');

// O usar plugin: WP Offload Media
```

Configurar Cloudflare:

```
Page Rules:
- tienda.com/wp-content/* → Cache Level: Everything
- tienda.com/wp-includes/* → Cache Level: Everything
- tienda.com/product/* → Cache Level: Standard (HTML dinámico)
```

**Resultado**: TTFB -200ms

## Optimización 8: Queries personalizadas para productos

```php
// En lugar de usar WP_Query (lento), query directa
function get_featured_products_fast($limit = 10) {
    global $wpdb;

    $cache_key = 'featured_products_' . $limit;
    $products = wp_cache_get($cache_key);

    if (false === $products) {
        $products = $wpdb->get_results($wpdb->prepare("
            SELECT p.ID, p.post_title, pm1.meta_value as price, pm2.meta_value as image_id
            FROM {$wpdb->posts} p
            INNER JOIN {$wpdb->postmeta} pm1 ON (p.ID = pm1.post_id AND pm1.meta_key = '_price')
            INNER JOIN {$wpdb->postmeta} pm2 ON (p.ID = pm2.post_id AND pm2.meta_key = '_thumbnail_id')
            INNER JOIN {$wpdb->term_relationships} tr ON (p.ID = tr.object_id)
            INNER JOIN {$wpdb->term_taxonomy} tt ON (tr.term_taxonomy_id = tt.term_taxonomy_id)
            INNER JOIN {$wpdb->terms} t ON (tt.term_id = t.term_id AND t.slug = 'featured')
            WHERE p.post_type = 'product'
            AND p.post_status = 'publish'
            LIMIT %d
        ", $limit));

        wp_cache_set($cache_key, $products, '', HOUR_IN_SECONDS);
    }

    return $products;
}
```

**Resultado**: Query featured products 450ms → 12ms

## Métricas finales

**Antes**:
- Homepage: 8.2s
- Página producto: 8.0s
- Checkout: 12.4s
- 247 queries (homepage)
- Abandono carrito: 40%

**Después**:
- Homepage: 1.1s
- Página producto: 1.4s
- Checkout: 2.8s
- 28 queries (homepage)
- Abandono carrito: 18%

**Impacto negocio**:
- Conversión +35%
- Ventas +€50k/mes
- Satisfacción cliente mejorada notablemente

## Herramientas usadas

1. **Query Monitor**: Debugging queries
2. **Redis Object Cache**: Persistencia
3. **WP Rocket**: Page cache + minificación
4. **ShortPixel**: Optimización imágenes
5. **Cloudflare**: CDN + WAF
6. **New Relic**: Monitoreo APM

¿Tienes una tienda WooCommerce? ¿Qué optimizaciones te han funcionado mejor?
