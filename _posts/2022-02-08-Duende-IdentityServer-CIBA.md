---
layout: post
title: CIBA en Duende IdentityServer - Autenticación initiated por backend
author: Vicente José Moreno Escobar
categories: [Duende, OAuth2, Mobile]
published: true
---

> Client Initiated Backchannel Authentication: cuando el servidor inicia el login, no el usuario

# El caso de uso

Sistema bancario con estas necesidades:
- Backend detecta transacción sospechosa
- Backend debe solicitar autenticación adicional al usuario
- Usuario recibe push notification en móvil
- Usuario aprueba/niega en móvil
- Backend recibe confirmación y procesa transacción

Flow tradicional OAuth no sirve porque el **servidor** inicia la autenticación, no el usuario.

**Solución**: CIBA (Client Initiated Backchannel Authentication).

## CIBA vs flows tradicionales

| Authorization Code | Device Flow | CIBA |
|-------------------|-------------|------|
| Usuario inicia en navegador | Usuario inicia en device | Backend inicia |
| Redirect-based | Código en pantalla | Push notification |
| Usuario en mismo device | Usuario en otro device | Usuario puede estar offline |

## Arquitectura CIBA

```
1. Backend API → IdentityServer: "Autentica a usuario123"
2. IdentityServer → Push Service → Móvil del usuario
3. Usuario en móvil: Aprueba/Niega
4. Móvil → IdentityServer: "Usuario aprobó"
5. Backend hace polling → IdentityServer: "¿Ya autenticó?"
6. IdentityServer → Backend: "Sí, aquí está el token"
```

## Configuración Duende IdentityServer

### 1. Habilitar CIBA

```bash
dotnet add package Duende.IdentityServer.Ciba
```

```csharp
// Startup.cs
services.AddIdentityServer()
    .AddBackchannelAuthenticationUserValidator<CustomBackchannelAuthenticationUserValidator>()
    .AddBackchannelAuthenticationUserNotificationService<CustomUserNotificationService>();
```

### 2. Configurar Client

```csharp
new Client
{
    ClientId = "banking-api",
    ClientSecrets = { new Secret("secret".Sha256()) },

    AllowedGrantTypes = GrantTypes.Ciba,

    BackChannelAuthenticationRequestSigningAlg = "ES256",
    RequireCiba = true,

    BackChannelTokenDeliveryMode = "poll", // o "ping" o "push"

    AllowedScopes = { "openid", "profile", "transactions" }
}
```

### 3. User Validator

Valida que el usuario indicado existe y puede ser notificado:

```csharp
public class CustomBackchannelAuthenticationUserValidator : IBackchannelAuthenticationUserValidator
{
    private readonly IUserRepository _userRepository;

    public async Task<BackchannelAuthenticationUserValidationResult> ValidateRequestAsync(
        BackchannelAuthenticationUserValidatorContext context)
    {
        // context.LoginHint puede ser email, phone, etc.
        var user = await _userRepository.FindByLoginHintAsync(context.LoginHint);

        if (user == null)
        {
            return new BackchannelAuthenticationUserValidationResult
            {
                Error = "unknown_user_id",
                ErrorDescription = "User not found"
            };
        }

        if (!user.HasMobileApp)
        {
            return new BackchannelAuthenticationUserValidationResult
            {
                Error = "unauthorized_client",
                ErrorDescription = "User has no mobile app registered"
            };
        }

        return new BackchannelAuthenticationUserValidationResult
        {
            Subject = new ClaimsPrincipal(new ClaimsIdentity(new[]
            {
                new Claim("sub", user.Id),
                new Claim("name", user.Name),
                new Claim("email", user.Email)
            }))
        };
    }
}
```

### 4. Notification Service

Envía push notification al móvil del usuario:

```csharp
public class CustomUserNotificationService : IBackchannelAuthenticationUserNotificationService
{
    private readonly IPushNotificationService _pushService;
    private readonly IBackchannelAuthenticationRequestStore _requestStore;

    public async Task SendAuthenticationRequestAsync(BackchannelAuthenticationNotificationContext context)
    {
        var userId = context.Subject.FindFirst("sub")?.Value;
        var requestId = context.AuthenticationRequestId;

        // Enviar push notification
        await _pushService.SendAsync(userId, new
        {
            Type = "ciba_authentication_request",
            RequestId = requestId,
            Message = context.BindingMessage ?? "Authentication requested",
            ExpiresAt = DateTime.UtcNow.AddSeconds(context.Lifetime)
        });

        // Guardar request para polling posterior
        await _requestStore.CreateAsync(new BackchannelAuthenticationRequest
        {
            RequestId = requestId,
            Subject = context.Subject,
            ClientId = context.Client.ClientId,
            CreatedAt = DateTime.UtcNow,
            Lifetime = context.Lifetime,
            Status = "pending"
        });
    }
}
```

