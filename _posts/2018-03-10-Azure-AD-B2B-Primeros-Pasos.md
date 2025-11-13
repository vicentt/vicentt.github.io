---
layout: post
title: Azure AD B2B - Colaboración con usuarios externos sin dolores de cabeza
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD, Seguridad]
published: true
---

> Cómo implementamos colaboración segura con partners externos usando Azure AD B2B

# El problema 🤔

Teníamos un portal interno que necesitaban acceder consultores externos y partners. Las opciones tradicionales eran un desastre:

1. **Crear cuentas locales**: Gestión manual horrible
2. **Usuario/password compartidos**: Pesadilla de seguridad
3. **VPN para todos**: Costoso y complejo

Entonces descubrimos Azure AD B2B (Business to Business).

## ¿Qué es Azure AD B2B?

En palabras simples: invitas usuarios externos que mantienen sus propias identidades. Ellos usan sus credenciales corporativas, tú controlas qué pueden hacer.

No es:
- Azure AD B2C (ese es para clientes/consumidores)
- Federation tradicional (más simple que eso)
- Guest accounts manuales (esto se automatiza)

## Caso de uso real

Teníamos 15 consultores de 3 empresas diferentes que necesitaban acceso a:
- SharePoint con documentación de proyecto
- Azure Portal para ver recursos específicos
- Aplicación web interna de seguimiento

### Solución anterior (la mala)

```
1. Crear cuenta en nuestro AD: consultor1@tusdominio.onmicrosoft.com
2. Enviar credenciales por email
3. Usuario tiene que recordar OTRO usuario/password
4. Cuando termina proyecto: ¿quién se acuerda de borrar la cuenta?
```

Resultado: 50+ cuentas "zombie" en nuestro AD de consultores que ya no trabajan con nosotros.

### Solución con Azure AD B2B

```
1. Invitar: consultor@suempresa.com
2. Ellos aceptan invitación
3. Acceden con sus credenciales corporativas
4. Nosotros controlamos permisos
5. Cuando termina: revocamos acceso (su cuenta sigue en su empresa)
```

## Implementación paso a paso

### 1. Invitar usuarios

Hay varias formas. La más simple es por portal:

```
Azure Portal → Azure Active Directory → Users → New guest user
```

Pero para hacerlo programáticamente (que es lo que hicimos):

```csharp
var invitation = new Invitation
{
    InvitedUserEmailAddress = "consultor@partner.com",
    InviteRedirectUrl = "https://myapp.contoso.com",
    InvitedUserDisplayName = "Juan Consultor",
    SendInvitationMessage = true
};

var result = await graphClient.Invitations
    .Request()
    .AddAsync(invitation);
```

### 2. Asignar a grupos

No asignes permisos usuario por usuario. Usa grupos:

```csharp
// Crear grupo para consultores externos
var group = new Group
{
    DisplayName = "Consultores-Proyecto-X",
    MailEnabled = false,
    SecurityEnabled = true,
    MailNickname = "consultores-x"
};

await graphClient.Groups
    .Request()
    .AddAsync(group);

// Añadir usuarios al grupo
await graphClient.Groups[groupId].Members.References
    .Request()
    .AddAsync(user);
```

### 3. Configurar aplicación para aceptar usuarios externos

En tu app registration:

```
Azure AD → App registrations → Tu app → Authentication
→ Supported account types → "Accounts in any organizational directory"
```

En código:

```csharp
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(options =>
    {
        options.Instance = "https://login.microsoftonline.com/";
        options.TenantId = "tu-tenant-id";
        options.ClientId = "tu-client-id";

        // Importante: acepta usuarios de cualquier tenant
        options.TokenValidationParameters.ValidateIssuer = false;
    });
```

## Lecciones aprendidas

### 1. Email de invitación personalizado

El email por defecto es genérico. Personalizarlo reduce confusión:

```csharp
var customizedMessageBody = @"
Hola,

Has sido invitado a colaborar en el proyecto X.
Por favor acepta la invitación para acceder a los recursos.

Saludos,
El equipo
";

invitation.InvitedUserMessageInfo = new InvitedUserMessageInfo
{
    CustomizedMessageBody = customizedMessageBody
};
```

### 2. No todos los partners tienen Azure AD

Algunos consultores freelance solo tenían Gmail. Azure AD B2B soporta cuentas personales (Microsoft Accounts), pero la experiencia es peor.

**Solución**: Pedimos a los freelancers que crearan una cuenta Microsoft con su email profesional.

### 3. Auditoría y compliance

Con B2B tienes logs de quién accedió a qué:

```
Azure AD → Audit logs
Filtrar por: "Invite external user" y "Redeem external user invitation"
```

Esto fue crítico para compliance con GDPR.

### 4. Revocación de acceso

Cuando termina el proyecto:

```
Azure AD → Users → Usuario guest → Delete
```

O programáticamente:

```csharp
await graphClient.Users[userId]
    .Request()
    .DeleteAsync();
```

La cuenta del usuario en SU empresa no se toca, solo se borra la relación B2B.

## Comparativa con alternativas

| Opción | Pros | Contras |
|--------|------|---------|
| **Cuentas locales** | Control total | Gestión manual, usuarios odian tener múltiples passwords |
| **Federation** | SSO real | Complejo, requiere cooperación del partner |
| **Azure AD B2B** | Fácil, el partner controla su identidad, buen SSO | Requiere que partner tenga Azure AD o Microsoft Account |
| **VPN** | Funciona siempre | Costoso, complejo, mala UX |

## Consejos prácticos

1. **Usa grupos para todo**: Nunca asignes permisos directamente a usuarios B2B
2. **Automatiza invitaciones**: Si invitas >10 usuarios, usa Microsoft Graph API
3. **Revisa usuarios guest regularmente**: Crea un script mensual que liste guests sin actividad
4. **Documenta el proceso**: Los nuevos partners agradecerán un PDF de "cómo aceptar invitación"
5. **MFA obligatorio**: Puedes forzar que usuarios B2B usen MFA incluso si su empresa no lo tiene

## Script útil: Listar guests sin actividad

```powershell
Connect-AzureAD

$guests = Get-AzureADUser -Filter "userType eq 'Guest'" -All $true

foreach ($guest in $guests) {
    $signIns = Get-AzureADAuditSignInLogs -Filter "userId eq '$($guest.ObjectId)'" -Top 1

    if ($signIns.Count -eq 0) {
        Write-Host "Usuario sin logins: $($guest.Mail)"
    }
}
```

## Conclusión

Azure AD B2B resolvió nuestro problema de colaboración externa de forma elegante. No es perfecto, pero es muchísimo mejor que las alternativas.

Si trabajas con consultores o partners externos regularmente, B2B debería estar en tu radar.

¿Usas Azure AD B2B? ¿Qué problemas te has encontrado?
