---
layout: post
title: Azure AD Conditional Access - Seguridad adaptativa sin frustrar usuarios
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD, Seguridad]
published: true
---

> Implementando políticas de acceso condicional que mejoran seguridad sin arruinar UX

# El problema

Nos pedían endurecer la seguridad después de un intento de phishing exitoso. Las opciones tradicionales eran:

1. **MFA para todos siempre**: Seguro pero usuarios lo odiaban
2. **VPN obligatoria**: Complejo y caro
3. **Bloquear acceso externo**: No viable, teníamos remote workers

Entonces descubrimos Azure AD Conditional Access.

## ¿Qué es Conditional Access?

Políticas if-then para control de acceso:

```
IF (usuario desde ubicación desconocida)
   AND (dispositivo no gestionado)
   AND (aplicación es crítica)
THEN
   Require MFA + dispositivo compliant
```

En lugar de aplicar MFA ciegamente a todos, lo aplicas solo cuando el riesgo lo justifica.

## Nuestro primer intento (demasiado restrictivo)

Creamos una política que parecía sensata:

```
Nombre: "Bloquear acceso no corporativo"

Condiciones:
  - Todas las aplicaciones cloud
  - Todos los usuarios
  - Ubicaciones: Cualquiera EXCEPTO red corporativa

Controles:
  - Bloquear acceso
```

**Resultado**: Caos absoluto. Nadie podía trabajar desde casa.

## Políticas que realmente funcionaron

### Política 1: MFA para administradores

```
Nombre: "MFA obligatorio para admins"

Asignaciones:
  Usuarios: Grupo "Global Administrators"
  Aplicaciones cloud: Todas
  Condiciones: Ninguna

Controles de acceso:
  Conceder acceso
  Require MFA
```

Código para asignarla vía Graph API:

```csharp
var policy = new ConditionalAccessPolicy
{
    DisplayName = "MFA obligatorio para admins",
    State = ConditionalAccessPolicyState.Enabled,
    Conditions = new ConditionalAccessConditionSet
    {
        Applications = new ConditionalAccessApplications
        {
            IncludeApplications = new[] { "All" }
        },
        Users = new ConditionalAccessUserCondition
        {
            IncludeGroups = new[] { adminGroupId }
        }
    },
    GrantControls = new ConditionalAccessGrantControls
    {
        Operator = "AND",
        BuiltInControls = new[] { ConditionalAccessGrantControl.Mfa }
    }
};

await graphClient.Identity.ConditionalAccess.Policies
    .Request()
    .AddAsync(policy);
```

### Política 2: MFA para ubicaciones de riesgo

```
Nombre: "MFA para acceso externo a apps críticas"

Asignaciones:
  Usuarios: Todos EXCEPTO cuentas de servicio
  Aplicaciones: Office 365, Azure Portal
  Condiciones:
    Ubicaciones: Cualquiera EXCEPTO IPs corporativas

Controles:
  Require MFA
```

Configuración de ubicaciones confiables:

```csharp
// Definir ubicaciones corporativas como confiables
var namedLocation = new IpNamedLocation
{
    DisplayName = "Oficinas Corporativas",
    IsTrusted = true,
    IpRanges = new[]
    {
        new IPv4CidrRange { CidrAddress = "203.0.113.0/24" },  // Oficina Madrid
        new IPv4CidrRange { CidrAddress = "198.51.100.0/24" }  // Oficina Barcelona
    }
};

await graphClient.Identity.ConditionalAccess.NamedLocations
    .Request()
    .AddAsync(namedLocation);
```

### Política 3: Bloqueo de países de alto riesgo

```
Nombre: "Bloquear países de alto riesgo"

Asignaciones:
  Usuarios: Todos
  Aplicaciones: Todas
  Condiciones:
    Ubicaciones: Países específicos (Rusia, China, etc.)

Controles:
  Bloquear acceso
```

**Importante**: Excluimos usuarios que legítimamente viajaban a esos países.

```csharp
var policy = new ConditionalAccessPolicy
{
    DisplayName = "Bloquear países de alto riesgo",
    Conditions = new ConditionalAccessConditionSet
    {
        Locations = new ConditionalAccessLocationCondition
        {
            IncludeLocations = new[] { "RU", "CN", "KP" } // ISO country codes
        },
        Users = new ConditionalAccessUserCondition
        {
            IncludeUsers = new[] { "All" },
            ExcludeGroups = new[] { travelersGroupId } // Grupo de viajeros frecuentes
        }
    },
    GrantControls = new ConditionalAccessGrantControls
    {
        BuiltInControls = new[] { ConditionalAccessGrantControl.Block }
    }
};
```

### Política 4: Require dispositivos gestionados

```
Nombre: "Dispositivo gestionado para email corporativo"

Asignaciones:
  Usuarios: Todos
  Aplicaciones: Exchange Online
  Condiciones:
    Plataformas: iOS, Android

Controles:
  Require dispositivo marked as compliant
  OR
  Require Azure AD Hybrid Joined device
```

Esto forzaba que dispositivos móviles estuvieran inscritos en Intune:

```csharp
var policy = new ConditionalAccessPolicy
{
    DisplayName = "Dispositivo gestionado para email",
    Conditions = new ConditionalAccessConditionSet
    {
        Applications = new ConditionalAccessApplications
        {
            IncludeApplications = new[] { "00000002-0000-0ff1-ce00-000000000000" } // Exchange Online
        },
        Platforms = new ConditionalAccessPlatformCondition
        {
            IncludePlatforms = new[] { "iOS", "android" }
        }
    },
    GrantControls = new ConditionalAccessGrantControls
    {
        Operator = "OR",
        BuiltInControls = new[]
        {
            ConditionalAccessGrantControl.CompliantDevice,
            ConditionalAccessGrantControl.DomainJoinedDevice
        }
    }
};
```

## Usando señales de riesgo (Azure AD Identity Protection)

Conditional Access puede reaccionar a señales de riesgo automáticas:

### Política basada en riesgo de usuario

```
Nombre: "MFA si usuario de riesgo"

Condiciones:
  Riesgo de usuario: Alto o Medio

Controles:
  Require cambio de password
  Require MFA
```

```csharp
var policy = new ConditionalAccessPolicy
{
    DisplayName = "MFA si usuario de riesgo",
    Conditions = new ConditionalAccessConditionSet
    {
        UserRiskLevels = new[] { RiskLevel.High, RiskLevel.Medium }
    },
    GrantControls = new ConditionalAccessGrantControls
    {
        Operator = "AND",
        BuiltInControls = new[]
        {
            ConditionalAccessGrantControl.Mfa,
            ConditionalAccessGrantControl.PasswordChange
        }
    }
};
```

Azure AD Identity Protection detecta automáticamente:
- Credenciales filtradas en dark web
- Viaje imposible (login desde Madrid y 2h después desde Tokio)
- Actividad anómala de usuario

### Política basada en riesgo de sign-in

```
Nombre: "Bloquear sign-ins de alto riesgo"

Condiciones:
  Riesgo de sign-in: Alto

Controles:
  Bloquear acceso
```

## Modo Report-Only

**Crítico**: Nunca actives una política directamente en producción.

```csharp
// Crear política en modo report-only primero
var policy = new ConditionalAccessPolicy
{
    DisplayName = "Nueva política - Testing",
    State = ConditionalAccessPolicyState.EnabledForReportingButNotEnforced,
    // ... resto de configuración
};
```

Esto te deja ver qué habría pasado sin afectar usuarios reales:

```
Azure AD → Sign-ins log → Filtrar por "Conditional Access"
```

Después de 1-2 semanas monitoreando, cambiamos a modo activo:

```csharp
policy.State = ConditionalAccessPolicyState.Enabled;
await graphClient.Identity.ConditionalAccess.Policies[policy.Id]
    .Request()
    .UpdateAsync(policy);
```

## Exclusiones críticas

**Siempre excluye**:

### 1. Cuentas de emergencia (break glass)

```csharp
var policy = new ConditionalAccessPolicy
{
    Conditions = new ConditionalAccessConditionSet
    {
        Users = new ConditionalAccessUserCondition
        {
            IncludeUsers = new[] { "All" },
            ExcludeUsers = new[] { breakGlassAccountId }
        }
    }
};
```

Estas cuentas solo se usan si todo falla (MFA roto, etc.)

### 2. Cuentas de servicio

```csharp
// Excluir grupo de service accounts
ExcludeGroups = new[] { serviceAccountsGroupId }
```

## Monitorización

Creamos un dashboard en Azure Monitor:

```kusto
// Sign-ins bloqueados por Conditional Access
SigninLogs
| where ConditionalAccessStatus == "failure"
| summarize count() by ConditionalAccessPolicies, bin(TimeGenerated, 1h)
| render timechart

// Usuarios más afectados
SigninLogs
| where ConditionalAccessStatus == "failure"
| summarize count() by UserPrincipalName
| top 20 by count_
```

Alertas automáticas si >50 sign-ins bloqueados en 1 hora:

```csharp
// Crear alert rule vía ARM template
{
  "condition": {
    "allOf": [{
      "query": "SigninLogs | where ConditionalAccessStatus == 'failure' | count",
      "threshold": 50,
      "operator": "GreaterThan"
    }]
  },
  "actions": {
    "actionGroups": ["/subscriptions/.../actionGroups/SecurityTeam"]
  }
}
```

## Resultados

**Antes**:
- Phishing exitoso cada 2-3 meses
- Accesos desde IPs sospechosas sin control
- Usuarios compartían credenciales

**Después** (6 meses con Conditional Access):
- 0 phishing exitosos
- 2,300 intentos de acceso bloqueados desde países de riesgo
- 95% de dispositivos móviles gestionados por Intune
- Usuarios satisfechos (MFA solo cuando es necesario)

## Recomendaciones

1. **Empieza gradual**: Una política a la vez
2. **Report-Only es tu amigo**: Úsalo siempre primero
3. **Excluye cuentas de emergencia**: Te salvará algún día
4. **Combina con Identity Protection**: Señales de riesgo automáticas valen oro
5. **Monitoriza constantemente**: Los usuarios encontrarán edge cases que no anticipaste

¿Usas Conditional Access? ¿Qué políticas te han funcionado mejor?
