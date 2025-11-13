---
layout: post
title: IdentityServer 4 - Integración con proveedores externos (Google, Facebook, Azure AD)
author: Vicente José Moreno Escobar
categories: [IdentityServer, OAuth2, Integración]
published: true
---

> Cuando los usuarios quieren "Sign in with Google" en tu app corporativa

# El contexto

Portal B2C que necesitaba múltiples opciones de login:
- Email/password local
- Google
- Facebook
- Azure AD corporativo (para empleados)

IdentityServer 4 como authorization server centralizado.

## External Login en IdentityServer

IdentityServer actúa como **federation gateway**:

```
Usuario → IdentityServer → Google/Facebook/Azure AD
              ↓
         Valida con proveedor externo
              ↓
         Emite sus propios tokens
              ↓
         Apps cliente (no saben de Google/etc)
```

Ventaja: Las apps solo conocen IdentityServer, no cada proveedor.

## Configuración Google

### 1. Crear OAuth client en Google Cloud Console

```
https://console.cloud.google.com
→ APIs & Services → Credentials → Create OAuth 2.0 Client ID

Application type: Web application
Authorized redirect URIs:
  https://auth.tusitio.com/signin-google
```

Anotar Client ID y Client Secret.

### 2. Configurar en IdentityServer

```bash
dotnet add package Microsoft.AspNetCore.Authentication.Google
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddIdentityServer()
        .AddInMemoryClients(Config.Clients)
        // ...

    services.AddAuthentication()
        .AddGoogle("Google", options =>
        {
            options.SignInScheme = IdentityServerConstants.ExternalCookieAuthenticationScheme;

            options.ClientId = Configuration["Google:ClientId"];
            options.ClientSecret = Configuration["Google:ClientSecret"];

            // Scopes adicionales si necesitas más info
            options.Scope.Add("profile");
            options.Scope.Add("email");

            // Mapear claims de Google a claims de IdentityServer
            options.ClaimActions.MapJsonKey("picture", "picture");
            options.ClaimActions.MapJsonKey("locale", "locale");

            // Guardar tokens de Google (opcional)
            options.SaveTokens = true;
        });
}
```

### 3. appsettings.json

```json
{
  "Google": {
    "ClientId": "tu-client-id.apps.googleusercontent.com",
    "ClientSecret": "tu-client-secret"
  }
}
```

## Configuración Facebook

### 1. Crear app en Facebook Developers

```
https://developers.facebook.com/apps
→ Create App → Consumer → Add Facebook Login

Settings → Basic:
  App Domains: tusitio.com

Facebook Login → Settings:
  Valid OAuth Redirect URIs:
    https://auth.tusitio.com/signin-facebook
```

### 2. IdentityServer

```bash
dotnet add package Microsoft.AspNetCore.Authentication.Facebook
```

```csharp
services.AddAuthentication()
    .AddFacebook("Facebook", options =>
    {
        options.SignInScheme = IdentityServerConstants.ExternalCookieAuthenticationScheme;

        options.AppId = Configuration["Facebook:AppId"];
        options.AppSecret = Configuration["Facebook:AppSecret"];

        options.Scope.Add("email");
        options.Scope.Add("public_profile");

        options.Fields.Add("name");
        options.Fields.Add("email");
        options.Fields.Add("picture");

        options.SaveTokens = true;
    });
```

## Configuración Azure AD

Para permitir login con cuentas corporativas Microsoft:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
```

```csharp
services.AddAuthentication()
    .AddOpenIdConnect("AzureAD", "Azure AD", options =>
    {
        options.SignInScheme = IdentityServerConstants.ExternalCookieAuthenticationScheme;
        options.SignOutScheme = IdentityServerConstants.SignoutScheme;

        options.Authority = "https://login.microsoftonline.com/common";
        options.ClientId = Configuration["AzureAd:ClientId"];
        options.ClientSecret = Configuration["AzureAd:ClientSecret"];

        options.ResponseType = "code";
        options.UsePkce = true;

        options.Scope.Clear();
        options.Scope.Add("openid");
        options.Scope.Add("profile");
        options.Scope.Add("email");

        options.TokenValidationParameters.ValidateIssuer = false; // Multi-tenant

        options.GetClaimsFromUserInfoEndpoint = true;
        options.SaveTokens = true;

        // Mapear claims de Azure AD
        options.ClaimActions.MapJsonKey("sub", "sub");
        options.ClaimActions.MapJsonKey("name", "name");
        options.ClaimActions.MapJsonKey("email", "email");
    });
