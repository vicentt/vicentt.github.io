---
layout: post
title: Migrar a IdentityServer 3 en producción sin morir en el intento
author: Vicente José Moreno Escobar
categories: [IdentityServer, OAuth2, .NET]
published: true
---

> La experiencia de migrar nuestro sistema de autenticación a IdentityServer 3 con 10,000 usuarios activos

# El contexto

Teníamos un sistema de autenticación custom que funcionaba... más o menos. Con el crecimiento, necesitábamos algo más robusto y mantenible.

IdentityServer 3 era la opción obvia en 2018 para .NET Framework.

## Desafíos principales

### 1. Migración de usuarios sin forzar reset de passwords

Nuestro hash de passwords era diferente al de IdentityServer:

```csharp
// Nuestro sistema legacy
var hash = SHA256(password + salt);

// IdentityServer esperaba
var hash = BCrypt o PBKDF2
```

**Solución**: Validación híbrida durante transición:

```csharp
public class HybridPasswordValidator : IPasswordValidator
{
    public async Task<bool> ValidateAsync(string username, string password)
    {
        var user = await _userStore.FindByUsernameAsync(username);

        // Si tiene hash legacy
        if (user.UsesLegacyHash)
        {
            if (ValidateLegacyHash(password, user.PasswordHash))
            {
                // Migrar a nuevo hash en background
                await MigrateToNewHashAsync(user, password);
                return true;
            }
        }

        // Validación normal IdentityServer
        return await _passwordHasher.VerifyHashedPassword(
            user, user.PasswordHash, password);
    }
}
```

### 2. Configuración de clientes

Teníamos 8 aplicaciones diferentes consumiendo nuestro auth. Cada una necesitaba configuración específica:

```csharp
new Client
{
    ClientId = "webapp-cliente",
    ClientName = "Portal Web Cliente",
    ClientSecrets = new List<Secret>
    {
        new Secret("secreto".Sha256())
    },

    AllowedGrantTypes = GrantTypes.Code,
    RequirePkce = true,

    RedirectUris = new List<string>
    {
        "https://webapp.cliente.com/signin-oidc"
    },

    PostLogoutRedirectUris = new List<string>
    {
        "https://webapp.cliente.com/signout-callback-oidc"
    },

    AllowedScopes = new List<string>
    {
        IdentityServerConstants.StandardScopes.OpenId,
        IdentityServerConstants.StandardScopes.Profile,
        "api.cliente"
    },

    AccessTokenLifetime = 3600,
    IdentityTokenLifetime = 300
}
```

### 3. Custom Claims

Nuestros usuarios tenían claims personalizados que las aplicaciones dependían:

```csharp
public class CustomProfileService : IProfileService
{
    public async Task GetProfileDataAsync(ProfileDataRequestContext context)
    {
        var user = await _userManager.FindByIdAsync(context.Subject.GetSubjectId());

        var claims = new List<Claim>
        {
            new Claim("employee_id", user.EmployeeId),
            new Claim("department", user.Department),
            new Claim("cost_center", user.CostCenter)
        };

        // Añadir roles como claims
        var roles = await _userManager.GetRolesAsync(user);
        claims.AddRange(roles.Select(role => new Claim("role", role)));

        context.IssuedClaims = claims;
    }

    public async Task IsActiveAsync(IsActiveContext context)
    {
        var user = await _userManager.FindByIdAsync(context.Subject.GetSubjectId());
        context.IsActive = user != null && user.IsActive;
    }
}
```

## Estrategia de migración

### Fase 1: Dual running (2 semanas)

Ambos sistemas corriendo en paralelo:
- Sistema legacy: producción
- IdentityServer 3: staging con datos reales

Ejecutamos pruebas de carga:

