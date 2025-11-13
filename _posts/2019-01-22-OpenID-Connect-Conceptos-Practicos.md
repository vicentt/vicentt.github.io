---
layout: post
title: OpenID Connect - De la teoría a la práctica sin morir en el intento
author: Vicente José Moreno Escobar
categories: [OpenID Connect, OAuth2, Seguridad]
published: true
---

> Cuando finalmente entendí la diferencia entre OAuth 2.0 y OpenID Connect

# La confusión inicial

Durante meses usé OAuth 2.0 y OpenID Connect (OIDC) intercambiablemente. Pensaba que eran lo mismo. No lo son.

Esta confusión me causó bugs sutiles que tardé semanas en diagnosticar.

## OAuth 2.0 vs OpenID Connect

La diferencia fundamental:

**OAuth 2.0**: Framework de **autorización**
- "¿Qué puede hacer esta app?"
- Da acceso a recursos
- No dice QUIÉN es el usuario

**OpenID Connect**: Capa de **autenticación** sobre OAuth 2.0
- "¿Quién es este usuario?"
- Proporciona información de identidad
- Se construye sobre OAuth 2.0

### Analogía del mundo real

Imagina un hotel:

```
OAuth 2.0 = Tarjeta de acceso a la habitación
  → Te da acceso (autorización)
  → No dice quién eres

OpenID Connect = Check-in en recepción
  → Verifican tu identidad (autenticación)
  → Te dan una tarjeta de acceso (autorización)
```

## El proyecto que me enseñó la diferencia

Teníamos una aplicación que necesitaba:
1. Saber quién es el usuario (nombre, email)
2. Acceder a su calendario (recurso protegido)

### Implementación incorrecta (solo OAuth 2.0)

```csharp
// Configuración OAuth 2.0 básica
services.AddAuthentication(OAuth2Defaults.AuthenticationScheme)
    .AddOAuth("MyProvider", options =>
    {
        options.ClientId = "client-id";
        options.ClientSecret = "client-secret";
        options.AuthorizationEndpoint = "https://auth.ejemplo.com/authorize";
        options.TokenEndpoint = "https://auth.ejemplo.com/token";

        options.Scope.Add("calendar.read");
    });
```

**Problema**: OAuth 2.0 me daba un access token para el calendario, pero no información del usuario de forma estándar.

Intentaba obtener el nombre del usuario así:

```csharp
[Authorize]
public IActionResult Profile()
{
    var userName = User.Identity.Name; // null o vacío!
    var email = User.FindFirst(ClaimTypes.Email)?.Value; // null!

    return View(new { Name = userName, Email = email });
}
```

**No funcionaba** porque OAuth 2.0 no está diseñado para eso.

### Implementación correcta (OpenID Connect)

```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.Authority = "https://auth.ejemplo.com";
    options.ClientId = "client-id";
    options.ClientSecret = "client-secret";
    options.ResponseType = "code";

    // Scopes OIDC
    options.Scope.Clear();
    options.Scope.Add("openid");      // Obligatorio para OIDC
    options.Scope.Add("profile");     // Nombre, foto, etc.
    options.Scope.Add("email");       // Email del usuario
    options.Scope.Add("calendar.read"); // Recurso adicional

    options.GetClaimsFromUserInfoEndpoint = true;
    options.SaveTokens = true;
});
```

Ahora sí funcionaba:

```csharp
[Authorize]
public IActionResult Profile()
{
    var userName = User.Identity.Name; // "Juan Pérez"
    var email = User.FindFirst(ClaimTypes.Email)?.Value; // "juan@ejemplo.com"
    var picture = User.FindFirst("picture")?.Value; // URL de foto

    return View(new { Name = userName, Email = email, Picture = picture });
}
```

## Los tres tipos de tokens en OIDC

OIDC introduce conceptos que OAuth 2.0 no tiene:

### 1. ID Token

Token JWT que contiene información del usuario:

```json
{
  "iss": "https://auth.ejemplo.com",
  "sub": "248289761001",
  "aud": "client-id",
  "exp": 1516239022,
  "iat": 1516239022,
  "name": "Juan Pérez",
  "email": "juan@ejemplo.com",
  "picture": "https://ejemplo.com/juan.jpg"
}
```

**Importante**: El ID Token NO se usa para llamar APIs. Es solo para identificar al usuario.

### 2. Access Token

Token opaco (o JWT) para acceder a recursos protegidos:

