---
layout: post
title: IdentityServer 4 - El salto a .NET Core y OpenID Connect moderno
author: Vicente José Moreno Escobar
categories: [IdentityServer, .NET Core, OAuth2]
published: true
---

> Migración a IdentityServer 4 y .NET Core 2.2 - Nuevas posibilidades, nuevos retos

# Por qué IdentityServer 4

Después de usar IdentityServer 3 durante un año, nos dimos cuenta de varias limitaciones:
- Atado a .NET Framework (no cross-platform)
- Performance no optimizada
- La versión 4 traía mejoras significativas

Además, .NET Core 2.2 estaba maduro y queríamos aprovechar sus ventajas.

## Diferencias clave vs IdentityServer 3

### 1. .NET Core en lugar de .NET Framework

```csharp
// IdentityServer 3 - OWIN/Katana
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        app.UseIdentityServer(new IdentityServerOptions
        {
            // Configuración IS3
        });
    }
}

// IdentityServer 4 - ASP.NET Core
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddIdentityServer()
            .AddInMemoryClients(Config.Clients)
            .AddInMemoryIdentityResources(Config.IdentityResources)
            .AddInMemoryApiResources(Config.ApiResources);
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseIdentityServer();
    }
}
```

### 2. Configuración más limpia

IS4 introduce conceptos más claros:

```csharp
// Config.cs - Mucho más legible que IS3
public static class Config
{
    public static IEnumerable<IdentityResource> IdentityResources =>
        new List<IdentityResource>
        {
            new IdentityResources.OpenId(),
            new IdentityResources.Profile(),
            new IdentityResources.Email()
        };

    public static IEnumerable<ApiResource> ApiResources =>
        new List<ApiResource>
        {
            new ApiResource("api1", "My API")
            {
                Scopes = { "api1.read", "api1.write" }
            }
        };

    public static IEnumerable<Client> Clients =>
        new List<Client>
        {
            new Client
            {
                ClientId = "client",
                AllowedGrantTypes = GrantTypes.ClientCredentials,
                ClientSecrets = { new Secret("secret".Sha256()) },
                AllowedScopes = { "api1.read" }
            }
        };
}
```

## La migración paso a paso

### 1. Crear proyecto .NET Core 2.2

```bash
dotnet new web -n IdentityServer4Demo
cd IdentityServer4Demo
dotnet add package IdentityServer4
```

### 2. Configuración básica

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    var builder = services.AddIdentityServer(options =>
    {
        options.Events.RaiseErrorEvents = true;
        options.Events.RaiseInformationEvents = true;
        options.Events.RaiseFailureEvents = true;
        options.Events.RaiseSuccessEvents = true;
    })
    .AddInMemoryIdentityResources(Config.IdentityResources)
    .AddInMemoryApiResources(Config.ApiResources)
    .AddInMemoryClients(Config.Clients)
    .AddTestUsers(TestUsers.Users); // Solo para desarrollo

    // Signing credential (certificado para firmar tokens)
    builder.AddDeveloperSigningCredential(); // Solo dev!
}
```

### 3. Migrar usuarios desde IS3

Nuestro sistema anterior tenía usuarios en SQL Server. Había que migrarlos:

```csharp
public class CustomUserStore : IUserStore<ApplicationUser>
{
    private readonly ApplicationDbContext _context;

    public async Task<ApplicationUser> FindByUsernameAsync(string username)
    {
        return await _context.Users
            .FirstOrDefaultAsync(u => u.Username == username);
    }

    public async Task<bool> ValidateCredentialsAsync(
        string username,
        string password)
    {
        var user = await FindByUsernameAsync(username);
        if (user == null) return false;

        // Validar password (usando mismo hash que IS3)
        var hasher = new PasswordHasher<ApplicationUser>();
        var result = hasher.VerifyHashedPassword(user, user.PasswordHash, password);

        return result == PasswordVerificationResult.Success;
    }
}

// Registrar en Startup.cs
services.AddScoped<IUserStore<ApplicationUser>, CustomUserStore>();

