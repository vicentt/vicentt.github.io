---
layout: post
title: WordPress como Headless CMS - Cuando el frontend necesita ser React
author: Vicente José Moreno Escobar
categories: [WordPress, React, APIs]
published: true
---

> Desacoplando WordPress: backend potente + frontend moderno

# El proyecto

Cliente con sitio WordPress exitoso quería:
- Frontend moderno en React para mejor performance
- Mantener WordPress como CMS (editores lo conocían bien)
- SEO impecable
- Deployment independiente de frontend y backend

Solución: **WordPress Headless** con Next.js frontend.

## Arquitectura

```
WordPress (Backend)
    ↓ (REST API / GraphQL)
Next.js (Frontend)
    ↓
Usuario
```

WordPress solo maneja contenido. Next.js renderiza todo.

## Configuración WordPress

### 1. Habilitar REST API

WordPress ya trae REST API, pero necesitábamos customizarla:

```php
// functions.php

// Añadir CORS headers para permitir requests desde Next.js
add_action('rest_api_init', function() {
    remove_filter('rest_pre_serve_request', 'rest_send_cors_headers');
    add_filter('rest_pre_serve_request', function($value) {
        header('Access-Control-Allow-Origin: https://www.tusitio.com');
        header('Access-Control-Allow-Methods: GET, POST, OPTIONS');
        header('Access-Control-Allow-Credentials: true');
        return $value;
    });
}, 15);

// Exponer ACF fields en REST API
add_action('rest_api_init', function() {
    if (function_exists('acf')) {
        foreach (get_post_types(['public' => true]) as $post_type) {
            register_rest_field($post_type, 'acf', [
                'get_callback' => function($post) {
                    return get_fields($post['id']);
                },
                'schema' => null,
            ]);
        }
    }
});

// Custom endpoint para menús
add_action('rest_api_init', function() {
    register_rest_route('wp/v2', '/menus/(?P<id>[a-zA-Z0-9_-]+)', [
        'methods' => 'GET',
        'callback' => function($request) {
            $menu = wp_get_nav_menu_items($request['id']);
            return $menu ?: new WP_Error('no_menu', 'Menu not found', ['status' => 404]);
        },
    ]);
});

// Custom endpoint para opciones globales
add_action('rest_api_init', function() {
    register_rest_route('wp/v2', '/options', [
        'methods' => 'GET',
        'callback' => function() {
            return [
                'site_name' => get_bloginfo('name'),
                'site_description' => get_bloginfo('description'),
                'footer_text' => get_option('footer_text'),
                'social_links' => get_option('social_links'),
            ];
        },
    ]);
});
```

### 2. ACF para campos custom

Advanced Custom Fields para añadir metadata rica:

```php
// Ejemplo: Post con campos adicionales
if (function_exists('acf_add_local_field_group')) {
    acf_add_local_field_group([
        'key' => 'group_post_extra',
        'title' => 'Post Extra Fields',
        'fields' => [
            [
                'key' => 'field_featured',
                'label' => 'Featured Post',
                'name' => 'featured',
                'type' => 'true_false',
            ],
            [
                'key' => 'field_author_bio',
                'label' => 'Custom Author Bio',
                'name' => 'author_bio',
                'type' => 'textarea',
            ],
            [
                'key' => 'field_related_posts',
                'label' => 'Related Posts',
                'name' => 'related_posts',
                'type' => 'relationship',
                'post_type' => ['post'],
            ],
        ],
        'location' => [
            [
                [
                    'param' => 'post_type',
                    'operator' => '==',
                    'value' => 'post',
                ],
            ],
        ],
    ]);
}
```

### 3. Autenticación para preview/draft

JWT Auth plugin para previews:

```bash
# Instalar plugin
wp plugin install jwt-authentication-for-wp-rest-api --activate
```

```php
// wp-config.php
define('JWT_AUTH_SECRET_KEY', 'tu-secret-key-super-segura');
define('JWT_AUTH_CORS_ENABLE', true);
```

```php
// functions.php - Endpoint de preview autenticado
add_action('rest_api_init', function() {
    register_rest_route('wp/v2', '/preview/(?P<id>\d+)', [
        'methods' => 'GET',
        'callback' => function($request) {
            $post = get_post($request['id']);

            if (!$post || $post->post_status !== 'draft') {
                return new WP_Error('no_post', 'Draft not found', ['status' => 404]);
            }

            return [
                'id' => $post->ID,
                'title' => $post->post_title,
                'content' => apply_filters('the_content', $post->post_content),
                'acf' => get_fields($post->ID),
            ];
        },
        'permission_callback' => function() {
            return current_user_can('edit_posts');
        },
    ]);
});
```

## Frontend Next.js

### 1. Fetch de posts

```typescript
// lib/api.ts
const WP_API_URL = process.env.WORDPRESS_API_URL;

export async function getAllPosts() {
  const res = await fetch(`${WP_API_URL}/wp/v2/posts?_embed&per_page=100`);

  if (!res.ok) {
    throw new Error('Failed to fetch posts');
  }

  return res.json();
}

export async function getPostBySlug(slug: string) {
  const res = await fetch(`${WP_API_URL}/wp/v2/posts?slug=${slug}&_embed`);
  const posts = await res.json();

  return posts[0];
}

export async function getAllPostSlugs() {
  const posts = await getAllPosts();
  return posts.map((post: any) => ({
    params: {
      slug: post.slug,
    },
  }));
}
```

### 2. Páginas estáticas con ISR

