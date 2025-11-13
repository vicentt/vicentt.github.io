---
layout: post
title: Duende IdentityServer - Migración desde IdentityServer 4
author: Vicente José Moreno Escobar
categories: [Duende, IdentityServer, OAuth2]
published: true
---

> Cuando IdentityServer 4 dejó de ser gratis y tocó migrar

# El cambio

Septiembre 2020: Anuncio de que IdentityServer 4 dejaba de ser open source.

Nueva versión: **Duende IdentityServer** con licenciamiento comercial para empresas.

Opciones:
1. Quedarse en IdentityServer 4 (sin updates)
2. Migrar a Duende (pagar licencia)
3. Migrar a alternativa (Auth0, Keycloak, etc.)

Elegimos **Duende** porque:
- Mismo equipo desarrollando
- Path de migración claro
- Features nuevas interesantes

## Licenciamiento

Duende es gratis si:
- Desarrollo/testing
- Organizaciones sin fines de lucro
- Ingresos <1M USD/año

Empresas grandes: **$1500-15000/año** según escala.

## Diferencias principales

| IdentityServer 4 | Duende IdentityServer |
|------------------|----------------------|
| Gratis OSS | Licencia comercial |
| .NET Core 3.1 | .NET 6+ |
| Namespace `IdentityServer4` | Namespace `Duende.IdentityServer` |
| Algunos features básicos | Features enterprise adicionales |

## Migración paso a paso

### 1. Actualizar packages

```bash
# Remover paquetes viejos
dotnet remove package IdentityServer4
dotnet remove package IdentityServer4.EntityFramework
dotnet remove package IdentityServer4.AspNetIdentity

# Instalar Duende
dotnet add package Duende.IdentityServer
dotnet add package Duende.IdentityServer.EntityFramework
dotnet add package Duende.IdentityServer.AspNetIdentity
```

### 2. Actualizar namespaces

Buscar y reemplazar en todo el proyecto:

```
IdentityServer4 → Duende.IdentityServer
```

```csharp
// Antes
using IdentityServer4;
using IdentityServer4.Models;
using IdentityServer4.Services;

// Después
using Duende.IdentityServer;
using Duende.IdentityServer.Models;
using Duende.IdentityServer.Services;
```

### 3. Actualizar configuración

Mayoría de configuración es compatible:

```csharp
// Startup.cs - prácticamente igual
services.AddIdentityServer(options =>
{
    options.Events.RaiseErrorEvents = true;
    options.Events.RaiseInformationEvents = true;
    options.Events.RaiseFailureEvents = true;
    options.Events.RaiseSuccessEvents = true;
})
.AddConfigurationStore(options =>
{
    options.ConfigureDbContext = b => b.UseSqlServer(connectionString);
})
.AddOperationalStore(options =>
{
    options.ConfigureDbContext = b => b.UseSqlServer(connectionString);
    options.EnableTokenCleanup = true;
})
.AddAspNetIdentity<ApplicationUser>();
```

### 4. Actualizar Entity Framework migrations

```bash
# Crear nueva migración para Duende
dotnet ef migrations add DuendeMigration -c PersistedGrantDbContext -o Data/Migrations/IdentityServer/PersistedGrantDb
dotnet ef migrations add DuendeMigration -c ConfigurationDbContext -o Data/Migrations/IdentityServer/ConfigurationDb

# Aplicar
dotnet ef database update -c PersistedGrantDbContext
dotnet ef database update -c ConfigurationDbContext
```

Schema de DB es compatible, solo cambios menores.

### 5. Configurar licencia

```csharp
// Startup.cs
services.AddIdentityServer(options =>
{
    options.LicenseKey = Configuration["Duende:LicenseKey"];
    // ... otras opciones
})
```

```json
// appsettings.json
{
  "Duende": {
    "LicenseKey": "tu-license-key-aqui"
  }
}
```

## Breaking changes menores

### 1. ProfileService cambió signature

```csharp
// IdentityServer 4
public class ProfileService : IProfileService
{
    public async Task GetProfileDataAsync(ProfileDataRequestContext context)
    {
        // ...
    }
}

// Duende - igual, sin cambios reales
// Pero algunos métodos internos cambiaron
```

### 2. Event names cambiaron ligeramente

```csharp
// Antes
options.Events.RaiseSuccessEvents = true;

// Después - igual
options.Events.RaiseSuccessEvents = true;
```

No hay cambios significativos aquí.

### 3. Validadores custom

```csharp
// Custom secret validator
public class CustomSecretValidator : ISecretValidator
{
    // Duende: mismo interface, sin cambios
    public Task<SecretValidationResult> ValidateAsync(
        IEnumerable<Secret> secrets,
        ParsedSecret parsedSecret)
    {
        // ...
    }
}
```

## Nuevas features de Duende

### 1. Mutual TLS (mTLS)

```csharp
services.AddIdentityServer()
    .AddMutualTlsSecretValidators();

// Client configurado para mTLS
new Client
{
    ClientId = "mtls-client",
    ClientSecrets =
    {
        new Secret
        {
            Type = IdentityServerConstants.SecretTypes.X509CertificateThumbprint,
            Value = "cert-thumbprint"
        }
    }
}
```

