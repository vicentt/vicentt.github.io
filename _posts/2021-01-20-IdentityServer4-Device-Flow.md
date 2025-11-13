---
layout: post
title: Device Authorization Flow - OAuth para dispositivos sin navegador
author: Vicente José Moreno Escobar
categories: [OAuth2, IdentityServer, IoT]
published: true
---

> Cuando tu Smart TV o IoT device necesita autenticarse

# El problema

Dispositivos sin navegador web completo:
- Smart TVs
- Consolas de juegos
- Dispositivos IoT
- CLI tools

No pueden hacer redirect OAuth tradicional. ¿Solución? **Device Authorization Flow**.

## ¿Cómo funciona?

```
1. Device → Auth Server: "Quiero autenticarme"
2. Auth Server → Device: "Muestra este código: ABCD-1234"
3. Usuario con su móvil → https://login.com/device
4. Usuario ingresa código: ABCD-1234
5. Usuario se autentica en móvil
6. Device hace polling → Auth Server: "¿Ya me autorizaron?"
7. Auth Server → Device: "Sí, aquí están tus tokens"
```

El usuario usa su teléfono/PC para autenticarse, el device solo muestra un código.

## Implementación IdentityServer 4

### 1. Habilitar Device Flow

```csharp
// Startup.cs
services.AddIdentityServer()
    .AddInMemoryClients(Config.Clients)
    .AddInMemoryApiScopes(Config.ApiScopes)
    .AddInMemoryDeviceFlowStores(); // Almacenar códigos de device
```

### 2. Configurar Client

```csharp
// Config.cs
new Client
{
    ClientId = "smart-tv",
    ClientName = "Smart TV Application",

    AllowedGrantTypes = GrantTypes.DeviceFlow,

    RequireClientSecret = false, // Device públic client

    AllowedScopes = { "openid", "profile", "api1" },

    // Device flow settings
    AllowOfflineAccess = true, // Refresh tokens útiles para devices

    DeviceCodeLifetime = 300, // Código válido por 5 minutos

    // Polling interval (segundos)
    // Cliente debe esperar mínimo este tiempo entre polls
    PollingInterval = 5
}
```

### 3. Device Flow Endpoint

IdentityServer expone automáticamente:

```
POST /connect/deviceauthorization
POST /connect/token
```

## Cliente Device (Smart TV ejemplo)

```csharp
// SmartTVApp/DeviceAuthService.cs
using IdentityModel.Client;

public class DeviceAuthService
{
    private readonly HttpClient _httpClient;
    private const string AuthorityUrl = "https://auth.ejemplo.com";

    public async Task<DeviceAuthResult> StartAuthorizationAsync()
    {
        // 1. Solicitar device code
        var disco = await _httpClient.GetDiscoveryDocumentAsync(AuthorityUrl);

        var response = await _httpClient.RequestDeviceAuthorizationAsync(
            new DeviceAuthorizationRequest
            {
                Address = disco.DeviceAuthorizationEndpoint,
                ClientId = "smart-tv",
                Scope = "openid profile api1 offline_access"
            });

        if (response.IsError)
        {
            throw new Exception($"Error: {response.Error}");
        }

        // 2. Retornar info para mostrar al usuario
        return new DeviceAuthResult
        {
            DeviceCode = response.DeviceCode,
            UserCode = response.UserCode,
            VerificationUri = response.VerificationUri,
            VerificationUriComplete = response.VerificationUriComplete, // URL con código embebido
            ExpiresIn = response.ExpiresIn,
            Interval = response.Interval
        };
    }

    public async Task<TokenResponse> PollForTokenAsync(string deviceCode, int interval)
    {
        var disco = await _httpClient.GetDiscoveryDocumentAsync(AuthorityUrl);

        while (true)
        {
            var response = await _httpClient.RequestDeviceTokenAsync(
                new DeviceTokenRequest
                {
                    Address = disco.TokenEndpoint,
                    ClientId = "smart-tv",
                    DeviceCode = deviceCode
                });

            if (response.IsError)
            {
                if (response.Error == "authorization_pending")
                {
                    // Usuario aún no ha autorizado, esperar y reintentar
                    await Task.Delay(interval * 1000);
                    continue;
                }

                if (response.Error == "slow_down")
                {
                    // Estamos haciendo polling muy rápido
                    interval += 5;
                    await Task.Delay(interval * 1000);
                    continue;
                }

                // Otro error (expired_token, access_denied, etc.)
                throw new Exception($"Error: {response.Error}");
            }

            // Success!
            return response;
        }
    }
}
```