```

## UI de Login con múltiples providers

IdentityServer scaffold UI necesita customización:

```csharp
// AccountController.cs
[HttpGet]
public async Task<IActionResult> Login(string returnUrl)
{
    var context = await _interaction.GetAuthorizationContextAsync(returnUrl);

    if (context?.IdP != null && await _schemeProvider.GetSchemeAsync(context.IdP) != null)
    {
        // Bypas si IdP específico solicitado
        return await ExternalLogin(context.IdP, returnUrl);
    }

    var schemes = await _schemeProvider.GetAllSchemesAsync();

    var providers = schemes
        .Where(x => x.DisplayName != null)
        .Select(x => new ExternalProvider
        {
            DisplayName = x.DisplayName ?? x.Name,
            AuthenticationScheme = x.Name
        }).ToList();

    var vm = new LoginViewModel
    {
        ReturnUrl = returnUrl,
        ExternalProviders = providers
    };

    return View(vm);
}
```

Vista de login:

```html
<!-- Views/Account/Login.cshtml -->
<div class="login-page">
    <div class="lead">
        <h1>Sign in</h1>
    </div>

    <div class="row">
        <!-- Login local -->
        <div class="col-sm-6">
            <div class="panel">
                <div class="panel-header">
                    <h3>Local Account</h3>
                </div>
                <div class="panel-body">
                    <form asp-route="Login">
                        <input type="hidden" asp-for="ReturnUrl" />
                        <fieldset>
                            <div class="form-group">
                                <label asp-for="Username">Username</label>
                                <input class="form-control" asp-for="Username" autofocus>
                            </div>
                            <div class="form-group">
                                <label asp-for="Password">Password</label>
                                <input type="password" class="form-control" asp-for="Password">
                            </div>
                            <div class="form-group">
                                <label>
                                    <input asp-for="RememberLogin">
                                    Remember Me
                                </label>
                            </div>
                            <button class="btn btn-primary" name="button" value="login">Login</button>
                        </fieldset>
                    </form>
                </div>
            </div>
        </div>

        <!-- External providers -->
        <div class="col-sm-6">
            <div class="panel">
                <div class="panel-header">
                    <h3>External Login</h3>
                </div>
                <div class="panel-body">
                    @if (Model.ExternalProviders.Any())
                    {
                        <form asp-action="ExternalLogin" asp-route-returnUrl="@Model.ReturnUrl">
                            @foreach (var provider in Model.ExternalProviders)
                            {
                                <button type="submit"
                                        class="btn btn-secondary btn-block external-provider-button"
                                        name="provider"
                                        value="@provider.AuthenticationScheme">
                                    @if (provider.DisplayName == "Google")
                                    {
                                        <i class="fab fa-google"></i>
                                    }
                                    else if (provider.DisplayName == "Facebook")
                                    {
                                        <i class="fab fa-facebook"></i>
                                    }
                                    else if (provider.DisplayName == "Azure AD")
                                    {
                                        <i class="fab fa-microsoft"></i>
                                    }
                                    Sign in with @provider.DisplayName
                                </button>
                            }
                        </form>
                    }
                    else
                    {
                        <p>No external login providers configured</p>
                    }
                </div>
            </div>
        </div>
    </div>
</div>
```

## Callback de External Login

```csharp
// AccountController.cs
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult ExternalLogin(string provider, string returnUrl)
{
    // Redirigir a proveedor externo
    var redirectUrl = Url.Action(nameof(ExternalLoginCallback), new { returnUrl });

    var properties = new AuthenticationProperties
    {
        RedirectUri = redirectUrl,
        Items =
        {
            { "scheme", provider }
        }
    };

    return Challenge(properties, provider);
}

[HttpGet]
public async Task<IActionResult> ExternalLoginCallback(string returnUrl)
{
    // Leer resultado de autenticación externa
    var result = await HttpContext.AuthenticateAsync(IdentityServerConstants.ExternalCookieAuthenticationScheme);

    if (result?.Succeeded != true)
    {
        throw new Exception("External authentication error");
    }

    var externalUser = result.Principal;
    var claims = externalUser.Claims.ToList();

    // Obtener provider name
    var provider = result.Properties.Items["scheme"];
    var providerUserId = claims.FirstOrDefault(x => x.Type == ClaimTypes.NameIdentifier)?.Value;

    // Buscar usuario existente o crear nuevo
    var user = await _userStore.FindByExternalProviderAsync(provider, providerUserId);

    if (user == null)
    {
        // Auto-provision usuario
        user = await AutoProvisionUser(provider, providerUserId, claims);
    }

    // Emitir cookie de autenticación local
    await HttpContext.SignInAsync(user.SubjectId, user.Username, provider, result.Properties, claims);

    // Limpiar external cookie
    await HttpContext.SignOutAsync(IdentityServerConstants.ExternalCookieAuthenticationScheme);

    // Redirigir de vuelta a returnUrl
    return Redirect(returnUrl);
}

