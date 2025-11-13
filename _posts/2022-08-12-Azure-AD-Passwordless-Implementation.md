---
layout: post
title: Passwordless con Azure AD - Adiós a las contraseñas
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD, Seguridad]
published: true
---

> Cuando decidimos eliminar contraseñas completamente de la organización

# El problema

Passwords son el eslabón más débil:
- 123456, password, qwerty en top 10
- Reutilización masiva
- Phishing exitoso cada semana
- Password resets: 30% de tickets de soporte

**Solución**: Eliminar passwords completamente. **Passwordless**.

## Opciones Passwordless en Azure AD

1. **Windows Hello for Business** (PCs/laptops)
2. **FIDO2 Security Keys** (YubiKey, etc)
3. **Microsoft Authenticator** (móviles)

Implementamos los 3.

## Fase 1: Microsoft Authenticator

### Habilitar Authenticator passwordless

```
Azure AD → Security → Authentication methods → Microsoft Authenticator

Settings:
- Enable: Yes
- Target: All users (o grupos específicos)
- Authentication mode: Passwordless
- Require number matching: Yes
- Show application name: Yes
- Show geographic location: Yes
```

### Configurar via PowerShell

```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.AuthenticationMethod"

$params = @{
    State = "enabled"
    IncludeTargets = @(
        @{
            TargetType = "group"
            Id = "all_users"
            IsRegistrationRequired = $true
        }
    )
    FeatureSettings = @{
        DisplayAppInformationRequiredState = @{
            State = "enabled"
            IncludeTarget = @{
                TargetType = "group"
                Id = "all_users"
            }
        }
        DisplayLocationInformationRequiredState = @{
            State = "enabled"
            IncludeTarget = @{
                TargetType = "group"
                Id = "all_users"
            }
        }
    }
}

Update-MgPolicyAuthenticationMethodPolicyAuthenticationMethodConfiguration `
    -AuthenticationMethodConfigurationId "MicrosoftAuthenticator" `
    -BodyParameter $params
```

### Enrollment del usuario

```
1. Usuario instala Microsoft Authenticator en móvil
2. Usuario va a https://aka.ms/mysecurityinfo
3. Add method → Microsoft Authenticator
4. Scanea QR code
5. Aprueba notification de test
6. ✅ Passwordless habilitado
```

### Experiencia de login

```
1. Usuario va a login.microsoft.com
2. Ingresa email
3. Recibe push notification en móvil
4. Ve número en pantalla PC
5. Ingresa mismo número en móvil
6. Autenticación biométrica (Face ID/Touch ID)
7. ✅ Logueado sin password
```

## Fase 2: FIDO2 Security Keys

### Habilitar FIDO2

```
Azure AD → Security → Authentication methods → FIDO2 security key

Settings:
- Enable: Yes
- Target: Admins + IT staff
- Allow self-service set up: Yes
- Enforce attestation: Yes
- Enforce key restrictions: No (o sí, si quieres whitelist de marcas)
```

### Marcas compatibles

- YubiKey 5 Series
- Feitian BioPass
- Google Titan Security Key
- Thales (Gemalto)

Compramos **YubiKey 5 NFC** para cada admin/usuario IT.

### Registro de YubiKey

```
1. Usuario va a https://aka.ms/mysecurityinfo
2. Add method → Security key
3. Selecciona tipo: USB o NFC
4. Inserta YubiKey en USB
5. Toca botón dorado de YubiKey
6. Ingresa PIN de YubiKey
7. ✅ YubiKey registrado
```

### Login con YubiKey

```
1. Usuario va a login.microsoft.com
2. Ingresa email
3. Opción: "Sign in with security key"
4. Inserta YubiKey
5. Toca botón
6. Ingresa PIN
7. ✅ Logueado
```

## Fase 3: Windows Hello for Business

### Requisitos

- Windows 10/11 Pro o Enterprise
- Azure AD joined o Hybrid joined
- TPM 2.0

### Configurar policy

```
Azure AD → Devices → Device settings
→ Configure Windows Hello for Business: Yes
→ Use security keys for sign-in: Yes
```

Intune policy:

```
Intune → Devices → Configuration profiles → Create profile
Platform: Windows 10 and later
Profile type: Templates → Identity Protection

Settings:
- Configure Windows Hello for Business: Enable
- Minimum PIN length: 6
- Use a Trusted Platform Module (TPM): Require
- Use biometrics: Allow
```

### Enrollment

Automático al unirse a Azure AD:

```
Windows → Settings → Accounts → Access work or school
→ Connect → Join this device to Azure AD

Durante setup:
→ Set up a PIN
→ Set up face recognition / fingerprint (si hardware compatible)
```

### Login

```
1. Usuario enciende PC
2. Ve su cara (Windows Hello facial recognition)
3. ✅ Logueado en <1 segundo
```

## Conditional Access para forzar passwordless

```
Azure AD → Security → Conditional Access → New policy

Name: Force Passwordless
Assignments:
  Users: All users
  Cloud apps: All cloud apps

Access controls:
  Grant: Require authentication strength
  Authentication strength: Passwordless MFA
```

Via PowerShell:

```powershell
$conditions = @{
    Users = @{
        IncludeUsers = @("All")
    }
    Applications = @{
        IncludeApplications = @("All")
    }
}

$grantControls = @{
    Operator = "OR"
    AuthenticationStrength = @{
        Id = "00000000-0000-0000-0000-000000000004"  # Passwordless MFA
    }
}

New-MgIdentityConditionalAccessPolicy `
    -DisplayName "Force Passwordless" `
    -State "enabled" `
    -Conditions $conditions `
    -GrantControls $grantControls
```

## Migración de usuarios

### Plan de rollout

**Semana 1-2: Pilot (10 usuarios)**
- IT staff
- Feedback diario
- Ajustar policies

**Semana 3-4: Early adopters (50 usuarios)**
- Usuarios tech-savvy
- Monitoreo de soporte tickets

**Semana 5-8: Departamentos (500 usuarios)**
- Por departamento
- Training sessions

**Semana 9-12: Toda la empresa (2000 usuarios)**
- Gradual
- Soporte intensivo

### Comunicación

Email template:

```
Subject: 🔐 Mejoramos tu seguridad: Di adiós a las contraseñas

Hola,

Estamos eliminando contraseñas de la empresa. Ahora usarás tu móvil o Face ID para iniciar sesión.

Pasos:
1. Instala Microsoft Authenticator en tu móvil
2. Visita https://aka.ms/mysecurityinfo
3. Sigue las instrucciones

Video tutorial: [enlace]

Soporte: helpdesk@empresa.com

¡Bienvenido al futuro sin contraseñas!
```

### Training

Session de 15 minutos por departamento:

```
1. ¿Por qué passwordless? (5 min)
   - Passwords son inseguros
   - Phishing bloqueado
   - Más conveniente

2. Demo en vivo (5 min)
   - Enrollment de Authenticator
   - Login passwordless

3. Q&A (5 min)
```

## Problemas encontrados

### 1. "Perdí mi móvil"

**Solución**: Método de backup requerido

```
Azure AD → Authentication methods → Registration campaign

Require users to register:
- Primary: Microsoft Authenticator
- Backup: FIDO2 security key OR phone sign-in
```

### 2. Usuarios sin smartphone

**Solución**: YubiKeys para todos ellos

```
Budget: 100 YubiKeys × $50 = $5,000
```

### 3. Legacy apps que requieren password

```
Error: "This app doesn't support passwordless"
```

**Solución temporal**: App passwords

```
Azure AD → Users → [Usuario] → Authentication methods
→ App passwords → Create new app password
```

Pero marca para deprecar la app legacy.

### 4. Viajes internacionales

Usuario en otro país sin móvil/roaming.

**Solución**: Temporary Access Pass

```powershell
# Admin crea TAP para usuario
New-MgUserAuthenticationTemporaryAccessPassMethod `
    -UserId "usuario@empresa.com" `
    -BodyParameter @{
        LifetimeInMinutes = 60
        IsUsableOnce = $false
    }

# Usuario usa TAP como password temporal
```

## Monitoreo

Dashboard en Azure Monitor:

```kusto
SigninLogs
| where TimeGenerated > ago(30d)
| extend AuthMethod = tostring(parse_json(AuthenticationDetails)[0].authenticationMethod)
| summarize Count = count() by AuthMethod
| render piechart

// Resultados:
// Microsoft Authenticator (Passwordless): 75%
// FIDO2 Security Key: 15%
// Windows Hello: 8%
// Password: 2% (legacy apps)
```

Alertas:

```kusto
// Alert si uso de password aumenta
SigninLogs
| where TimeGenerated > ago(1h)
| extend AuthMethod = tostring(parse_json(AuthenticationDetails)[0].authenticationMethod)
| where AuthMethod == "Password"
| summarize Count = count()
| where Count > 50
```

## Desactivar passwords

Cuando 95%+ de usuarios usan passwordless:

```
Azure AD → Users → [Usuario] → Authentication methods
→ Password → Delete

// O via PowerShell
Remove-MgUserAuthenticationPasswordMethod -UserId "usuario@empresa.com"
```

Ahora el usuario literalmente no puede usar password.

## Costo-beneficio

**Costos**:
- YubiKeys: $5,000 (100 unidades)
- Training: 40 horas staff × $50/hora = $2,000
- **Total**: $7,000

**Beneficios** (anuales):
- Password reset tickets: -70% = $15,000 ahorrados
- Phishing incidents: -95% = $50,000+ ahorrados
- Tiempo de login: -60% = Productividad +$10,000

**ROI**: 10x en primer año

## Resultados después de 12 meses

**Adopción**:
- Microsoft Authenticator: 85%
- FIDO2 Keys: 10%
- Windows Hello: 3%
- Password: 2% (legacy apps)

**Seguridad**:
- Phishing exitoso: 12/año → 0/año
- Password resets: 400/mes → 50/mes
- Compromised accounts: 5/año → 0/año

**Experiencia**:
- Login time: 15s → 3s
- Satisfacción usuarios: 4.7/5
- Quejas: "No puedo creer que antes usábamos passwords"

## Recomendaciones

1. **Empieza con Authenticator** - Más fácil de adoptar
2. **YubiKeys para admins** - Seguridad máxima
3. **Windows Hello si es posible** - Mejor UX
4. **Conditional Access estricto** - Forzar passwordless
5. **Método de backup obligatorio** - Para móviles perdidos
6. **Monitoreo constante** - Ver adopción
7. **Training continuo** - Nuevos empleados

¿Has implementado passwordless? ¿Qué resistencia encontraste?