## UI en Smart TV

```csharp
// SmartTVApp/LoginScreen.cs
public class LoginScreen
{
    private readonly DeviceAuthService _authService;

    public async Task ShowLoginAsync()
    {
        // Iniciar flow
        var authResult = await _authService.StartAuthorizationAsync();

        // Mostrar en pantalla
        Console.WriteLine("╔══════════════════════════════════════╗");
        Console.WriteLine("║   Para iniciar sesión, visita:      ║");
        Console.WriteLine("║                                      ║");
        Console.WriteLine($"║   {authResult.VerificationUri,-36}║");
        Console.WriteLine("║                                      ║");
        Console.WriteLine($"║   E ingresa el código: {authResult.UserCode,-14}║");
        Console.WriteLine("╚══════════════════════════════════════╝");

        // También mostrar QR code con VerificationUriComplete
        ShowQRCode(authResult.VerificationUriComplete);

        // Polling en background
        try
        {
            var tokenResponse = await _authService.PollForTokenAsync(
                authResult.DeviceCode,
                authResult.Interval
            );

            // Guardar tokens
            await SaveTokensAsync(tokenResponse);

            Console.WriteLine("✅ Login exitoso!");
            NavigateToHomeScreen();
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ Login falló: {ex.Message}");
        }
    }
}
```

## Verification Endpoint (Web)

Usuario visita esta página desde su móvil/PC:

```csharp
// Controllers/DeviceController.cs
[HttpGet]
public async Task<IActionResult> Index(string userCode = null)
{
    var vm = new DeviceAuthorizationViewModel
    {
        UserCode = userCode
    };

    return View(vm);
}

[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> VerifyCode(string userCode)
{
    // Buscar device authorization request
    var request = await _deviceFlowStore.FindByUserCodeAsync(userCode);

    if (request == null)
    {
        ModelState.AddModelError("", "Código inválido o expirado");
        return View("Index");
    }

    // Mostrar pantalla de consentimiento
    return View("Consent", new ConsentViewModel
    {
        UserCode = userCode,
        ClientName = request.ClientId,
        Scopes = request.RequestedScopes
    });
}

[HttpPost]
[Authorize] // Usuario debe estar autenticado
[ValidateAntiForgeryToken]
public async Task<IActionResult> Authorize(string userCode, bool allow)
{
    var request = await _deviceFlowStore.FindByUserCodeAsync(userCode);

    if (request == null)
    {
        return View("Error");
    }

    if (allow)
    {
        // Usuario autorizó, guardar
        request.Subject = User;
        request.IsAuthorized = true;

        await _deviceFlowStore.UpdateByUserCodeAsync(userCode, request);

        return View("Success");
    }
    else
    {
        // Usuario denegó
        request.IsAuthorized = false;
        await _deviceFlowStore.UpdateByUserCodeAsync(userCode, request);

        return View("Denied");
    }
}
```

## Vista de verificación

```html
<!-- Views/Device/Index.cshtml -->
@model DeviceAuthorizationViewModel

<div class="device-verification">
    <h1>Autorización de Dispositivo</h1>

    @if (Model.UserCode != null)
    {
        <p>Verificando código: <strong>@Model.UserCode</strong></p>
        <form asp-action="VerifyCode" method="post">
            <input type="hidden" name="userCode" value="@Model.UserCode" />
            <button type="submit" class="btn btn-primary">Continuar</button>
        </form>
    }
    else
    {
        <p>Ingresa el código mostrado en tu dispositivo:</p>
        <form asp-action="VerifyCode" method="post">
            <div class="form-group">
                <input type="text"
                       name="userCode"
                       class="form-control code-input"
                       placeholder="XXXX-XXXX"
                       maxlength="9"
                       autofocus />
            </div>
            <button type="submit" class="btn btn-primary">Verificar</button>
        </form>
    }
</div>
```

## Mejoras UX

### 1. QR Code para verificación rápida

```csharp
// Generar QR con VerificationUriComplete
using QRCoder;

public byte[] GenerateQRCode(string url)
{
    var qrGenerator = new QRCodeGenerator();
    var qrCodeData = qrGenerator.CreateQrCode(url, QRCodeGenerator.ECCLevel.Q);
    var qrCode = new PngByteQRCode(qrCodeData);
    return qrCode.GetGraphic(20);
}

// En Smart TV
var qrImage = GenerateQRCode(authResult.VerificationUriComplete);
DisplayQROnScreen(qrImage);
```

