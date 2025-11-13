---
layout: post
title: Azure AD B2C - Cuando tus clientes necesitan autenticarse
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD B2C, Autenticación]
published: true
---

> Implementando autenticación para 50,000 clientes con Azure AD B2C

# El reto

Portal de clientes que necesitaba:
- Registro self-service
- Login con redes sociales (Google, Facebook)
- Reset de password sin llamar a soporte
- Personalización de marca
- MFA opcional

Azure AD B2C era perfecto para esto.

## B2C vs B2B - La confusión inicial

Cuando empecé, confundía ambos:

**Azure AD B2B**: Para colaboradores/partners con cuentas corporativas
**Azure AD B2C**: Para clientes/consumidores con cuentas personales

Necesitábamos B2C.

## Configuración inicial

### 1. Crear tenant B2C

```bash
# Importante: tenant B2C es SEPARADO de tu tenant Azure AD normal
Nombre: clientescontoso.onmicrosoft.com
```

### 2. Registrar aplicación

```
Azure AD B2C → App registrations → New registration

Redirect URIs:
- https://portal.contoso.com/signin-oidc
- https://portal.contoso.com/signout-callback-oidc
```

### 3. User Flows

Los "User Flows" son wizards pre-built para:
- Sign up and sign in
- Profile editing
- Password reset

```csharp
// En tu app .NET
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(options =>
    {
        options.Instance = "https://clientescontoso.b2clogin.com/";
        options.Domain = "clientescontoso.onmicrosoft.com";
        options.ClientId = "<client-id>";
        options.SignUpSignInPolicyId = "B2C_1_signupsignin";
        options.ResetPasswordPolicyId = "B2C_1_passwordreset";
        options.EditProfilePolicyId = "B2C_1_profileediting";
    });
```

## Personalización de UI

Por defecto, las páginas de B2C son feas. Las personalizamos:

### 1. Templates HTML custom

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>Portal Clientes - Login</title>
    <link rel="stylesheet" href="https://portal.contoso.com/css/b2c-custom.css">
</head>
<body>
    <div id="api"></div>
    <!-- B2C inyecta el formulario aquí -->
</body>
</html>
```

### 2. Hospedar en Azure Storage

```bash
# Subir templates a Blob Storage
az storage blob upload \
  --account-name b2ctemplates \
  --container-name templates \
  --name unified.html \
  --file unified.html \
  --content-type "text/html"

# Habilitar CORS
az storage cors add \
  --services b \
  --methods GET OPTIONS \
  --origins https://clientescontoso.b2clogin.com \
  --allowed-headers "*"
```

### 3. Configurar en B2C

```
User flows → B2C_1_signupsignin → Page layouts
→ Unified sign up or sign in page
→ Use custom page content: https://b2ctemplates.blob.core.windows.net/templates/unified.html
```

## Integración con redes sociales

### Google

```
1. Google Cloud Console → Create OAuth client
2. Redirect URI: https://clientescontoso.b2clogin.com/clientescontoso.onmicrosoft.com/oauth2/authresp
3. Azure B2C → Identity providers → Google
4. Client ID + Secret de Google
```

### Facebook

Similar, pero Facebook cambia sus políticas cada 6 meses (dolor de cabeza constante).

## Custom attributes

Los clientes necesitaban campos adicionales:

```
Azure B2C → User attributes → Add
- CompanyName (string)
- PhoneNumber (string)
- PreferredLanguage (string)
```

Leerlos en la app:

```csharp
var companyName = User.FindFirst("extension_CompanyName")?.Value;
var phone = User.FindFirst("extension_PhoneNumber")?.Value;
```

**Importante**: El prefijo `extension_` es automático de B2C.

## Problemas encontrados

### 1. Password reset loop

Cuando un usuario clickeaba "Forgot password", B2C lanzaba un error que redirigía a la app, la app no sabía manejar el error, loop infinito.

**Solución**:

```csharp
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(options =>
    {
        // ... otras configs
    })
    .EnableTokenAcquisitionToCallDownstreamApi()
    .AddInMemoryTokenCaches();

