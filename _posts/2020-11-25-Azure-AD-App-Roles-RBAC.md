---
layout: post
title: Azure AD App Roles - RBAC para aplicaciones empresariales
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD, Autorización]
published: true
---

> Cuando necesitas algo más sofisticado que "está autenticado o no"

# El problema

Aplicación con diferentes niveles de acceso:
- **Viewers**: Solo lectura
- **Contributors**: Lectura + escritura
- **Administrators**: Todo

Azure AD Authentication nos daba identidad, pero no autorización granular.

**Solución**: App Roles de Azure AD.

## App Roles vs Groups

| App Roles | Azure AD Groups |
|-----------|-----------------|
| Definidos en app manifest | Creados en Azure AD |
| Específicos de la app | Compartidos entre apps |
| Incluidos en token automáticamente | Requieren configuración adicional |
| Máximo 200 por app | Sin límite |

Para casos simples con <10 roles: **App Roles** son más fáciles.

## Configuración en Azure Portal

### 1. Definir App Roles

```
Azure AD → App registrations → Tu app → App roles → Create app role

Display name: Administrator
Value: admin
Description: Full administrative access
Allowed member types: Users/Groups

[Repeat para otros roles]
```

Roles definidos:
- `viewer` - Solo lectura
- `contributor` - Lectura y escritura
- `admin` - Acceso completo

### 2. Via App Manifest (más rápido)

```json
// App manifest
{
  "appRoles": [
    {
      "allowedMemberTypes": ["User"],
      "description": "Viewers can only read data",
      "displayName": "Viewer",
      "id": "e1a4c3e2-4f5e-4b3a-9c7d-8e2f1a3b5c7d",
      "isEnabled": true,
      "lang": null,
      "origin": "Application",
      "value": "viewer"
    },
    {
      "allowedMemberTypes": ["User"],
      "description": "Contributors can read and write data",
      "displayName": "Contributor",
      "id": "a2b4d6e8-1c3e-4f5a-9b7d-2e4f6a8c1e3b",
      "isEnabled": true,
      "value": "contributor"
    },
    {
      "allowedMemberTypes": ["User"],
      "description": "Administrators have full access",
      "displayName": "Administrator",
      "id": "b3c5e7f9-2d4f-5a6c-8b9d-3f5a7c9e2f4b",
      "isEnabled": true,
      "value": "admin"
    }
  ]
}
```

**Importante**: El `id` debe ser un GUID único. Generarlo con:

```powershell
[guid]::NewGuid()
```

### 3. Asignar usuarios a roles

```
Azure AD → Enterprise applications → Tu app → Users and groups
→ Add user/group

Select user: juan@empresa.com
Select role: Administrator
```

O vía PowerShell:

```powershell
Connect-AzureAD

$app = Get-AzureADServicePrincipal -Filter "displayName eq 'TuApp'"
$user = Get-AzureADUser -ObjectId "juan@empresa.com"

# Obtener AppRole ID
$adminRole = $app.AppRoles | Where-Object {$_.Value -eq "admin"}

# Asignar
New-AzureADUserAppRoleAssignment `
    -ObjectId $user.ObjectId `
    -PrincipalId $user.ObjectId `
    -ResourceId $app.ObjectId `
    -Id $adminRole.Id
```

## Configuración ASP.NET Core

### 1. Recibir roles en token

```csharp
// Startup.cs
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(options =>
    {
        options.Instance = "https://login.microsoftonline.com/";
        options.TenantId = Configuration["AzureAd:TenantId"];
        options.ClientId = Configuration["AzureAd:ClientId"];
        options.ClientSecret = Configuration["AzureAd:ClientSecret"];

        // Importante: mapear roles claim
        options.TokenValidationParameters.RoleClaimType = "roles";
    });
```

### 2. Configurar autorización por roles