services.AddIdentityServer()
    .AddAspNetIdentity<ApplicationUser>(); // Integra con ASP.NET Core Identity
```

### 4. ProfileService custom

Para incluir claims personalizados en los tokens:

```csharp
public class CustomProfileService : IProfileService
{
    private readonly IUserClaimsPrincipalFactory<ApplicationUser> _claimsFactory;
    private readonly UserManager<ApplicationUser> _userManager;

    public CustomProfileService(
        UserManager<ApplicationUser> userManager,
        IUserClaimsPrincipalFactory<ApplicationUser> claimsFactory)
    {
        _userManager = userManager;
        _claimsFactory = claimsFactory;
    }

    public async Task GetProfileDataAsync(ProfileDataRequestContext context)
    {
        var sub = context.Subject.GetSubjectId();
        var user = await _userManager.FindByIdAsync(sub);
        var principal = await _claimsFactory.CreateAsync(user);

        var claims = principal.Claims.ToList();

        // Añadir custom claims
        claims.Add(new Claim("employee_id", user.EmployeeId));
        claims.Add(new Claim("department", user.Department));

        context.IssuedClaims = claims;
    }

    public async Task IsActiveAsync(IsActiveContext context)
    {
        var sub = context.Subject.GetSubjectId();
        var user = await _userManager.FindByIdAsync(sub);
        context.IsActive = user != null && user.IsActive;
    }
}

// Registrar
services.AddTransient<IProfileService, CustomProfileService>();
```

## API Resources vs API Scopes

IS4 introduce una distinción importante:

```csharp
// API Resource - La API en sí
new ApiResource("invoice-api", "Invoice API")
{
    Scopes = { "invoice.read", "invoice.write", "invoice.delete" },
    UserClaims = { "role", "department" }
}

// Los Scopes se definen separadamente
new ApiScope("invoice.read", "Read invoices"),
new ApiScope("invoice.write", "Create and update invoices"),
new ApiScope("invoice.delete", "Delete invoices")
```

Esto permite granularidad más fina:

```csharp
new Client
{
    ClientId = "accounting-app",
    AllowedScopes = { "invoice.read", "invoice.write" } // No puede borrar
}

new Client
{
    ClientId = "admin-app",
    AllowedScopes = { "invoice.read", "invoice.write", "invoice.delete" }
}
```

## Certificado de firma en producción

**Crítico**: NO usar `AddDeveloperSigningCredential()` en producción.

```csharp
// Cargar certificado desde Azure Key Vault
public void ConfigureServices(IServiceCollection services)
{
    var azureServiceTokenProvider = new AzureServiceTokenProvider();
    var keyVaultClient = new KeyVaultClient(
        new KeyVaultClient.AuthenticationCallback(
            azureServiceTokenProvider.KeyVaultTokenCallback));

    var certSecret = await keyVaultClient
        .GetSecretAsync("https://myvault.vault.azure.net/secrets/SigningCert");

    var certBytes = Convert.FromBase64String(certSecret.Value);
    var cert = new X509Certificate2(certBytes);

    services.AddIdentityServer()
        .AddSigningCredential(cert)
        // ...
}
```

O generarlo y persistirlo:

```csharp
services.AddIdentityServer()
    .AddSigningCredential(LoadCertificate())
    // ...

private X509Certificate2 LoadCertificate()
{
    var certPath = Path.Combine(_environment.ContentRootPath, "signing-cert.pfx");

    if (!File.Exists(certPath))
    {
        throw new Exception($"Signing certificate not found: {certPath}");
    }

    return new X509Certificate2(certPath, "password",
        X509KeyStorageFlags.MachineKeySet |
        X509KeyStorageFlags.PersistKeySet |
        X509KeyStorageFlags.Exportable);
}
```

## Persistencia de configuración

En producción, no queremos configuración en memoria:

```csharp
// Install packages
// dotnet add package IdentityServer4.EntityFramework