```csharp
// Usar el access token para llamar a una API
var accessToken = await HttpContext.GetTokenAsync("access_token");

var client = new HttpClient();
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", accessToken);

var response = await client.GetAsync("https://api.ejemplo.com/calendar");
```

### 3. Refresh Token

Token para obtener nuevos access tokens sin re-autenticar al usuario.

## Flows de OIDC

OIDC hereda los flows de OAuth 2.0 pero añade el ID Token:

### Authorization Code Flow (el recomendado)

```
1. Usuario → App: "Quiero iniciar sesión"
2. App → Auth Server: "Redirige usuario para autenticación"
3. Usuario → Auth Server: Ingresa credenciales
4. Auth Server → App: Devuelve código de autorización
5. App → Auth Server: Cambia código por tokens
6. Auth Server → App: Devuelve ID Token + Access Token
7. App: Valida ID Token, identifica usuario
```

```csharp
// ASP.NET Core hace esto automáticamente con OpenIdConnect
options.ResponseType = "code"; // Authorization Code Flow
```

### Implicit Flow (deprecated, no usar)

```csharp
// NO HAGAS ESTO
options.ResponseType = "id_token token";
```

Antes se recomendaba para SPAs. Ya no. Usar Authorization Code + PKCE.

## Validación del ID Token

**Crítico**: Siempre validar el ID Token antes de confiar en él.

```csharp
// ASP.NET Core lo hace automáticamente, pero así es como funciona
public async Task<ClaimsPrincipal> ValidateIdToken(string idToken)
{
    var tokenHandler = new JwtSecurityTokenHandler();

    var validationParameters = new TokenValidationParameters
    {
        ValidIssuer = "https://auth.ejemplo.com",
        ValidAudience = "client-id",
        IssuerSigningKey = await GetSigningKeyAsync(),

        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true
    };

    var principal = tokenHandler.ValidateToken(idToken, validationParameters, out var validatedToken);
    return principal;
}
```

## UserInfo Endpoint

OIDC define un endpoint estándar para obtener más información del usuario:

```csharp
// Si necesitas más información del usuario
var userInfoClient = new HttpClient();
var accessToken = await HttpContext.GetTokenAsync("access_token");

userInfoClient.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", accessToken);

var response = await userInfoClient.GetAsync("https://auth.ejemplo.com/connect/userinfo");
var userInfo = await response.Content.ReadAsAsync<UserInfoResponse>();

// userInfo contiene claims adicionales no incluidos en el ID Token
```

## Problemas comunes que enfrenté

### 1. No solicitar scope "openid"

```csharp
// MAL - No es OIDC sin "openid"
options.Scope.Add("profile");
options.Scope.Add("email");

// BIEN
options.Scope.Add("openid");  // Primero!
options.Scope.Add("profile");
options.Scope.Add("email");
```

Sin `openid`, no recibes ID Token y no es realmente OIDC.

### 2. Confundir ID Token con Access Token

```csharp
// MAL - Usar ID Token para llamar APIs
var idToken = await HttpContext.GetTokenAsync("id_token");
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", idToken); // ERROR!

// BIEN - Usar Access Token
var accessToken = await HttpContext.GetTokenAsync("access_token");
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", accessToken);
```

### 3. No validar el nonce

El nonce previene ataques de replay:

```csharp
// ASP.NET Core lo maneja automáticamente
options.ProtocolValidator = new OpenIdConnectProtocolValidator
{
    RequireNonce = true // Por defecto, pero asegúrate
};
```

## Herramientas de debugging

### jwt.io

Para inspeccionar ID Tokens:

```
1. Copia el ID Token
2. Pega en https://jwt.io
3. Verifica claims y firma
```

### Fiddler/Postman

Para ver el flujo completo:

```
1. Captura requests OIDC
2. Inspecciona parámetros (response_type, scope, etc.)
3. Verifica respuestas (código, tokens)
```

## Conclusión

OpenID Connect es OAuth 2.0 + autenticación estandarizada. Si necesitas saber QUIÉN es el usuario, necesitas OIDC, no solo OAuth 2.0.

La mejor manera de aprenderlo es implementarlo. Las librerías modernas (como Microsoft.AspNetCore.Authentication.OpenIdConnect) hacen el trabajo pesado, pero entender qué pasa por debajo es crucial.

¿Usas OpenID Connect? ¿Qué problemas has encontrado?