### 2. Pushed Authorization Requests (PAR)

```csharp
services.AddIdentityServer(options =>
{
    options.Endpoints.EnablePushedAuthorizationEndpoint = true;
});

new Client
{
    RequirePushedAuthorization = true
}
```

### 3. CIBA (Client Initiated Backchannel Authentication)

Para dispositivos sin navegador:

```csharp
services.AddIdentityServer()
    .AddBackchannelAuthenticationUserValidator<CustomBackchannelAuthenticationUserValidator>();
```

### 4. Improved logging

```csharp
services.AddIdentityServer(options =>
{
    options.Logging.TokenRequestSensitiveDataFilter = (request) =>
    {
        // Filtrar datos sensibles de logs
        request.ClientSecret = "***";
        return request;
    };
});
```

### 5. Server-side sessions

```csharp
services.AddIdentityServer(options =>
{
    options.ServerSideSessions.UserDisplayNameClaimType = "name";
    options.ServerSideSessions.RemoveExpiredSessionsFrequency = TimeSpan.FromMinutes(10);
});
```

## Alternativas consideradas

### Auth0

**Pros**:
- Totalmente managed
- No mantenimiento
- Features enterprise

**Contras**:
- Costoso ($$$)
- Lock-in vendor
- Menos control

### Keycloak

**Pros**:
- Open source gratis
- Features completos
- Comunidad grande

**Contras**:
- Java-based (nosotros .NET shop)
- Curva de aprendizaje
- Menos integración con ASP.NET

### OpenIddict

**Pros**:
- Open source gratis
- .NET nativo
- Activamente mantenido

**Contras**:
- Menos maduro
- Documentación más limitada
- Comunidad más pequeña

## Costos

Nuestro caso (empresa mediana):

```
Duende Business Edition: $1500/año
- 1 production environment
- Unlimited users
- Priority support

vs.

Auth0: ~$800/mes = $9600/año
```

Duende era 6x más barato para nosotros.

## Testing post-migración

```csharp
// Integration tests
[Fact]
public async Task DiscoveryEndpoint_ReturnsExpectedValues()
{
    var client = _factory.CreateClient();

    var disco = await client.GetDiscoveryDocumentAsync();

    Assert.False(disco.IsError);
    Assert.NotNull(disco.TokenEndpoint);
    Assert.NotNull(disco.AuthorizeEndpoint);
}

[Fact]
public async Task ClientCredentials_Flow_Works()
{
    var client = _factory.CreateClient();

    var response = await client.RequestClientCredentialsTokenAsync(
        new ClientCredentialsTokenRequest
        {
            Address = "https://localhost:5001/connect/token",
            ClientId = "test-client",
            ClientSecret = "secret",
            Scope = "api1"
        });

    Assert.False(response.IsError);
    Assert.NotNull(response.AccessToken);
}
```

## Rollout

### Fase 1: Staging (1 semana)

- Deploy Duende en staging
- Testing exhaustivo
- Performance testing

### Fase 2: Canary (1 semana)

- 10% de tráfico a Duende
- Monitoreo intensivo
- 0 issues detectados

### Fase 3: Full production

- 100% migrado
- IdentityServer 4 apagado

## Monitoreo post-migración

```csharp
// Application Insights
services.AddApplicationInsightsTelemetry();

// Custom metrics
public class IdentityServerMetrics
{
    private readonly TelemetryClient _telemetry;

    public void RecordTokenIssued(string clientId, string grantType)
    {
        _telemetry.TrackEvent("TokenIssued", new Dictionary<string, string>
        {
            ["ClientId"] = clientId,
            ["GrantType"] = grantType
        });
    }
}
```

## Problemas encontrados

### 1. License key en environment incorrecto

```
Error: "Invalid or missing license key"
```

**Solución**: Verificar que license key está en environment variable/config correcto.

### 2. Clientes legacy esperaban IdentityServer 4 metadata

Algunos clients validaban issuer name.

**Solución**: Mantener issuer name igual:

```csharp
options.IssuerUri = "https://auth.empresa.com"; // Mismo que antes
```

### 3. Performance degradation inicial

Duende tiene más logging por defecto.

**Solución**: Ajustar logging levels:

```json
{
  "Logging": {
    "LogLevel": {
      "Duende": "Warning"
    }
  }
}
```

## Resultados

**Métricas después de 6 meses**:

- Performance: Igual o mejor que IS4
- Features nuevas: Usamos PAR y mTLS
- Costo: $1500/año vs $0 antes (pero con support)
- Tiempo invertido en migración: 40 horas
- Issues post-migración: 0 critical

## Recomendación

Si usas IdentityServer 4:

**Migra a Duende si**:
- Tienes presupuesto ($1500+ no es mucho para enterprise)
- Quieres continuar recibiendo updates
- Valoras el support profesional

**Considera alternativas si**:
- Budget muy limitado
- Preferirías solución managed (Auth0, Okta)
- No necesitas features avanzadas (Keycloak, OpenIddict)

¿Has migrado a Duende? ¿O elegiste otra alternativa?
