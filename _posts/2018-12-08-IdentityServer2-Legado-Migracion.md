---
layout: post
title: IdentityServer 2 - Cuando el legado te atrapa y tienes que migrar
author: Vicente José Moreno Escobar
categories: [IdentityServer, OAuth2, .NET]
published: true
---

> La historia de cómo mantuvimos un sistema basado en IdentityServer 2 hasta 2018

# El contexto

En 2018 aún teníamos un sistema de autenticación basado en IdentityServer 2. Sí, la versión 2, lanzada en 2012. IdentityServer 3 ya llevaba años en el mercado, y la versión 4 estaba ganando tracción.

¿Por qué seguíamos con la versión 2? La clásica: "Si funciona, no lo toques".

## Por qué IdentityServer 2 era problemático en 2018

### 1. .NET Framework 4.0

IdentityServer 2 corría sobre .NET Framework 4.0. El resto de nuestras aplicaciones ya estaban en .NET 4.6.

```xml
<!-- IdentityServer 2 - Atrapado en el pasado -->
<targetFramework>net40</targetFramework>
```

Esto nos impedía usar características modernas de C# y .NET.

### 2. OAuth 2.0 incompleto

La especificación OAuth 2.0 estaba incompleta en IdentityServer 2:

```csharp
// Solo soportaba algunos grant types
public enum OAuth2GrantType
{
    AuthorizationCode,
    ResourceOwnerPassword,
    ClientCredentials
    // No soportaba Implicit flow moderno
    // No soportaba PKCE
}
```

### 3. Sin OpenID Connect

OpenID Connect no existía cuando se desarrolló IdentityServer 2. Teníamos que hacer workarounds para autenticación:

```csharp
// Nuestro hack para simular OIDC
public class PseudoOIDCClaims
{
    public static IEnumerable<Claim> GetUserClaims(string username)
    {
        var user = _userRepository.Find(username);

        return new[]
        {
            new Claim("sub", user.Id),
            new Claim("name", user.Name),
            new Claim("email", user.Email)
            // Simulando claims de OIDC manualmente
        };
    }
}
```

## Problemas que enfrentamos

### 1. No había updates de seguridad

IdentityServer 2 dejó de recibir updates en 2015. Cualquier vulnerabilidad descubierta quedaba sin parchear.

**Solución temporal**: WAF (Web Application Firewall) delante:

```
Internet → Azure Application Gateway (WAF) → IdentityServer 2
```

### 2. Integración con aplicaciones modernas

Las nuevas aplicaciones (SPAs, móviles) necesitaban flows que IdentityServer 2 no soportaba bien.

**Solución temporal**: Proxy middleware:

```csharp
// Middleware que traducía requests modernas a formato IS2
public class IdentityServerProxyMiddleware
{
    public async Task Invoke(HttpContext context)
    {
        // Si es un request PKCE, convertirlo a AuthorizationCode básico
        if (context.Request.Query.ContainsKey("code_challenge"))
        {
            var legacyRequest = ConvertPKCEToLegacy(context.Request);
            context.Request = legacyRequest;
        }

        await _next(context);
    }
}
```

### 3. Performance degradada

Con el tiempo, IdentityServer 2 mostraba problemas de performance bajo carga:

```
2018-12-01 14:23:15 - Token endpoint: 1200ms
2018-12-01 14:24:32 - Token endpoint: 1450ms
2018-12-01 14:25:01 - Token endpoint: 2100ms (!)
```

No había profiler moderno que nos ayudara a diagnosticar.

## La decisión de migrar

Después de un incidente de seguridad (un CVE en una dependencia sin patch), el management finalmente aprobó la migración.

### Plan de migración

**Fase 1**: Dual running (4 semanas)
- IdentityServer 2: Producción
- IdentityServer 3: Staging/Testing

**Fase 2**: Migración gradual (8 semanas)
- App por app, migrar a IS3
- Rollback automático si >5% errores

**Fase 3**: Deprecación IS2 (2 semanas)
- Monitoreo intensivo
- Apagado gradual

### Retos de la migración

#### 1. Base de datos incompatible

El schema de IdentityServer 2 era completamente diferente:

```sql
-- IdentityServer 2
CREATE TABLE Clients (
    Id INT PRIMARY KEY,
    ClientId NVARCHAR(100),
    ClientSecret NVARCHAR(100),
    -- Estructura antigua
)

-- IdentityServer 3 esperaba un schema diferente
CREATE TABLE Clients (
    Id INT PRIMARY KEY,
    ClientId NVARCHAR(200),
    ClientSecrets NVARCHAR(MAX), -- JSON array
    -- Muchos más campos
)
```

**Solución**: Script de migración custom:

```csharp
public class IS2ToIS3Migrator
{
    public async Task MigrateClients()
    {
        var is2Clients = await _is2Db.Clients.ToListAsync();

        foreach (var oldClient in is2Clients)
        {
            var newClient = new IS3Client
            {
                ClientId = oldClient.ClientId,
                ClientSecrets = new List<Secret>
                {
                    new Secret(oldClient.ClientSecret.Sha256())
                },
                AllowedGrantTypes = ConvertGrantTypes(oldClient.Flow),
                // Mapeo manual de cada campo
            };

            await _is3Db.Clients.AddAsync(newClient);
        }

        await _is3Db.SaveChangesAsync();
    }
}
```

#### 2. Tokens activos

Teníamos ~5000 tokens activos en cualquier momento. Invalidarlos todos habría forzado re-login masivo.

**Solución**: Validador híbrido:

```csharp
public class HybridTokenValidator : ITokenValidator
{
    public async Task<TokenValidationResult> ValidateAccessTokenAsync(string token)
    {
        // Intentar validar con IS3 primero
        var is3Result = await _is3Validator.ValidateAccessTokenAsync(token);
        if (is3Result.IsValid) return is3Result;

        // Si falla, intentar con IS2
        var is2Result = await _is2Validator.ValidateAccessTokenAsync(token);
        if (is2Result.IsValid)
        {
            // Token válido de IS2, pero loggear para monitoreo
            _logger.LogWarning("Token IS2 aún en uso: {ClientId}", is2Result.ClientId);
            return is2Result;
        }

        return TokenValidationResult.Invalid;
    }
}
```

## Resultados post-migración

**Antes (IdentityServer 2)**:
- Performance: 1200ms promedio en token endpoint
- Seguridad: Sin updates desde 2015
- Compatibilidad: Hacks para aplicaciones modernas
- Mantenibilidad: Imposible añadir features nuevas

**Después (IdentityServer 3)**:
- Performance: 180ms promedio en token endpoint
- Seguridad: Updates regulares
- Compatibilidad: Full OAuth 2.0 + OpenID Connect
- Mantenibilidad: Código moderno, extensible

## Lecciones aprendidas

1. **El costo de posponer migraciones crece exponencialmente**: Cada mes que pasaba, la deuda técnica crecía
2. **"Si funciona, no lo toques" es peligroso en seguridad**: Las vulnerabilidades no perdonan
3. **Dual running es crítico**: Nos salvó múltiples veces durante la migración
4. **Scripts de migración necesitan testing exhaustivo**: Migramos 50,000 usuarios sin perder datos gracias a testing riguroso

¿Has tenido que migrar sistemas de autenticación legacy? ¿Cuánto tiempo estuviste postergándolo?
