---
layout: post
title: Azure AD PIM - Administración de accesos privilegiados Just-In-Time
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD, Seguridad]
published: true
---

> Cuando los administradores no deberían ser administradores todo el tiempo

# El incidente

Un administrador con cuenta comprometida. El atacante tenía acceso Global Admin permanente. Daño masivo en 20 minutos.

Después de ese incidente, implementamos **Privileged Identity Management (PIM)**.

## El problema con permisos permanentes

Escenario típico:
```
Juan = Global Administrator (permanente)
```

Juan 99% del tiempo hace tareas normales (email, docs).
Pero tiene privilegios de dios 24/7.

Si su cuenta se compromete:
- Atacante tiene Global Admin inmediatamente
- Puede borrar todo
- Puede crear backdoors
- Puede extraer datos

## Privileged Identity Management (PIM)

PIM implementa **Just-In-Time access**:

```
Juan = Eligible for Global Administrator
     ↓ (cuando lo necesita)
     Activa rol por 4 horas
     ↓ (con MFA + justificación)
     Juan = Active Global Administrator (temporal)
     ↓ (después de 4 horas)
     Juan = Eligible (privilegios expirados)
```

## Configuración inicial

### 1. Habilitar PIM

```
Azure AD → Privileged Identity Management → Azure AD roles → Get started
```

Requiere licencia Azure AD Premium P2.

### 2. Hacer roles "eligible" en lugar de "permanent"

```
PIM → Azure AD roles → Assignments

Para cada admin:
  1. Remove permanent assignment
  2. Add eligible assignment
```

Vía PowerShell:

```powershell
Connect-AzureAD

# Obtener role
$role = Get-AzureADDirectoryRole | Where-Object {$_.DisplayName -eq "Global Administrator"}

# Listar miembros actuales
$members = Get-AzureADDirectoryRoleMember -ObjectId $role.ObjectId

# Convertir a eligible
foreach ($member in $members) {
    # Remover permanent
    Remove-AzureADDirectoryRoleMember -ObjectId $role.ObjectId -MemberId $member.ObjectId

    # Añadir como eligible (requiere Microsoft Graph)
    # Usar Azure Portal o Graph API para esto
}
```

### 3. Configurar Role Settings

```
PIM → Azure AD roles → Settings → Global Administrator → Edit
```

**Activation**:
- Maximum duration: 4 hours
- Require MFA: Yes
- Require justification: Yes
- Require ticket information: Yes (opcional, para control adicional)
- Require approval: No (para admins confiables) / Yes (para roles muy sensibles)

**Assignment**:
- Allow permanent eligible assignment: No
- Expire eligible assignments after: 6 months (fuerza revisión)

**Notification**:
- Send notifications when admins activate: Yes
- Send to: Security team email

## Workflow del admin

### Activación de rol

Admin necesita privilegios:

```
Azure Portal → PIM → My roles → Azure AD roles
→ Global Administrator → Activate

Formulario:
  - Duration: 4 hours (máximo configurado)
  - Justification: "Necesito crear nuevo tenant para proyecto X"
  - MFA challenge
  - Submit
```

Después de aprobar MFA:
```
Status: Active
Expires: 4 hours from now
```

Durante esas 4 horas, el admin tiene privilegios completos.

### Vía PowerShell (automatizado)

Para entornos donde admins activan frecuentemente:

```powershell
# Instalar módulo
Install-Module AzureADPreview

# Conectar
Connect-AzureAD

# Solicitar activación
$roleDefinitionId = (Get-AzureADMSPrivilegedRoleDefinition -ProviderId aadRoles -ResourceId $tenantId | Where-Object {$_.DisplayName -eq "Global Administrator"}).Id

$schedule = New-Object Microsoft.Open.MSGraph.Model.AzureADMSPrivilegedSchedule
$schedule.Type = "Once"
$schedule.StartDateTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss.fffZ")
$schedule.endDateTime = (Get-Date).AddHours(4).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss.fffZ")

Open-AzureADMSPrivilegedRoleAssignmentRequest `
    -ProviderId 'aadRoles' `
    -ResourceId $tenantId `
    -RoleDefinitionId $roleDefinitionId `
    -SubjectId $userId `
    -Type 'UserAdd' `
    -AssignmentState 'Active' `
    -Schedule $schedule `
    -Reason "Deployment de nueva infraestructura"
```

## Aprobación de activaciones (opcional)

Para roles ultra-sensibles:

```
PIM → Settings → Global Administrator → Edit
→ Activation → Require approval: Yes
→ Select approvers: [Security team members]
```

Workflow con aprobación:

```
1. Admin solicita activación
     ↓
2. Approver recibe notificación email
     ↓
3. Approver revisa en: PIM → Approve requests
     ↓
4. Si aprueba → Admin tiene privilegios
5. Si niega → Admin no puede activar
```

## Access Reviews

Revisión periódica de quién tiene acceso:

```
PIM → Azure AD roles → Access reviews → New

Review name: Quarterly Global Admin Review
Start date: 2024-01-01
Frequency: Quarterly
Duration: 14 days
Reviewers: Self-review + Manager approval
```

Cada trimestre, los admins deben justificar por qué siguen necesitando el rol.

## Alertas de seguridad

PIM genera alertas automáticas:

### Alerta: "Too many Global Administrators"

```
Trigger: >5 Global Admins activos
Action: Revisar y reducir
```

### Alerta: "Roles are being assigned outside PIM"

```
Trigger: Alguien asignó rol permanente directamente
Action: Investigar quién y por qué
```

### Alerta: "Potential stale accounts"

```
Trigger: Admin eligible que no ha activado en 90+ días
Action: Remover acceso innecesario
```

Configurar alertas:

```
PIM → Azure AD roles → Alerts → Alert settings
```

## Monitoreo y auditoría

Ver activaciones históricas:

```
PIM → Azure AD roles → Resource audit

Filtrar por:
  - Action: Activate
  - Role: Global Administrator
  - Time range: Last 30 days
```

Query en Azure Monitor:

```kusto
AuditLogs
| where OperationName == "Add member to role completed (PIM activation)"
| where TargetResources has "Global Administrator"
| project TimeGenerated, Identity, ActivityDisplayName, Result
| order by TimeGenerated desc
```

Alertas para activaciones sospechosas:

```kusto
AuditLogs
| where OperationName == "Add member to role completed (PIM activation)"
| where TimeGenerated between (datetime(2024-01-01T22:00:00Z) .. datetime(2024-01-01T06:00:00Z)) // Horario nocturno
| project TimeGenerated, Identity, TargetResources
```

Alert rule:
```
Condition: Count > 0
Action: Email security team
```

## Roles específicos recomendados para PIM

No todos los roles necesitan PIM:

**Definitivamente PIM**:
- Global Administrator
- Privileged Role Administrator
- Security Administrator
- Exchange Administrator
- SharePoint Administrator

**Considerar PIM**:
- User Administrator
- Application Administrator
- Cloud Application Administrator

**Probablemente NO necesita PIM**:
- Directory Readers (solo lectura)
- Guest Inviter
- Message Center Reader

## Break-glass accounts

**Crítico**: Mantener 2 cuentas de emergencia FUERA de PIM:

```
Cuentas:
  - breakglass1@tudominio.onmicrosoft.com
  - breakglass2@tudominio.onmicrosoft.com

Configuración:
  - Global Administrator (permanent)
  - NO conditioned by PIM
  - NO MFA (para emergencias donde MFA falla)
  - Contraseñas largas (20+ caracteres) en caja fuerte física
  - Monitoreadas 24/7 (alerta si se usan)
```

Si PIM falla o hay lockout masivo, estas cuentas pueden restaurar acceso.

## Problemas encontrados

### 1. Admins olvidaban activar rol

```
Admin: "¿Por qué no puedo crear usuarios?"
Respuesta: "Olvidaste activar tu rol Global Admin"
```

**Solución**: Entrenamiento + mensajes de error claros.

### 2. Aprobaciones retrasaban trabajo crítico

Aprobador de vacaciones → Activación bloqueada.

**Solución**: Múltiples approvers + activación sin aprobación para roles menos sensibles.

### 3. Roles expiraban en medio de cambio

Admin haciendo deployment largo → Rol expira → Deployment falla.

**Solución**: Extender duration máxima a 8 horas para roles operativos.

## Resultados

**Antes**:
- 12 Global Admins permanentes
- Cuentas comprometidas = impacto inmediato
- 0 auditoría de cuándo se usan privilegios

**Después (con PIM)**:
- 12 Global Admins eligible
- Promedio 2-3 activos simultáneamente
- Cada activación requiere MFA + justificación
- Auditoría completa de uso de privilegios
- Tiempo de exposición reducido 95%

**Incidentes desde implementación**: 0 compromisos con impacto debido a PIM.

¿Usas PIM en tu organización? ¿Qué configuración de activation time funciona mejor para ti?