public void ConfigureServices(IServiceCollection services)
{
    var migrationsAssembly = typeof(Startup).Assembly.GetName().Name;
    var connectionString = Configuration.GetConnectionString("IdentityServer");

    services.AddIdentityServer()
        .AddConfigurationStore(options =>
        {
            options.ConfigureDbContext = builder =>
                builder.UseSqlServer(connectionString,
                    sql => sql.MigrationsAssembly(migrationsAssembly));
        })
        .AddOperationalStore(options =>
        {
            options.ConfigureDbContext = builder =>
                builder.UseSqlServer(connectionString,
                    sql => sql.MigrationsAssembly(migrationsAssembly));

            options.EnableTokenCleanup = true;
            options.TokenCleanupInterval = 3600; // 1 hora
        });
}
```

Crear migraciones:

```bash
dotnet ef migrations add InitialIdentityServerPersistedGrantDbMigration -c PersistedGrantDbContext
dotnet ef migrations add InitialIdentityServerConfigurationDbMigration -c ConfigurationDbContext
dotnet ef database update -c PersistedGrantDbContext
dotnet ef database update -c ConfigurationDbContext
```

## Personalización de UI

IS4 trae UI básica, pero normalmente quieres personalizarla:

```bash
# Scaffold UI
dotnet new is4ui
```

Esto genera:
- Login page
- Consent page
- Logout page
- Error page

Personalizamos la página de login:

```html
<!-- Views/Account/Login.cshtml -->
@model LoginViewModel

<div class="login-page">
    <div class="login-box">
        <h1>Iniciar sesión</h1>

        <form asp-route="Login">
            <input type="hidden" asp-for="ReturnUrl" />

            <div class="form-group">
                <label asp-for="Username">Usuario</label>
                <input asp-for="Username" class="form-control" autofocus />
            </div>

            <div class="form-group">
                <label asp-for="Password">Contraseña</label>
                <input type="password" asp-for="Password" class="form-control" />
            </div>

            <div class="form-group">
                <label>
                    <input asp-for="RememberLogin" />
                    Recordar sesión
                </label>
            </div>

            <button type="submit" class="btn btn-primary">Entrar</button>
        </form>

        <a asp-action="ForgotPassword">¿Olvidaste tu contraseña?</a>
    </div>
</div>
```

## Performance

IS4 es mucho más rápido que IS3:

**Benchmark** (token endpoint, 1000 requests):

```
IdentityServer 3:
  - Promedio: 450ms
  - p95: 1200ms
  - p99: 2100ms

IdentityServer 4:
  - Promedio: 85ms
  - p95: 180ms
  - p99: 320ms
```

Mejoras que encontramos:
- Caching más agresivo de metadata
- Menos allocations
- Async/await optimizado en .NET Core

## Problemas que enfrentamos

### 1. Breaking changes en configuración

Muchas cosas cambiaron de IS3:

```csharp
// IS3
new Client
{
    Flow = Flows.Implicit
}

// IS4
new Client
{
    AllowedGrantTypes = GrantTypes.Implicit
}
```

### 2. Discovery endpoint cambió

```
// IS3
https://auth.ejemplo.com/.well-known/openid-configuration

// IS4 - igual, pero devuelve JSON diferente
https://auth.ejemplo.com/.well-known/openid-configuration
```

Las apps cliente tuvieron que actualizarse.

### 3. CORS más estricto

```csharp
// Configurar CORS explícitamente para clientes JavaScript
new Client
{
    ClientId = "spa-client",
    AllowedGrantTypes = GrantTypes.Code,
    RequirePkce = true,
    RequireClientSecret = false,

    AllowedCorsOrigins = { "https://localhost:5002" },
    RedirectUris = { "https://localhost:5002/callback" },
    PostLogoutRedirectUris = { "https://localhost:5002/" }
}
```

## Conclusión

IdentityServer 4 es una mejora masiva sobre IS3:
- Más rápido
- Más moderno (.NET Core)
- Mejor documentación
- Ecosistema más activo

La migración tomó 3 semanas pero valió cada hora invertida.

¿Has migrado a IdentityServer 4? ¿Qué obstáculos encontraste?