Usuario escanea QR → Abre URL con código pre-rellenado → Más rápido.

### 2. Formateo del código

```csharp
// Generar código legible
public string GenerateUserCode()
{
    var random = new Random();
    var part1 = random.Next(1000, 9999);
    var part2 = random.Next(1000, 9999);

    return $"{part1}-{part2}"; // Ej: 4523-8192
}
```

### 3. Auto-refresh en página de verificación

```javascript
// En la página web de verificación
// Actualizar automáticamente cuando usuario ya autenticó
<script>
let checkInterval = setInterval(async () => {
    const response = await fetch(`/device/check?userCode=@Model.UserCode`);
    const data = await response.json();

    if (data.isAuthorized) {
        clearInterval(checkInterval);
        window.location.href = '/device/success';
    } else if (data.isExpired) {
        clearInterval(checkInterval);
        window.location.href = '/device/expired';
    }
}, 3000); // Check cada 3 segundos
</script>
```

## Seguridad

### 1. Rate limiting

```csharp
// Prevenir brute force de user codes
public class DeviceCodeRateLimiter
{
    private readonly IDistributedCache _cache;
    private const int MaxAttempts = 5;
    private const int WindowMinutes = 15;

    public async Task<bool> IsAllowedAsync(string ipAddress)
    {
        var key = $"device_attempts_{ipAddress}";
        var attemptsStr = await _cache.GetStringAsync(key);

        int attempts = string.IsNullOrEmpty(attemptsStr) ? 0 : int.Parse(attemptsStr);

        if (attempts >= MaxAttempts)
        {
            return false;
        }

        await _cache.SetStringAsync(key, (attempts + 1).ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(WindowMinutes)
            });

        return true;
    }
}
```

### 2. Códigos de un solo uso

```csharp
// Device code solo se puede usar una vez
public async Task<bool> ConsumeDeviceCodeAsync(string deviceCode)
{
    var request = await _deviceFlowStore.FindByDeviceCodeAsync(deviceCode);

    if (request == null || request.IsConsumed)
    {
        return false;
    }

    request.IsConsumed = true;
    await _deviceFlowStore.UpdateAsync(request);

    return true;
}
```

## CLI Tool ejemplo

Caso de uso común: CLI que necesita acceder a APIs:

```csharp
// azure-cli-style tool
class Program
{
    static async Task Main(string[] args)
    {
        if (args[0] == "login")
        {
            await LoginAsync();
        }
        else if (args[0] == "list-resources")
        {
            await ListResourcesAsync();
        }
    }

    static async Task LoginAsync()
    {
        var authService = new DeviceAuthService();
        var authResult = await authService.StartAuthorizationAsync();

        Console.WriteLine($"Para completar login, visita: {authResult.VerificationUri}");
        Console.WriteLine($"E ingresa el código: {authResult.UserCode}");

        var tokens = await authService.PollForTokenAsync(
            authResult.DeviceCode,
            authResult.Interval
        );

        // Guardar tokens en keychain/credential store
        await SecureStorage.SaveTokensAsync(tokens);

        Console.WriteLine("✅ Login exitoso!");
    }

    static async Task ListResourcesAsync()
    {
        var accessToken = await SecureStorage.GetAccessTokenAsync();

        if (accessToken == null)
        {
            Console.WriteLine("No estás autenticado. Ejecuta: mytool login");
            return;
        }

        // Usar token para llamar API
        var client = new HttpClient();
        client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", accessToken);

        var response = await client.GetAsync("https://api.ejemplo.com/resources");
        // ...
    }
}
```

## Problemas comunes

### 1. Polling demasiado agresivo

```
Error: slow_down
```

**Solución**: Respetar `interval` del response inicial, incrementar si recibes `slow_down`.

### 2. Código expirado

```
Error: expired_token
```

**Solución**: Reiniciar flow completo, generar nuevo código.

### 3. Usuario ingresa código incorrecto

Dar feedback claro:

```csharp
if (request == null)
{
    return View("InvalidCode", new { Message = "Código inválido o expirado. Verifica el código mostrado en tu dispositivo." });
}
```

## Resultados

Device Flow perfecto para:
- Smart TVs
- Dispositivos IoT
- CLIs
- Apps de consola

UX mucho mejor que intentar navegar en un teclado de TV.

¿Has implementado Device Flow? ¿Qué tipo de dispositivos autentican con él?
