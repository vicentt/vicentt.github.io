---
layout: post
title: Azure AD Access Reviews - Gobierno automático de permisos
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD, Governance]
published: true
---

> Cuando necesitas revisar quién tiene acceso a qué, trimestre tras trimestre

# El problema

Organización con 2000+ empleados y colaboradores externos. Permisos otorgados pero nunca revocados.

Auditoría reveló:
- 200+ empleados **ex-empleados** aún con acceso
- Consultores con acceso meses después de terminar proyecto
- Grupos con 50+ miembros, nadie sabía por qué
- Acceso a aplicaciones críticas sin justificación

**Solución**: Azure AD Access Reviews automáticas.

## ¿Qué son Access Reviews?

Procesos periódicos donde owners/managers revisan y aprueban o remueven acceso de usuarios.

```
Trimestre → Access Review created
          → Reviewers notificados
          → Reviewers aprueban/niegan accesos
          → Accesos denegados se remueven automáticamente
```

## Caso 1: Review de grupos de Azure AD

### Crear Access Review para grupo

```
Azure AD → Identity Governance → Access reviews → New access review

Nombre: Quarterly Review - VPN Access Group
Descripción: Review usuarios con acceso VPN

Start date: 2024-01-01
Frequency: Quarterly
Duration: 14 days

Scope:
  - Users and groups
  - Select group: VPN-Users

Reviewers:
  - Group owner(s)
  - Fallback: IT Security Team

Settings:
  - Upon completion → Auto apply results to resource
  - If reviewers don't respond → Remove access
  - Enable reviewer decision helpers → Show last sign-in date
```

### Via PowerShell (automatizar creación)

```powershell
Install-Module Microsoft.Graph.Identity.Governance

Connect-MgGraph -Scopes "AccessReview.ReadWrite.All"

$params = @{
    displayName = "Quarterly VPN Access Review"
    descriptionForAdmins = "Review VPN access quarterly"
    descriptionForReviewers = "Please review if these users still need VPN access"
    scope = @{
        "@odata.type" = "#microsoft.graph.accessReviewQueryScope"
        query = "/groups/[group-id]/members"
        queryType = "MicrosoftGraph"
    }
    reviewers = @(
        @{
            query = "/groups/[group-id]/owners"
            queryType = "MicrosoftGraph"
        }
    )
    settings = @{
        recurrence = @{
            pattern = @{
                type = "absoluteMonthly"
                interval = 3
            }
            range = @{
                type = "noEnd"
                startDate = "2024-01-01"
            }
        }
        instanceDurationInDays = 14
        autoApplyDecisionsEnabled = $true
        defaultDecisionEnabled = $true
        defaultDecision = "Deny"
        recommendationsEnabled = $true
    }
}

New-MgIdentityGovernanceAccessReviewDefinition -BodyParameter $params
```

## Caso 2: Review de App Roles

Revisar quién tiene roles de aplicaciones (admin, contributor, etc.):

```powershell
$params = @{
    displayName = "Review of Application Administrators"
    scope = @{
        "@odata.type" = "#microsoft.graph.accessReviewQueryScope"
        query = "/servicePrincipals/[app-id]/appRoleAssignedTo"
        queryType = "MicrosoftGraph"
    }
    reviewers = @(
        @{
            query = "./manager"
            queryType = "MicrosoftGraph"
        }
    )
    fallbackReviewers = @(
        @{
            query = "/users/[security-admin-id]"
            queryType = "MicrosoftGraph"
        }
    )
    settings = @{
        instanceDurationInDays = 7
        recurrence = @{
            pattern = @{
                type = "absoluteMonthly"
                interval = 6
            }
        }
        autoApplyDecisionsEnabled = $true
        defaultDecisionEnabled = $true
        defaultDecision = "Deny"
        justificationRequiredOnApproval = $true
    }
}

New-MgIdentityGovernanceAccessReviewDefinition -BodyParameter $params
```

## Caso 3: Review de Guest Users

Revisar usuarios externos (B2B):

```
Access review settings:
  Scope: All guest users
  Reviewers: Resource owners
  Frequency: Monthly
  Duration: 7 days
  Auto-apply: Yes
  Default decision: Remove access
  Require justification: Yes
```