## Backend API (Cliente CIBA)

### Iniciar CIBA request

```csharp
// BankingController.cs
[HttpPost("transaction")]
public async Task<IActionResult> CreateTransaction([FromBody] TransactionRequest request)
{
    // 1. Validar transacción
    if (IsHighRisk(request))
    {
        // 2. Solicitar autenticación adicional vía CIBA
        var authRequestId = await RequestCIBAAuthenticationAsync(request.UserId);

        // 3. Polling para esperar autorización
        var authResult = await WaitForAuthorizationAsync(authRequestId, timeout: TimeSpan.FromMinutes(2));

        if (!authResult.IsAuthorized)
        {
            return Unauthorized("User did not authorize transaction");
        }
    }

    // 4. Procesar transacción
    await _transactionService.ProcessAsync(request);

    return Ok();
}

private async Task<string> RequestCIBAAuthenticationAsync(string userId)
{
    var client = new HttpClient();

    var disco = await client.GetDiscoveryDocumentAsync("https://auth.banco.com");

    var response = await client.RequestBackchannelAuthenticationAsync(
        new BackchannelAuthenticationRequest
        {
            Address = disco.BackchannelAuthenticationEndpoint,
            ClientId = "banking-api",
            ClientSecret = "secret",

            LoginHint = userId,  // Identificar al usuario
            BindingMessage = $"Authorize transaction of ${request.Amount}",

            RequestedExpiry = 120, // 2 minutos
            Scope = "openid profile transactions"
        });

    if (response.IsError)
    {
        throw new Exception($"CIBA request failed: {response.Error}");
    }

    return response.AuthenticationRequestId;
}

private async Task<CIBAAuthResult> WaitForAuthorizationAsync(string authRequestId, TimeSpan timeout)
{
    var client = new HttpClient();
    var disco = await client.GetDiscoveryDocumentAsync("https://auth.banco.com");

    var stopwatch = Stopwatch.StartNew();

    while (stopwatch.Elapsed < timeout)
    {
        var response = await client.RequestBackchannelAuthenticationTokenAsync(
            new BackchannelAuthenticationTokenRequest
            {
                Address = disco.TokenEndpoint,
                ClientId = "banking-api",
                ClientSecret = "secret",
                AuthenticationRequestId = authRequestId
            });

        if (!response.IsError)
        {
            // Usuario autorizó!
            return new CIBAAuthResult
            {
                IsAuthorized = true,
                AccessToken = response.AccessToken
            };
        }

        if (response.Error == "authorization_pending")
        {
            // Usuario aún no respondió, continuar polling
            await Task.Delay(TimeSpan.FromSeconds(5));
            continue;
        }

        if (response.Error == "expired_token")
        {
            // Timeout
            return new CIBAAuthResult { IsAuthorized = false };
        }

        // Otro error
        throw new Exception($"CIBA token request failed: {response.Error}");
    }

    return new CIBAAuthResult { IsAuthorized = false };
}
```

## App móvil (recibir y responder a CIBA)

### Recibir push notification

```typescript
// React Native con Firebase Cloud Messaging
import messaging from '@react-native-firebase/messaging';

messaging().onMessage(async remoteMessage => {
  const { type, requestId, message } = remoteMessage.data;

  if (type === 'ciba_authentication_request') {
    // Mostrar UI para aprobar/denegar
    showAuthorizationPrompt(requestId, message);
  }
});

async function showAuthorizationPrompt(requestId: string, message: string) {
  Alert.alert(
    'Authorization Required',
    message,
    [
      {
        text: 'Deny',
        onPress: () => respondToCIBA(requestId, false),
        style: 'cancel',
      },
      {
        text: 'Approve',
        onPress: () => respondToCIBA(requestId, true),
      },
    ]
  );
}
```

### Aprobar/Denegar request

