---
layout: post
title: IdentityServer 4 - Refresh Tokens y sesiones de larga duración
author: Vicente José Moreno Escobar
categories: [IdentityServer, OAuth2, Seguridad]
published: true
---

> Implementando refresh tokens correctamente para apps móviles

# El problema

Apps móviles con access tokens de 1 hora. Usuarios tenían que re-autenticarse constantemente. Mala UX.

Refresh tokens eran la solución, pero implementarlos correctamente tiene sus matices.

## Refresh Tokens 101

**Access Token**: Vida corta (15-60 min), acceso a recursos
**Refresh Token**: Vida larga (días/meses), obtener nuevos access tokens

Flujo:
```
1. Usuario se autentica → Recibe access + refresh token
2. Access token expira
3. App usa refresh token → Obtiene nuevo access token
4. Si refresh token también expiró → Re-autenticación requerida
```

## Configuración en IdentityServer 4

```csharp
// Config.cs
new Client
{
    ClientId = "mobile-app",
    ClientName = "Mobile App",
    AllowedGrantTypes = GrantTypes.Code,
    RequirePkce = true,
    RequireClientSecret = false, // Public client (mobile)

    RedirectUris = { "myapp://callback" },
    PostLogoutRedirectUris = { "myapp://logout" },

    AllowedScopes =
    {
        IdentityServerConstants.StandardScopes.OpenId,
        IdentityServerConstants.StandardScopes.Profile,
        IdentityServerConstants.StandardScopes.OfflineAccess, // Esto habilita refresh tokens!
        "api1"
    },

    // Configuración de tokens
    AccessTokenLifetime = 3600, // 1 hora

    // Absolute: token muere después de X tiempo sin importar uso
    // Sliding: token se extiende con cada uso
    RefreshTokenExpiration = TokenExpiration.Sliding,

    // Sliding refresh token lifetime
    SlidingRefreshTokenLifetime = 1296000, // 15 días

    // Máximo lifetime absoluto
    AbsoluteRefreshTokenLifetime = 2592000, // 30 días

    // OneTimeOnly: cada refresh token solo se usa una vez (más seguro)
    // ReUse: mismo refresh token se puede reusar (menos seguro pero mejor UX)
    RefreshTokenUsage = TokenUsage.OneTimeOnly,

    UpdateAccessTokenClaimsOnRefresh = true // Actualizar claims al refresh
}
```

## Tipos de Refresh Token Expiration

### Sliding (recomendado para móviles)

```csharp
RefreshTokenExpiration = TokenExpiration.Sliding,
SlidingRefreshTokenLifetime = 1296000, // 15 días
```

Si el usuario usa la app cada 10 días, el refresh token nunca expira.
Si pasan 15 días sin usar la app, el refresh token expira.

### Absolute

```csharp
RefreshTokenExpiration = TokenExpiration.Absolute,
AbsoluteRefreshTokenLifetime = 2592000 // 30 días
```

Después de 30 días, refresh token expira sin importar cuánto lo hayas usado.

### Combinación (más común)

```csharp
RefreshTokenExpiration = TokenExpiration.Sliding,
SlidingRefreshTokenLifetime = 1296000, // 15 días inactivos
AbsoluteRefreshTokenLifetime = 2592000 // Máximo 30 días totales
```

## OneTimeOnly vs ReUse

### OneTimeOnly (más seguro)

```csharp
RefreshTokenUsage = TokenUsage.OneTimeOnly
```

Cada vez que usas un refresh token, obtienes un NUEVO refresh token y el anterior se invalida.

**Pros**: Más seguro contra token theft
**Contras**: Complejidad en apps con múltiples pestañas/procesos

### ReUse (mejor UX)

```csharp
RefreshTokenUsage = TokenUsage.ReUse
```

El mismo refresh token se puede usar múltiples veces.

**Pros**: Más simple, mejor para debugging
**Contras**: Si lo roban, es válido hasta que expire

### Mi recomendación

Para móviles: **OneTimeOnly** con sliding expiration.

## Cliente móvil (ejemplo React Native)

```typescript
// AuthService.ts
import {
  authorize,
  refresh,
  AuthConfiguration
} from 'react-native-app-auth';

const config: AuthConfiguration = {
  issuer: 'https://auth.ejemplo.com',
  clientId: 'mobile-app',
  redirectUrl: 'myapp://callback',
  scopes: ['openid', 'profile', 'api1', 'offline_access'],
  additionalParameters: {
    prompt: 'login'
  },
};

class AuthService {
  private refreshToken: string | null = null;

  async login() {
    const result = await authorize(config);

    // Guardar tokens de forma segura
    await SecureStore.setItemAsync('access_token', result.accessToken);
    await SecureStore.setItemAsync('refresh_token', result.refreshToken);
    this.refreshToken = result.refreshToken;

    return result;
  }

  async getAccessToken(): Promise<string> {
    // Intentar obtener token existente
    let accessToken = await SecureStore.getItemAsync('access_token');
    const tokenExpiry = await SecureStore.getItemAsync('token_expiry');

    // Verificar si expiró
    if (tokenExpiry && Date.now() > parseInt(tokenExpiry)) {
      // Token expirado, refrescar
      accessToken = await this.refreshAccessToken();
    }

    return accessToken!;
  }

  async refreshAccessToken(): Promise<string> {
    const storedRefreshToken = await SecureStore.getItemAsync('refresh_token');

    if (!storedRefreshToken) {
      throw new Error('No refresh token available');
    }

    try {
      const result = await refresh(config, {
        refreshToken: storedRefreshToken
      });

      // Guardar nuevos tokens
      await SecureStore.setItemAsync('access_token', result.accessToken);
      await SecureStore.setItemAsync('refresh_token', result.refreshToken);

      // Si es OneTimeOnly, el refresh token cambió
      this.refreshToken = result.refreshToken;

      const expiry = Date.now() + (result.accessTokenExpirationDate * 1000);
      await SecureStore.setItemAsync('token_expiry', expiry.toString());

      return result.accessToken;
    } catch (error) {
      // Refresh falló, forzar re-login
      await this.logout();
      throw new Error('Refresh token expired, please login again');
    }
  }

  async logout() {
    await SecureStore.deleteItemAsync('access_token');
    await SecureStore.deleteItemAsync('refresh_token');
    await SecureStore.deleteItemAsync('token_expiry');
  }
}
```