// Manejar el error de password reset
services.Configure<OpenIdConnectOptions>(OpenIdConnectDefaults.AuthenticationScheme, options =>
{
    options.Events.OnRemoteFailure = async context =>
    {
        context.HandleResponse();

        // Si usuario canceló password reset
        if (context.Failure.Message.Contains("AADB2C90118"))
        {
            context.Response.Redirect("/");
        }
        else
        {
            context.Response.Redirect($"/Error?message={context.Failure.Message}");
        }
    };
});
```

### 2. Emails de verificación personalizados

Los emails por defecto eran genéricos. Los personalizamos vía Custom Policies (IEF - Identity Experience Framework).

```xml
<ContentDefinition Id="api.localaccountsignup">
    <LoadUri>https://portal.contoso.com/templates/signup.html</LoadUri>
    <DataUri>urn:com:microsoft:aad:b2c:elements:contract:unifiedssp:2.1.0</DataUri>
</ContentDefinition>
```

### 3. Rate limiting

B2C tiene límites:
- 60 requests/min por tenant
- Más allá, empiezas a ver 429 (Too Many Requests)

**Solución**: Implementar caching de tokens y retry logic:

```csharp
public class B2CTokenService
{
    private readonly IMemoryCache _cache;

    public async Task<string> GetTokenAsync(string userId)
    {
        var cacheKey = $"token_{userId}";

        if (_cache.TryGetValue(cacheKey, out string token))
        {
            return token;
        }

        // Retry logic
        var policy = Policy
            .Handle<HttpRequestException>()
            .WaitAndRetryAsync(3, retryAttempt =>
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

        token = await policy.ExecuteAsync(async () =>
        {
            return await FetchTokenFromB2C(userId);
        });

        _cache.Set(cacheKey, token, TimeSpan.FromMinutes(50));
        return token;
    }
}
```

## MFA (Multi-Factor Authentication)

Activar MFA para clientes que lo quisieran:

```
User flows → B2C_1_signupsignin → Properties
→ Multifactor authentication → Optional
```

Los usuarios podían habilitar MFA desde su perfil.

## Migración de usuarios existentes

Teníamos 10,000 usuarios en sistema legacy. Dos opciones:

**Opción 1**: Forzar re-registro (mala UX)
**Opción 2**: Migrar con Graph API

```csharp
var graphClient = new GraphServiceClient(authProvider);

foreach (var legacyUser in legacyUsers)
{
    var b2cUser = new User
    {
        Identities = new List<ObjectIdentity>
        {
            new ObjectIdentity
            {
                SignInType = "emailAddress",
                Issuer = "clientescontoso.onmicrosoft.com",
                IssuerAssignedId = legacyUser.Email
            }
        },
        PasswordProfile = new PasswordProfile
        {
            Password = GenerateRandomPassword(),
            ForceChangePasswordNextSignIn = true
        },
        DisplayName = legacyUser.FullName,
        GivenName = legacyUser.FirstName,
        Surname = legacyUser.LastName
    };

    await graphClient.Users.Request().AddAsync(b2cUser);

    // Enviar email con instrucciones de reset
    await SendPasswordResetEmail(legacyUser.Email);
}
```

## Costos

Azure AD B2C pricing (2018):
- Primeros 50,000 MAU (Monthly Active Users): Gratis
- 50,000 - 100,000 MAU: $0.00325 por usuario
- 100,000+: Contactar ventas

Para nosotros (50,000 usuarios): **$0/mes** durante el primer año.

## Monitoreo

Métricas clave que monitoreamos:

```
Azure Monitor → Logs

// Logins exitosos
AuditLogs
| where OperationName == "Sign in to local account"
| where Result == "success"
| summarize count() by bin(TimeGenerated, 1h)

// Logins fallidos
AuditLogs
| where OperationName == "Sign in to local account"
| where Result == "failure"
| summarize count() by ResultReason, bin(TimeGenerated, 1h)

// Registros nuevos
AuditLogs
| where OperationName == "Sign up to local account"
| summarize count() by bin(TimeGenerated, 1d)
```

## Resultados

**Antes (sistema custom)**:
- 5-10 tickets/día de "olvidé mi password"
- Soporte gastaba 20h/semana en gestión de cuentas
- No teníamos MFA
- UI fea

**Después (B2C)**:
- 0-1 tickets/día relacionados con auth
- Self-service completo
- MFA opcional funcionando
- UI con marca corporativa
- Integración con Google/Facebook

## Recomendaciones

1. **User Flows para 80% de casos**: Son suficientes para mayoría de escenarios
2. **Custom Policies solo si necesitas**: Complejos de mantener
3. **Monitorea limits de rate**: 60 req/min se alcanzan rápido
4. **Caché tokens agresivamente**: Reduce llamadas a B2C
5. **Plan de migración claro**: No intentes migrar todo de golpe

¿Usas Azure AD B2C? ¿Qué problemas has encontrado?