```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("ViewerPolicy", policy =>
        policy.RequireRole("viewer", "contributor", "admin"));

    options.AddPolicy("ContributorPolicy", policy =>
        policy.RequireRole("contributor", "admin"));

    options.AddPolicy("AdminPolicy", policy =>
        policy.RequireRole("admin"));
});
```

### 3. Proteger endpoints

```csharp
// Controllers/DataController.cs
[Authorize(Policy = "ViewerPolicy")]
[HttpGet]
public IActionResult GetData()
{
    var data = _service.GetData();
    return Ok(data);
}

[Authorize(Policy = "ContributorPolicy")]
[HttpPost]
public IActionResult CreateData([FromBody] DataModel data)
{
    _service.CreateData(data);
    return Created("", data);
}

[Authorize(Policy = "AdminPolicy")]
[HttpDelete("{id}")]
public IActionResult DeleteData(int id)
{
    _service.DeleteData(id);
    return NoContent();
}
```

O con atributo directo:

```csharp
[Authorize(Roles = "admin,contributor")]
public IActionResult SomeAction() { }
```

## Verificar roles en código

```csharp
public class DataService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public void ProcessData()
    {
        var user = _httpContextAccessor.HttpContext.User;

        if (user.IsInRole("admin"))
        {
            // Lógica para admins
            ProcessFullData();
        }
        else if (user.IsInRole("contributor"))
        {
            // Lógica para contributors
            ProcessLimitedData();
        }
        else
        {
            // Viewer
            ProcessReadOnlyData();
        }
    }

    // O más limpio con pattern matching
    public void ProcessDataClean()
    {
        var roles = _httpContextAccessor.HttpContext.User
            .FindAll("roles")
            .Select(c => c.Value)
            .ToList();

        if (roles.Contains("admin"))
        {
            ProcessFullData();
        }
        // ...
    }
}
```

## Roles en React SPA

### 1. Obtener roles del token

```typescript
// src/authConfig.ts
import { Configuration } from '@azure/msal-browser';

export const msalConfig: Configuration = {
  auth: {
    clientId: 'tu-client-id',
    authority: 'https://login.microsoftonline.com/tu-tenant-id',
    redirectUri: 'http://localhost:3000',
  },
};

export const loginRequest = {
  scopes: ['User.Read'],
};
```

```typescript
// src/services/AuthService.ts
import { PublicClientApplication } from '@azure/msal-browser';
import { msalConfig } from '../authConfig';

const msalInstance = new PublicClientApplication(msalConfig);

export const getUserRoles = (): string[] => {
  const accounts = msalInstance.getAllAccounts();

  if (accounts.length === 0) {
    return [];
  }

  const account = accounts[0];
  const roles = account.idTokenClaims?.roles || [];

  return roles as string[];
};

export const hasRole = (role: string): boolean => {
  const roles = getUserRoles();
  return roles.includes(role);
};

export const hasAnyRole = (requiredRoles: string[]): boolean => {
  const userRoles = getUserRoles();
  return requiredRoles.some(role => userRoles.includes(role));
};
```

### 2. Conditional rendering por rol

```typescript
// src/components/ProtectedButton.tsx
import React from 'react';
import { hasRole } from '../services/AuthService';

interface Props {
  requiredRole: string;
  onClick: () => void;
}

export const ProtectedButton: React.FC<Props> = ({ requiredRole, onClick, children }) => {
  if (!hasRole(requiredRole)) {
    return null;
  }

  return <button onClick={onClick}>{children}</button>;
};

// Uso
<ProtectedButton requiredRole="admin" onClick={handleDelete}>
  Delete
</ProtectedButton>
```

### 3. Route protection