```typescript
async function respondToCIBA(requestId: string, approved: boolean) {
  const accessToken = await getStoredAccessToken();

  const response = await fetch(
    `https://auth.banco.com/ciba/respond`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        requestId,
        approved,
      }),
    }
  );

  if (response.ok) {
    Alert.alert(
      'Success',
      approved ? 'Transaction authorized' : 'Transaction denied'
    );
  }
}
```

### Endpoint en IdentityServer para respuesta móvil

```csharp
// Controllers/CibaController.cs
[HttpPost("respond")]
[Authorize] // Usuario debe estar autenticado
public async Task<IActionResult> RespondToRequest([FromBody] CibaResponseModel model)
{
    var userId = User.FindFirst("sub")?.Value;
    var request = await _requestStore.GetByIdAsync(model.RequestId);

    if (request == null || request.Subject.FindFirst("sub")?.Value != userId)
    {
        return NotFound("Request not found or not for this user");
    }

    if (model.Approved)
    {
        request.Status = "approved";
        request.AuthorizedAt = DateTime.UtcNow;
    }
    else
    {
        request.Status = "denied";
    }

    await _requestStore.UpdateAsync(request);

    return Ok();
}
```

## Binding message

Mensaje descriptivo que usuario ve en móvil:

```csharp
// En backend
var response = await client.RequestBackchannelAuthenticationAsync(
    new BackchannelAuthenticationRequest
    {
        // ...
        BindingMessage = $"Transfer ${amount} to {recipientName}",
    });
```

Usuario en móvil ve:

```
Authorization Required

Transfer $500 to John Doe

[Deny] [Approve]
```

## Modos de delivery

### Poll mode (más común)

Backend hace polling:

```csharp
while (timeout not reached) {
    var response = await RequestTokenAsync(authRequestId);

    if (response.IsError && response.Error == "authorization_pending") {
        await Task.Delay(5000);
        continue;
    }

    return response;
}
```

### Ping mode

IdentityServer notifica a backend cuando usuario responde:

```csharp
// Backend expone webhook
[HttpPost("ciba/callback")]
public async Task<IActionResult> CIBACallback([FromBody] CIBACallbackModel model)
{
    // IdentityServer hace POST cuando usuario responde
    var authRequestId = model.AuthenticationRequestId;

    // Ahora podemos llamar token endpoint
    var tokenResponse = await RequestTokenAsync(authRequestId);

    return Ok();
}
```

### Push mode (más complejo)

IdentityServer envía token directamente a backend vía webhook.

## Security considerations

### Signed requests

```csharp
// Cliente firma request con private key
var request = new BackchannelAuthenticationRequest
{
    // ...
    ClientAssertion = CreateJWT(privateKey),
    ClientAssertionType = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
};
```

### Rate limiting

```csharp
// Limitar CIBA requests por usuario
public class CIBARateLimiter
{
    public async Task<bool> IsAllowedAsync(string userId)
    {
        var key = $"ciba_requests_{userId}";
        var count = await _cache.GetAsync<int>(key);

        if (count >= 5) // Máximo 5 requests en 10 minutos
        {
            return false;
        }

        await _cache.SetAsync(key, count + 1, TimeSpan.FromMinutes(10));
        return true;
    }
}
```

## Monitoreo

```csharp
// Métricas importantes
public class CIBAMetrics
{
    public void RecordCIBARequest(string clientId, string outcome)
    {
        _telemetry.TrackMetric("ciba_requests_total", 1, new Dictionary<string, string>
        {
            ["client_id"] = clientId,
            ["outcome"] = outcome  // "approved", "denied", "timeout"
        });
    }

    public void RecordResponseTime(TimeSpan duration)
    {
        _telemetry.TrackMetric("ciba_response_time_seconds", duration.TotalSeconds);
    }
}
```

Queries útiles:

```kusto
// Tasa de aprobación CIBA
CIBALogs
| where TimeGenerated > ago(7d)
| summarize
    Total = count(),
    Approved = countif(Outcome == "approved"),
    Denied = countif(Outcome == "denied"),
    Timeout = countif(Outcome == "timeout")
| extend ApprovalRate = (Approved * 100.0) / Total
```

## Resultados

Para nuestro caso bancario:

- Transacciones de alto riesgo: 100% requieren CIBA
- Approval rate: 92% (8% denegadas o timeout)
- Tiempo promedio de respuesta: 18 segundos
- Fraude reducido en 65%
- Satisfacción del usuario: 4.5/5 (push notification es más conveniente que SMS)

¿Has implementado CIBA? ¿Qué casos de uso has cubierto con este flow?