private async Task<User> AutoProvisionUser(string provider, string providerUserId, List<Claim> claims)
{
    var email = claims.FirstOrDefault(x => x.Type == ClaimTypes.Email)?.Value;
    var name = claims.FirstOrDefault(x => x.Type == ClaimTypes.Name)?.Value;

    var user = new User
    {
        SubjectId = Guid.NewGuid().ToString(),
        Username = email,
        Email = email,
        ProviderName = provider,
        ProviderSubjectId = providerUserId
    };

    await _userStore.AddUserAsync(user);

    return user;
}
```

## Almacenamiento de External Logins

```csharp
// Data/ApplicationUser.cs
public class ApplicationUser
{
    public string Id { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }

    // Para local accounts
    public string PasswordHash { get; set; }

    // Para external logins
    public string ProviderName { get; set; } // "Google", "Facebook", etc.
    public string ProviderSubjectId { get; set; } // ID del usuario en el proveedor

    public DateTime Created { get; set; }
    public DateTime? LastLogin { get; set; }
}

// UserStore
public class UserStore
{
    private readonly ApplicationDbContext _context;

    public async Task<ApplicationUser> FindByExternalProviderAsync(string provider, string providerUserId)
    {
        return await _context.Users
            .FirstOrDefaultAsync(u =>
                u.ProviderName == provider &&
                u.ProviderSubjectId == providerUserId);
    }

    public async Task<ApplicationUser> FindByUsernameAsync(string username)
    {
        return await _context.Users
            .FirstOrDefaultAsync(u => u.Username == username);
    }
}
```

## Linking de cuentas

Permitir que usuario vincule múltiples proveedores:

```csharp
// AccountController.cs
[HttpGet]
[Authorize]
public async Task<IActionResult> LinkLogin(string provider)
{
    var redirectUrl = Url.Action(nameof(LinkLoginCallback));

    var properties = new AuthenticationProperties
    {
        RedirectUri = redirectUrl,
        Items = { { "scheme", provider } }
    };

    return Challenge(properties, provider);
}

[HttpGet]
[Authorize]
public async Task<IActionResult> LinkLoginCallback()
{
    var result = await HttpContext.AuthenticateAsync(IdentityServerConstants.ExternalCookieAuthenticationScheme);
    var externalUserId = result.Principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var provider = result.Properties.Items["scheme"];

    var currentUserId = User.GetSubjectId();

    // Vincular external provider al usuario actual
    await _userStore.AddExternalLoginAsync(currentUserId, provider, externalUserId);

    await HttpContext.SignOutAsync(IdentityServerConstants.ExternalCookieAuthenticationScheme);

    return RedirectToAction("Manage");
}
```

## Problemas comunes

### 1. Callback URL mismatch

```
Error: redirect_uri_mismatch
```

**Solución**: Verificar que redirect URI coincide EXACTAMENTE:

```
Configurado en Google: https://auth.tusitio.com/signin-google
Usado por IdentityServer: https://auth.tusitio.com/signin-google

// Sin trailing slash, HTTPS, caso exacto
```

### 2. Facebook pide App Review para email

Desde 2018, Facebook requiere revisión para pedir scope `email`.

**Solución**: Enviar app a revisión o hacer email opcional.

### 3. Usuarios duplicados

Usuario se registra con email local, luego con Google usando mismo email.

**Solución**: Matching automático por email:

```csharp
var emailFromGoogle = claims.FirstOrDefault(x => x.Type == ClaimTypes.Email)?.Value;

var existingUser = await _userStore.FindByEmailAsync(emailFromGoogle);

if (existingUser != null)
{
    // Link automáticamente
    await _userStore.AddExternalLoginAsync(existingUser.Id, provider, providerUserId);
    user = existingUser;
}
else
{
    // Crear nuevo
    user = await AutoProvisionUser(provider, providerUserId, claims);
}
```

## Mejores prácticas

1. **Auto-provision usuarios** - No forzar registro manual después de external login
2. **Guardar tokens externos** - `SaveTokens = true` si necesitas llamar APIs del proveedor
3. **Permitir linking** - Usuarios deberían poder vincular múltiples providers
4. **Email como identificador único** - Unificar cuentas por email cuando sea posible
5. **Monitorear proveedores** - Facebook/Google cambian políticas frecuentemente

## Resultados

Con external providers configurados:
- 60% de usuarios eligen Google
- 25% Facebook
- 10% Azure AD (empleados)
- 5% Local accounts

Reducción dramática de "olvidé mi contraseña" porque usuarios usan OAuth.

¿Qué proveedores externos usas? ¿Has tenido problemas con cambios de políticas?