```typescript
// src/components/ProtectedRoute.tsx
import React from 'react';
import { Route, Redirect } from 'react-router-dom';
import { hasAnyRole } from '../services/AuthService';

interface Props {
  component: React.ComponentType<any>;
  requiredRoles: string[];
  path: string;
}

export const ProtectedRoute: React.FC<Props> = ({ component: Component, requiredRoles, ...rest }) => {
  return (
    <Route
      {...rest}
      render={props =>
        hasAnyRole(requiredRoles) ? (
          <Component {...props} />
        ) : (
          <Redirect to="/unauthorized" />
        )
      }
    />
  );
};

// Uso
<ProtectedRoute
  path="/admin"
  component={AdminPanel}
  requiredRoles={['admin']}
/>
```

## Roles jerárquicos

Para implementar herencia (admin tiene permisos de contributor):

```csharp
// Services/RoleService.cs
public class RoleService
{
    private static readonly Dictionary<string, HashSet<string>> RoleHierarchy = new()
    {
        ["viewer"] = new HashSet<string> { "viewer" },
        ["contributor"] = new HashSet<string> { "viewer", "contributor" },
        ["admin"] = new HashSet<string> { "viewer", "contributor", "admin" }
    };

    public static bool HasPermission(ClaimsPrincipal user, string requiredRole)
    {
        var userRoles = user.FindAll("roles").Select(c => c.Value);

        foreach (var userRole in userRoles)
        {
            if (RoleHierarchy.TryGetValue(userRole, out var permissions))
            {
                if (permissions.Contains(requiredRole))
                {
                    return true;
                }
            }
        }

        return false;
    }
}

// Authorization handler custom
public class RoleRequirement : IAuthorizationRequirement
{
    public string Role { get; }

    public RoleRequirement(string role)
    {
        Role = role;
    }
}

public class HierarchicalRoleHandler : AuthorizationHandler<RoleRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        RoleRequirement requirement)
    {
        if (RoleService.HasPermission(context.User, requirement.Role))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Registrar
services.AddSingleton<IAuthorizationHandler, HierarchicalRoleHandler>();

services.AddAuthorization(options =>
{
    options.AddPolicy("RequireViewer", policy =>
        policy.Requirements.Add(new RoleRequirement("viewer")));

    options.AddPolicy("RequireContributor", policy =>
        policy.Requirements.Add(new RoleRequirement("contributor")));
});
```

## Auditoría de cambios de roles

```csharp
// Middleware para logging
public class RoleAuditMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RoleAuditMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.User.Identity?.IsAuthenticated == true)
        {
            var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            var roles = string.Join(", ", context.User.FindAll("roles").Select(c => c.Value));
            var endpoint = context.Request.Path;

            _logger.LogInformation(
                "User {UserId} with roles [{Roles}] accessed {Endpoint}",
                userId,
                roles,
                endpoint
            );
        }

        await _next(context);
    }
}

// Registrar
app.UseMiddleware<RoleAuditMiddleware>();
```

## Problemas comunes

### 1. Roles claim no aparece en token

**Causa**: Usuario no tiene rol asignado en Enterprise App.

**Debug**:
```csharp
var claims = User.Claims.Select(c => new { c.Type, c.Value });
// Verificar si "roles" está presente
```

### 2. Role claim type incorrecto

**Solución**:
```csharp
options.TokenValidationParameters.RoleClaimType = "roles"; // No "role"
```

### 3. Cambios de roles no reflejan inmediatamente

**Causa**: Token cacheado.

**Solución**: Forzar logout/login o esperar expiración del token.

## Mejores prácticas

1. **Mantén roles simples**: <5 roles es ideal
2. **Documenta permisos**: Qué puede hacer cada rol
3. **Usa policies en lugar de roles directos**: Más flexible
4. **Audita cambios**: Log quién modifica roles
5. **Testea permisos**: Unit tests para cada rol

## Resultados

Con App Roles implementados:
- Autorización centralizada en Azure AD
- 0 hardcoding de permisos
- Cambios de roles sin redeploy
- Auditoría completa en Azure AD logs

¿Usas App Roles o prefieres Azure AD Groups? ¿Por qué?