```csharp
// Test de carga con NBomber
var scenario = ScenarioBuilder
    .CreateScenario("login_test", async context =>
    {
        var response = await _httpClient.PostAsync(
            "https://ids-staging.cliente.com/connect/token",
            new FormUrlEncodedContent(new[]
            {
                new KeyValuePair<string, string>("grant_type", "password"),
                new KeyValuePair<string, string>("username", "testuser"),
                new KeyValuePair<string, string>("password", "testpass"),
                new KeyValuePair<string, string>("client_id", "test-client")
            }));

        return response.IsSuccessStatusCode
            ? Response.Ok()
            : Response.Fail();
    })
    .WithWarmUpDuration(TimeSpan.FromSeconds(10))
    .WithLoadSimulations(
        Simulation.KeepConstant(copies: 100, during: TimeSpan.FromMinutes(5))
    );
```

### Fase 2: Migración por aplicación (4 semanas)

No migramos todo de golpe. Una aplicación por semana:

**Semana 1**: App menos crítica (admin interno)
**Semana 2**: Portal clientes (crítico pero bajo tráfico)
**Semana 3**: API móvil (tráfico medio)
**Semana 4**: Portal principal (alto tráfico)

### Fase 3: Apagar sistema legacy

Después de 1 mes con todas las apps en IdentityServer:
- Monitoreamos logs buscando errores
- Verificamos que nadie usaba endpoints antiguos
- Apagamos gradualmente

## Problemas encontrados

### 1. Session management

IdentityServer no persistía sesiones por defecto. Con múltiples servidores, era problemático:

```csharp
// Configurar session store distribuido
services.AddIdentityServer()
    .AddOperationalStore(options =>
    {
        options.ConfigureDbContext = builder =>
            builder.UseSqlServer(connectionString);

        options.EnableTokenCleanup = true;
        options.TokenCleanupInterval = 3600;
    });
```

### 2. CORS para SPAs

Nuestras SPAs necesitaban CORS configurado correctamente:

```csharp
services.AddCors(options =>
{
    options.AddPolicy("AllowSPAs", builder =>
    {
        builder
            .WithOrigins(
                "https://app1.cliente.com",
                "https://app2.cliente.com"
            )
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials();
    });
});
```

### 3. Refresh tokens no funcionaban inicialmente

Los refresh tokens se invalidaban al reiniciar el servidor:

```csharp
// Solución: persistir en DB
new Client
{
    ClientId = "mobile-app",
    RefreshTokenUsage = TokenUsage.ReUse, // o OneTimeOnly
    RefreshTokenExpiration = TokenExpiration.Sliding,
    SlidingRefreshTokenLifetime = 1296000, // 15 días
    AbsoluteRefreshTokenLifetime = 2592000 // 30 días
}
```

## Monitorización post-migración

Implementamos dashboard con métricas clave:

```csharp
// Custom middleware para tracking
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();

    try
    {
        await next();
    }
    finally
    {
        sw.Stop();

        _metrics.RecordCounter("requests_total", 1,
            new Dictionary<string, object>
            {
                { "endpoint", context.Request.Path },
                { "status_code", context.Response.StatusCode }
            });

        _metrics.RecordHistogram("request_duration_ms", sw.ElapsedMilliseconds,
            new Dictionary<string, object>
            {
                { "endpoint", context.Request.Path }
            });
    }
});
```

## Resultados

**Antes (sistema legacy)**:
- Tiempo de login: 800ms promedio
- Downtime mensual: 2-3 horas
- Bugs de seguridad: varios al año

**Después (IdentityServer 3)**:
- Tiempo de login: 400ms promedio
- Downtime mensual: 0 horas
- Cumplimiento OAuth 2.0 completo
- Logs y auditoría robusta

## Lecciones aprendidas

1. **Migración gradual > Big Bang**: Una app por vez reduce riesgo
2. **Dual running es crítico**: Poder rollback rápido salvó el proyecto
3. **Persistencia es clave**: Todo en memoria = problemas en producción
4. **Monitoring desde día 1**: Sin métricas, vuelas a ciegas
5. **Custom claims hay que planificarlos**: Las apps dependen de ellos

¿Has migrado a IdentityServer? ¿Qué estrategia seguiste?