```typescript
// pages/blog/[slug].tsx
import { GetStaticPaths, GetStaticProps } from 'next';
import { getAllPostSlugs, getPostBySlug } from '../../lib/api';

interface Post {
  id: number;
  title: { rendered: string };
  content: { rendered: string };
  date: string;
  acf: {
    featured: boolean;
    author_bio: string;
  };
  _embedded: {
    'wp:featuredmedia': Array<{
      source_url: string;
    }>;
  };
}

export default function BlogPost({ post }: { post: Post }) {
  return (
    <article>
      <h1 dangerouslySetInnerHTML={{ __html: post.title.rendered }} />

      {post._embedded?.['wp:featuredmedia']?.[0] && (
        <img
          src={post._embedded['wp:featuredmedia'][0].source_url}
          alt={post.title.rendered}
        />
      )}

      <div dangerouslySetInnerHTML={{ __html: post.content.rendered }} />

      {post.acf?.author_bio && (
        <div className="author-bio">
          <p>{post.acf.author_bio}</p>
        </div>
      )}
    </article>
  );
}

export const getStaticPaths: GetStaticPaths = async () => {
  const paths = await getAllPostSlugs();

  return {
    paths,
    fallback: 'blocking', // ISR: generar páginas on-demand
  };
};

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const post = await getPostBySlug(params!.slug as string);

  if (!post) {
    return {
      notFound: true,
    };
  }

  return {
    props: {
      post,
    },
    revalidate: 60, // Re-generar cada 60 segundos si hay tráfico
  };
};
```

### 3. Preview mode

```typescript
// pages/api/preview.ts
import { NextApiRequest, NextApiResponse } from 'next';

export default async function preview(req: NextApiRequest, res: NextApiResponse) {
  const { secret, id, token } = req.query;

  // Verificar secret
  if (secret !== process.env.PREVIEW_SECRET) {
    return res.status(401).json({ message: 'Invalid secret' });
  }

  // Fetch del draft con JWT token
  const wpRes = await fetch(
    `${process.env.WORDPRESS_API_URL}/wp/v2/preview/${id}`,
    {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    }
  );

  if (!wpRes.ok) {
    return res.status(401).json({ message: 'Invalid token' });
  }

  const post = await wpRes.json();

  // Enable Preview Mode
  res.setPreviewData({});

  // Redirect to the path of the post
  res.redirect(`/blog/${post.slug}`);
}
```

Modificar `getStaticProps` para preview:

```typescript
export const getStaticProps: GetStaticProps = async ({ params, preview = false, previewData }) => {
  let post;

  if (preview) {
    // Fetch draft version
    post = await getPreviewPost(params!.slug as string);
  } else {
    post = await getPostBySlug(params!.slug as string);
  }

  // ...
};
```

### 4. Revalidación on-demand

WordPress dispara webhook cuando se publica post:

```php
// functions.php
add_action('publish_post', function($post_id) {
    $post = get_post($post_id);

    // Trigger revalidation en Next.js
    wp_remote_post(getenv('NEXTJS_REVALIDATE_URL'), [
        'body' => json_encode([
            'secret' => getenv('REVALIDATE_SECRET'),
            'slug' => $post->post_name,
        ]),
        'headers' => ['Content-Type' => 'application/json'],
    ]);
});
```

Next.js endpoint:

```typescript
// pages/api/revalidate.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { secret, slug } = req.body;

  if (secret !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: 'Invalid secret' });
  }

  try {
    await res.revalidate(`/blog/${slug}`);
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send('Error revalidating');
  }
}
```

## Alternativa: WPGraphQL

Para queries más eficientes, usar GraphQL en lugar de REST:

```bash
# Instalar plugin
wp plugin install wp-graphql --activate
```

```typescript
// lib/api.ts con GraphQL
import { GraphQLClient, gql } from 'graphql-request';

const client = new GraphQLClient(`${process.env.WORDPRESS_API_URL}/graphql`);

export async function getAllPosts() {
  const query = gql`
    query AllPosts {
      posts {
        nodes {
          id
          slug
          title
          content
          date
          featuredImage {
            node {
              sourceUrl
            }
          }
          acf {
            featured
            authorBio
          }
        }
      }
    }
  `;

  const data = await client.request(query);
  return data.posts.nodes;
}

export async function getPostBySlug(slug: string) {
  const query = gql`
    query PostBySlug($slug: ID!) {
      post(id: $slug, idType: SLUG) {
        id
        title
        content
        date
        featuredImage {
          node {
            sourceUrl
          }
        }
        acf {
          featured
          authorBio
        }
      }
    }
  `;

  const data = await client.request(query, { slug });
  return data.post;
}
```

**Ventajas GraphQL**:
- Queries precisas (solo pedir lo que necesitas)
- Menos over-fetching
- Mejor performance

## Optimizaciones

### 1. Caché de imágenes

```typescript
// next.config.js
module.exports = {
  images: {
    domains: ['tu-wordpress-backend.com'],
    formats: ['image/avif', 'image/webp'],
  },
};
```

```tsx
import Image from 'next/image';

<Image
  src={post.featuredImage.node.sourceUrl}
  alt={post.title}
  width={800}
  height={600}
  priority
/>
```

### 2. CDN para WordPress media

```php
// wp-config.php
define('WP_CONTENT_URL', 'https://cdn.tusitio.com/wp-content');
```

## Resultados

**Antes (WordPress tradicional)**:
- Tiempo de carga: 3.2s
- Lighthouse: 62/100
- Limitado a PHP/WordPress stack

**Después (Headless)**:
- Tiempo de carga: 0.8s
- Lighthouse: 98/100
- ISR permite actualización sin rebuild completo
- Frontend moderno con React/Next.js
- Backend que editores ya conocen

¿Has usado WordPress como headless CMS? ¿Qué stack de frontend prefieres?