```powershell
$params = @{
    displayName = "Monthly Guest User Review"
    scope = @{
        "@odata.type" = "#microsoft.graph.accessReviewQueryScope"
        query = "/users?$filter=userType eq 'Guest'"
        queryType = "MicrosoftGraph"
    }
    reviewers = @(
        @{
            query = "/users/[sponsor-id]"  # Usuario que invitó
            queryType = "MicrosoftGraph"
        }
    )
    settings = @{
        instanceDurationInDays = 7
        recurrence = @{
            pattern = @{
                type = "absoluteMonthly"
                interval = 1
            }
        }
        autoApplyDecisionsEnabled = $true
        defaultDecisionEnabled = $true
        defaultDecision = "Deny"
        justificationRequiredOnApproval = $true
        recommendationsEnabled = $true
    }
}
```

## Workflow del Reviewer

### 1. Notificación por email

```
Subject: [Action Required] Access Review: VPN Access Group

You have been assigned to review access for the VPN Access Group.

Review period: Jan 1 - Jan 14, 2024

[Start Review Button]
```

### 2. Portal de review

```
Access Reviews → My Access Reviews → VPN Access Group

Users to review (20):
┌──────────────────────────────────────────────────────────┐
│ Juan Pérez                                               │
│ Email: juan@empresa.com                                  │
│ Last sign-in: 2 days ago                                 │
│ Recommendation: Approve (active user)                    │
│                                                           │
│ [Approve] [Deny] [Don't know]                            │
│ Justification: _____________________________             │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ María González                                            │
│ Email: maria@empresa.com                                  │
│ Last sign-in: Never                                       │
│ Recommendation: Deny (inactive user)                      │
│                                                           │
│ [Approve] [Deny] [Don't know]                            │
│ Justification: _____________________________             │
└──────────────────────────────────────────────────────────┘
```

### 3. Decisiones bulk

```typescript
// Script para aprobar/denegar en bulk via Graph API
const reviewDecisions = [
  { userId: 'user1-id', decision: 'Approve', justification: 'Active employee' },
  { userId: 'user2-id', decision: 'Deny', justification: 'No longer needs access' },
];

for (const decision of reviewDecisions) {
  await graphClient
    .identityGovernance
    .accessReviews
    .definitions(reviewDefinitionId)
    .instances(instanceId)
    .decisions(decision.userId)
    .patch({
      decision: decision.decision,
      justification: decision.justification,
    });
}
```

## Decisiones automáticas

### Baseline: Remover usuarios inactivos

```powershell
settings = @{
    defaultDecisionEnabled = $true
    defaultDecision = "Deny"  # Si reviewer no responde
    inactiveUserRecommendationsEnabled = $true
    inactiveUserRecommendationsDays = 90  # Inactivos >90 días → Deny
}
```

### Machine Learning recommendations

Azure AD analiza patrones de acceso:

```
User: Juan
Last sign-in to this app: 120 days ago
Peers with similar role: 95% don't have this access
ML Recommendation: DENY
```

```powershell
settings = @{
    recommendationsEnabled = $true
    recommendationLookBackDuration = "P30D"  # 30 días de lookback
}
```

## Notificaciones customizadas

### Email templates

```powershell
# Personalizar emails
$params = @{
    displayName = "Custom Access Review"
    settings = @{
        mailNotificationsEnabled = $true
        reminderNotificationsEnabled = $true
        applyActions = @(
            @{
                "@odata.type" = "#microsoft.graph.removeAccessApplyAction"
            }
        )
    }
    additionalNotificationRecipients = @(
        @{
            notificationRecipients = @(
                @{
                    emailAddress = "security-team@empresa.com"
                }
            )
            notificationTemplateType = "CompletedAdditionalRecipients"
        }
    )
}
```

### Reminders

```
Day 1: Initial notification
Day 7: Reminder (50% completion)
Day 12: Final reminder (deadline approaching)
Day 14: Review closes
Day 15: Auto-apply decisions
```

## Reporting y Analytics

### Ver resultados de reviews

```powershell
# Obtener resultados de access review
$reviewId = "[review-id]"

$decisions = Get-MgIdentityGovernanceAccessReviewDefinitionInstanceDecision `
    -AccessReviewScheduleDefinitionId $reviewId `
    -AccessReviewInstanceId $instanceId

foreach ($decision in $decisions) {
    [PSCustomObject]@{
        User = $decision.Principal.DisplayName
        Decision = $decision.Decision
        Justification = $decision.Justification
        ReviewedBy = $decision.ReviewedBy.DisplayName
        ReviewedDate = $decision.ReviewedDateTime
    }
}
```