## Interceptor Axios para refresh automático

```typescript
// api/client.ts
import axios from 'axios';
import AuthService from '../services/AuthService';

const apiClient = axios.create({
  baseURL: 'https://api.ejemplo.com',
});

// Request interceptor: añadir token
apiClient.interceptors.request.use(
  async (config) => {
    const token = await AuthService.getAccessToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor: manejar token expirado
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Si es 401 y no hemos reintentado ya
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Intentar refresh
        const newToken = await AuthService.refreshAccessToken();
        originalRequest.headers.Authorization = `Bearer ${newToken}`;

        // Reintentar request original con nuevo token
        return apiClient(originalRequest);
      } catch (refreshError) {
        // Refresh falló, redirigir a login
        // navigation.navigate('Login');
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);

export default apiClient;
```

## Revocación de Refresh Tokens

Importante para logout y seguridad:

```csharp
// Startup.cs - Configurar revocation endpoint
services.AddIdentityServer()
    .AddOperationalStore(options =>
    {
        options.EnableTokenCleanup = true;
        options.TokenCleanupInterval = 3600; // 1 hora
    });
```

Revocar token al logout:

```csharp
// API endpoint custom para revocar
[HttpPost("revoke")]
public async Task<IActionResult> RevokeToken([FromBody] string refreshToken)
{
    await _persistedGrantStore.RemoveAllGrantsAsync(
        subjectId: User.GetSubjectId(),
        clientId: "mobile-app"
    );

    return Ok();
}
```

Cliente móvil:

```typescript
async logout() {
  const refreshToken = await SecureStore.getItemAsync('refresh_token');

  // Llamar endpoint de revocación
  if (refreshToken) {
    await apiClient.post('/auth/revoke', { refreshToken });
  }

  // Limpiar storage local
  await SecureStore.deleteItemAsync('access_token');
  await SecureStore.deleteItemAsync('refresh_token');
}
```

## Auditoría y monitoreo

Ver refresh token usage:

```sql
-- Queries útiles en OperationalStore DB
-- Refresh tokens activos por usuario
SELECT SubjectId, ClientId, CreationTime, Expiration
FROM PersistedGrants
WHERE Type = 'refresh_token'
AND Expiration > GETUTCDATE()
ORDER BY SubjectId, CreationTime DESC

-- Refresh tokens por cliente
SELECT ClientId, COUNT(*) as ActiveTokens
FROM PersistedGrants
WHERE Type = 'refresh_token'
AND Expiration > GETUTCDATE()
GROUP BY ClientId

-- Refresh tokens expirados sin limpiar
SELECT COUNT(*)
FROM PersistedGrants
WHERE Type = 'refresh_token'
AND Expiration < GETUTCDATE()
```

Logs en IdentityServer:

```csharp
// Startup.cs
services.AddIdentityServer(options =>
{
    options.Events.RaiseSuccessEvents = true;
    options.Events.RaiseFailureEvents = true;
})
.AddEventSink<CustomEventSink>();

public class CustomEventSink : IEventSink
{
    public async Task PersistAsync(Event evt)
    {
        if (evt.EventType == EventTypes.Failure &&
            evt.Name == "Token Issued")
        {
            // Log refresh token failures
            _logger.LogWarning("Refresh token failed: {details}", evt);
        }
    }
}
```

## Problemas comunes

### 1. Refresh token no se emite

**Causa**: Scope `offline_access` no solicitado

```csharp
// Asegurar que el scope está configurado
AllowedScopes = { ..., IdentityServerConstants.StandardScopes.OfflineAccess }
```

### 2. "Invalid grant" al refrescar

**Causas posibles**:
- Refresh token ya usado (con OneTimeOnly)
- Refresh token expirado
- Client ID incorrecto

**Debug**:
```csharp
// Verificar en DB
SELECT * FROM PersistedGrants
WHERE Key = 'tu-refresh-token'
```

### 3. Claims desactualizados después de refresh

```csharp
// Solución: actualizar claims
UpdateAccessTokenClaimsOnRefresh = true
```

## Mejores prácticas

1. **Usa sliding expiration** para móviles
2. **OneTimeOnly** para máxima seguridad
3. **Almacena tokens en secure storage** (Keychain/Keystore)
4. **Implementa auto-refresh** en interceptors
5. **Revoca tokens en logout**
6. **Monitorea refresh token usage** para detectar anomalías
7. **Limpia tokens expirados** regularmente

## Resultados

Con refresh tokens correctamente implementados:
- Usuarios solo re-autentican cada 15-30 días
- UX mejorada dramáticamente
- Seguridad mantenida con OneTimeOnly
- 0 quejas de "tengo que logearme todo el tiempo"

¿Usas refresh tokens en tus apps? ¿Qué configuración prefieres?
