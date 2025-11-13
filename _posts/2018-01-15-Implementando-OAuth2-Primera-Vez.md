---
layout: post
title: Implementando OAuth 2.0 por primera vez - Lecciones aprendidas
author: Vicente José Moreno Escobar
categories: [OAuth2, Seguridad, APIs]
published: true
---

> Mi primera experiencia implementando OAuth 2.0 en un proyecto real y los errores que cometí

# El proyecto 🎯

A principios de 2018 nos enfrentamos a un reto: teníamos que exponer nuestras APIs a terceros, pero de forma segura. Hasta entonces solo usábamos API Keys básicas, lo cual claramente no era suficiente.

La decisión fue implementar OAuth 2.0. Suena simple, pero fue más complicado de lo que pensaba.

## Error #1: No entender los flujos

OAuth 2.0 tiene varios "flows" (flujos de autorización):
- Authorization Code
- Implicit
- Client Credentials
- Resource Owner Password Credentials

Mi primer error fue pensar "voy a usar el más simple". Elegí Implicit Flow porque tenía menos pasos. Mal.

**Lo que aprendí:** Cada flujo tiene su propósito:
- **Authorization Code**: Para aplicaciones web tradicionales
- **Client Credentials**: Para comunicación máquina a máquina
- **Implicit**: Solo para SPAs (y ahora ya no se recomienda)

Para nuestro caso (APIs consumidas por aplicaciones de terceros), Client Credentials era lo correcto.

## Error #2: Confundir autenticación con autorización

OAuth 2.0 es para **autorización**, no autenticación. Yo mezclaba ambos conceptos.

```csharp
// Mal enfoque - mezclando conceptos
[Authorize]
public IActionResult GetUserData()
{
    // ¿Quién es el usuario? OAuth no me lo dice directamente
    var userId = User.Identity.Name; // Esto no funciona como esperaba
    return Ok(_service.GetData(userId));
}
```

**Lo que aprendí:**
- OAuth 2.0 → Autorización (¿qué puede hacer?)
- OpenID Connect → Autenticación (¿quién es?)

Si necesitas saber quién es el usuario, necesitas OpenID Connect además de OAuth 2.0.

## Error #3: Tokens sin expiración o con expiración muy larga

En mis primeras pruebas, configuré tokens que duraban 30 días. Parecía conveniente para testing.

```csharp
options.AccessTokenLifetime = 2592000; // 30 días - TERRIBLE IDEA
```

**Lo que aprendí:** Los access tokens deben ser de vida corta (15-60 minutos). Los refresh tokens son para sesiones largas.

```csharp
options.AccessTokenLifetime = 3600; // 1 hora
options.RefreshTokenLifetime = 2592000; // 30 días para refresh
```

## Error #4: No validar correctamente los scopes

Implementé scopes en el servidor, pero no los validaba correctamente en la API:

```csharp
[Authorize] // Solo valida que haya token, no el scope
public IActionResult DeleteUser(int id)
{
    _service.DeleteUser(id);
    return Ok();
}
```

**Lo que aprendí:** Hay que validar los scopes explícitamente:

```csharp
[Authorize]
[RequiredScope("users.delete")]
public IActionResult DeleteUser(int id)
{
    _service.DeleteUser(id);
    return Ok();
}
```

## La implementación que funcionó

Después de varios intentos, esta es la estructura que nos funcionó:

### 1. Authorization Server (IdentityServer 2)

```csharp
new Client
{
    ClientId = "mobile-app",
    ClientSecrets = { new Secret("secret".Sha256()) },
    AllowedGrantTypes = GrantTypes.ClientCredentials,
    AllowedScopes = { "api.read", "api.write" },
    AccessTokenLifetime = 3600
}
```

### 2. API Protection

```csharp
services.AddAuthentication("Bearer")
    .AddIdentityServerAuthentication(options =>
    {
        options.Authority = "https://auth.midominio.com";
        options.RequireHttpsMetadata = true;
        options.ApiName = "mi-api";
    });
```

### 3. Scope Validation

```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("ReadAccess", policy =>
        policy.RequireClaim("scope", "api.read"));

    options.AddPolicy("WriteAccess", policy =>
        policy.RequireClaim("scope", "api.write"));
});
```

## Consejos para quien empieza con OAuth 2.0

1. **Lee la RFC 6749 completa**: Sí, es densa, pero vale la pena
2. **Usa HTTPS siempre**: OAuth sin HTTPS es inútil
3. **Empieza con Client Credentials**: Es el flujo más simple para entender los conceptos
4. **Prueba con Postman**: Te ayuda a entender qué pasa en cada paso
5. **No reinventes la rueda**: Usa librerías maduras (IdentityServer, Auth0, etc.)

## Herramientas útiles

- **jwt.io**: Para debuggear tokens JWT
- **Postman**: Para probar los flows
- **IdentityServer**: Para implementar un servidor OAuth rápido

## Conclusión

OAuth 2.0 tiene una curva de aprendizaje pronunciada, pero una vez lo entiendes, es tremendamente útil. Los errores que cometí me enseñaron más que cualquier tutorial.

Si estás empezando con OAuth 2.0, ten paciencia. Los conceptos se van aclarando con la práctica.

¿Has implementado OAuth 2.0? ¿Qué errores cometiste tú?