### Dashboard en Power BI

```kusto
// Query para Azure Monitor
AuditLogs
| where OperationName startswith "Review access"
| summarize
    Approved = countif(Result == "success" and Properties.decision == "Approve"),
    Denied = countif(Result == "success" and Properties.decision == "Deny"),
    NotReviewed = countif(Result == "timeout")
  by bin(TimeGenerated, 1d), ReviewName = tostring(Properties.reviewName)
| render timechart
```

## Automatización post-review

### Remover usuario de todos los grupos si denegado

```powershell
# Trigger cuando access review completa
$webhook = Register-AzureRMEventHubConsumerGroup ... # Configurar webhook

# Handler
function Handle-AccessReviewCompleted {
    param($reviewResults)

    foreach ($result in $reviewResults) {
        if ($result.Decision -eq "Deny") {
            $userId = $result.UserId

            # Remover de TODOS los grupos
            $groups = Get-MgUserMemberOf -UserId $userId

            foreach ($group in $groups) {
                Remove-MgGroupMemberByRef -GroupId $group.Id -DirectoryObjectId $userId
                Write-Log "Removed $userId from group $($group.DisplayName)"
            }

            # Deshabilitar cuenta si es guest
            $user = Get-MgUser -UserId $userId
            if ($user.UserType -eq "Guest") {
                Update-MgUser -UserId $userId -AccountEnabled:$false
            }

            # Notificar
            Send-MailMessage -To $user.Mail -Subject "Access Removed" `
                -Body "Your access has been removed following access review."
        }
    }
}
```

## Integration con Conditional Access

Bloquear acceso si review pendiente:

```powershell
# Crear CA policy
New-AzureADMSConditionalAccessPolicy -DisplayName "Block if Access Review Pending" `
    -Conditions @{
        Users = @{
            IncludeUsers = "All"
        }
        Applications = @{
            IncludeApplications = "app-id"
        }
    } `
    -GrantControls @{
        Operator = "AND"
        BuiltInControls = @("mfa")
        CustomAuthenticationFactors = @()
        TermsOfUse = @()
    } `
    -SessionControls @{
        SignInFrequency = @{
            Value = 1
            Type = "hours"
            IsEnabled = $true
        }
    }
```

## Mejores prácticas

### 1. Frecuencia apropiada

- **Grupos críticos** (admins, finance): Mensual
- **Grupos estándar**: Trimestral
- **Guest users**: Mensual
- **App roles**: Semestral

### 2. Reviewers correctos

```
Group owners → Saben quién necesita acceso
Managers → Para acceso de empleados
Resource owners → Para aplicaciones específicas
Fallback → Security team (si owner no responde)
```

### 3. Enable recommendations

ML recommendations mejoran accuracy en 40%:

```powershell
recommendationsEnabled = $true
```

### 4. Require justification

```powershell
justificationRequiredOnApproval = $true
```

Auditoría completa de por qué se aprobó acceso.

### 5. Auto-apply decisions

```powershell
autoApplyDecisionsEnabled = $true
```

Sin esto, decisiones son manuales (no escala).

## Monitoreo

```kusto
// Access reviews no completados a tiempo
AuditLogs
| where TimeGenerated > ago(30d)
| where OperationName == "Complete access review"
| where Result == "timeout"
| summarize count() by ReviewName = tostring(Properties.reviewName)
| order by count_ desc
```

Alerta si completion rate <80%:

```powershell
# Alert rule
$condition = @{
    query = "AuditLogs | where OperationName == 'Complete access review' | summarize CompletionRate = countif(Result == 'success') * 100.0 / count()"
    threshold = 80
    operator = "LessThan"
}
```

## Resultados

**Antes** (sin Access Reviews):
- 200+ cuentas zombies
- 0 revisión de permisos
- Compliance audits fallaban
- Riesgo de seguridad alto

**Después** (con Access Reviews):
- 95% de cuentas activas (200 removidas automáticamente)
- Revisión trimestral automática
- Compliance audits passing
- Reducción de ataque surface 40%

**Tiempo de IT**:
- Setup inicial: 8 horas
- Mantenimiento: 2 horas/trimestre
- ROI positivo desde trimestre 2

¿Usas Access Reviews en tu organización? ¿Qué frecuencia funciona mejor?
