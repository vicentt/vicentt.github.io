---
layout: post
title: Azure AD B2C Custom Policies - Más allá de los User Flows
author: Vicente José Moreno Escobar
categories: [Azure, Azure AD B2C, Autenticación]
published: true
---

> Cuando los User Flows no son suficientes y necesitas control total

# Por qué Custom Policies

Los User Flows de Azure AD B2C son geniales para casos comunes:
- Sign up/Sign in
- Password reset
- Profile editing

Pero nuestro cliente necesitaba:
- Validación custom durante registro (verificar que email sea corporativo)
- Integración con API externa para verificar elegibilidad
- Flujo multi-step con lógica condicional
- Personalización extrema de claims

Los User Flows no podían hacerlo. Tocaba usar **Custom Policies** (Identity Experience Framework).

## Custom Policies vs User Flows

| User Flows | Custom Policies |
|------------|----------------|
| Configuración visual | XML files |
| Casos comunes | Cualquier escenario |
| Fácil de configurar | Curva de aprendizaje empinada |
| Limitado | Control total |

## Arquitectura de Custom Policies

Las policies son archivos XML que se heredan:

```
TrustFrameworkBase.xml
    ↓
TrustFrameworkExtensions.xml
    ↓
SignUpOrSignIn.xml
```

### TrustFrameworkBase.xml

Viene del Starter Pack de Microsoft. Contiene building blocks:

```xml
<TrustFrameworkPolicy
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  PolicySchemaVersion="0.3.0.0"
  TenantId="tutenantb2c.onmicrosoft.com"
  PolicyId="B2C_1A_TrustFrameworkBase">

  <BuildingBlocks>
    <ClaimsSchema>
      <ClaimType Id="email">
        <DisplayName>Email Address</DisplayName>
        <DataType>string</DataType>
      </ClaimType>
      <!-- Más claims -->
    </ClaimsSchema>

    <ClaimsTransformations>
      <ClaimsTransformation Id="CreateRandomUPNUserName"
        TransformationMethod="CreateRandomString">
        <!-- Lógica transformación -->
      </ClaimsTransformation>
    </ClaimsTransformations>
  </BuildingBlocks>

  <ClaimsProviders>
    <!-- Providers de identidad: Azure AD, Google, Facebook, etc. -->
  </ClaimsProviders>
</TrustFrameworkPolicy>
```

### TrustFrameworkExtensions.xml

Aquí extendemos la base con nuestra lógica:

```xml
<TrustFrameworkPolicy
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  PolicySchemaVersion="0.3.0.0"
  TenantId="tutenantb2c.onmicrosoft.com"
  PolicyId="B2C_1A_TrustFrameworkExtensions"
  PublicPolicyUri="http://tutenantb2c.onmicrosoft.com/B2C_1A_TrustFrameworkExtensions">

  <BasePolicy>
    <TenantId>tutenantb2c.onmicrosoft.com</TenantId>
    <PolicyId>B2C_1A_TrustFrameworkBase</PolicyId>
  </BasePolicy>

  <BuildingBlocks>
    <!-- Nuestros claims custom -->
  </BuildingBlocks>

  <ClaimsProviders>
    <ClaimsProvider>
      <DisplayName>REST API Validation</DisplayName>
      <TechnicalProfiles>
        <TechnicalProfile Id="REST-ValidateUser">
          <DisplayName>Validate user eligibility</DisplayName>
          <Protocol Name="Proprietary"
            Handler="Web.TPEngine.Providers.RestfulProvider, Web.TPEngine, Version=1.0.0.0" />
          <Metadata>
            <Item Key="ServiceUrl">https://api.empresa.com/validate</Item>
            <Item Key="SendClaimsIn">Body</Item>
            <Item Key="AuthenticationType">None</Item>
          </Metadata>
          <InputClaims>
            <InputClaim ClaimTypeReferenceId="email" />
          </InputClaims>
          <OutputClaims>
            <OutputClaim ClaimTypeReferenceId="isEligible" />
          </OutputClaims>
        </TechnicalProfile>
      </TechnicalProfiles>
    </ClaimsProvider>
  </ClaimsProviders>
</TrustFrameworkPolicy>
```

## Caso real: Validación de email corporativo

Necesitábamos que solo emails `@empresa.com` pudieran registrarse.

### 1. Definir claim custom

```xml
<!-- TrustFrameworkExtensions.xml -->
<BuildingBlocks>
  <ClaimsSchema>
    <ClaimType Id="isValidCorporateEmail">
      <DisplayName>Is Valid Corporate Email</DisplayName>
      <DataType>boolean</DataType>
    </ClaimType>
  </ClaimsSchema>

  <ClaimsTransformations>
    <ClaimsTransformation Id="CheckEmailDomain"
      TransformationMethod="CompareClaims">
      <InputClaims>
        <InputClaim ClaimTypeReferenceId="email"
          TransformationClaimType="inputClaim1" />
      </InputClaims>
      <InputParameters>
        <InputParameter Id="operator" DataType="string" Value="CONTAINS" />
        <InputParameter Id="compareTo" DataType="string" Value="@empresa.com" />
      </InputParameters>
      <OutputClaims>
        <OutputClaim ClaimTypeReferenceId="isValidCorporateEmail"
          TransformationClaimType="outputClaim" />
      </OutputClaims>
    </ClaimsTransformation>
  </ClaimsTransformations>
</BuildingBlocks>
```

### 2. Validar en Technical Profile

```xml
<ClaimsProvider>
  <DisplayName>Local Account SignIn</DisplayName>
  <TechnicalProfiles>
    <TechnicalProfile Id="LocalAccountSignUpWithLogonEmail">
      <DisplayName>Email signup</DisplayName>
      <ValidationTechnicalProfiles>
        <ValidationTechnicalProfile ReferenceId="Validate-CorporateEmail" />
      </ValidationTechnicalProfiles>
    </TechnicalProfile>

    <TechnicalProfile Id="Validate-CorporateEmail">
      <DisplayName>Validate Corporate Email</DisplayName>
      <Protocol Name="Proprietary"
        Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine" />
      <OutputClaims>
        <OutputClaim ClaimTypeReferenceId="isValidCorporateEmail" />
      </OutputClaims>
      <OutputClaimsTransformations>
        <OutputClaimsTransformation ReferenceId="CheckEmailDomain" />
      </OutputClaimsTransformations>
      <UseTechnicalProfileForSessionManagement ReferenceId="SM-Noop" />
    </TechnicalProfile>
  </TechnicalProfiles>
</ClaimsProvider>
```

### 3. Mostrar error si falla validación

```xml
<TechnicalProfile Id="Validate-CorporateEmail">
  <!-- ... -->
  <Metadata>
    <Item Key="UserMessageIfClaimsTransformationBooleanValueIsNotEqual">
      Solo emails corporativos (@empresa.com) pueden registrarse
    </Item>
  </Metadata>
</TechnicalProfile>
```

## Llamar API REST durante el flujo

Necesitábamos verificar elegibilidad con sistema externo.

### 1. API externa

```csharp
// ASP.NET Core API
[ApiController]
[Route("api/[controller]")]
public class ValidateController : ControllerBase
{
    [HttpPost]
    public IActionResult ValidateUser([FromBody] ValidationRequest request)
    {
        // Verificar en base de datos si email es elegible
        var isEligible = _eligibilityService.Check(request.Email);

        if (!isEligible)
        {
            return BadRequest(new
            {
                version = "1.0.0",
                status = 400,
                userMessage = "No eres elegible para este servicio"
            });
        }

        return Ok(new
        {
            version = "1.0.0",
            status = 200,
            isEligible = true
        });
    }
}
```

### 2. Technical Profile para REST API

```xml
<TechnicalProfile Id="REST-ValidateEligibility">
  <DisplayName>Validate user eligibility</DisplayName>
  <Protocol Name="Proprietary"
    Handler="Web.TPEngine.Providers.RestfulProvider, Web.TPEngine" />
  <Metadata>
    <Item Key="ServiceUrl">https://api.empresa.com/api/validate</Item>
    <Item Key="SendClaimsIn">Body</Item>
    <Item Key="AuthenticationType">Bearer</Item>
    <Item Key="UseClaimAsBearerToken">bearerToken</Item>
    <Item Key="AllowInsecureAuthInProduction">false</Item>
  </Metadata>
  <CryptographicKeys>
    <Key Id="BearerAuthenticationToken" StorageReferenceId="B2C_1A_RestApiKey" />
  </CryptographicKeys>
  <InputClaims>
    <InputClaim ClaimTypeReferenceId="email" PartnerClaimType="email" />
  </InputClaims>
  <OutputClaims>
    <OutputClaim ClaimTypeReferenceId="isEligible" PartnerClaimType="isEligible" />
  </OutputClaims>
  <UseTechnicalProfileForSessionManagement ReferenceId="SM-Noop" />
</TechnicalProfile>
```

### 3. Configurar API key en portal

```
Azure AD B2C → Identity Experience Framework → Policy keys
→ Add → Manual
Name: RestApiKey
Secret: tu-api-key-secreta
```

## Flujo multi-step

Registro en múltiples pasos:

```xml
<UserJourney Id="SignUpOrSignIn">
  <OrchestrationSteps>
    <!-- Paso 1: Elegir método (local o social) -->
    <OrchestrationStep Order="1" Type="CombinedSignInAndSignUp">
      <ClaimsProviderSelections>
        <ClaimsProviderSelection ValidationClaimsExchangeId="LocalAccountSigninEmailExchange" />
        <ClaimsProviderSelection TargetClaimsExchangeId="GoogleExchange" />
      </ClaimsProviderSelections>
    </OrchestrationStep>

    <!-- Paso 2: Recoger info básica -->
    <OrchestrationStep Order="2" Type="ClaimsExchange">
      <ClaimsExchanges>
        <ClaimsExchange Id="SignUpWithLogonEmailExchange"
          TechnicalProfileReferenceId="LocalAccountSignUpWithLogonEmail" />
      </ClaimsExchanges>
    </OrchestrationStep>

    <!-- Paso 3: Validar con API externa -->
    <OrchestrationStep Order="3" Type="ClaimsExchange">
      <ClaimsExchanges>
        <ClaimsExchange Id="ValidateEligibilityExchange"
          TechnicalProfileReferenceId="REST-ValidateEligibility" />
      </ClaimsExchanges>
    </OrchestrationStep>

    <!-- Paso 4: Recoger info adicional -->
    <OrchestrationStep Order="4" Type="ClaimsExchange">
      <ClaimsExchanges>
        <ClaimsExchange Id="AADUserWriteUsingAlternativeSecurityId"
          TechnicalProfileReferenceId="AAD-UserWriteUsingAlternativeSecurityId" />
      </ClaimsExchanges>
    </OrchestrationStep>

    <!-- Paso 5: Emitir token -->
    <OrchestrationStep Order="5" Type="SendClaims"
      CpimIssuerTechnicalProfileReferenceId="JwtIssuer" />
  </OrchestrationSteps>
</UserJourney>
```

## Personalización de emails

Custom Policies permiten personalizar completamente los emails de verificación:

```xml
<BuildingBlocks>
  <ContentDefinitions>
    <ContentDefinition Id="api.localaccountpasswordreset">
      <LoadUri>~/tenant/templates/AzureBlue/emailVerification.cshtml</LoadUri>
      <DataUri>urn:com:microsoft:aad:b2c:elements:contract:emailverification:2.1.0</DataUri>
    </ContentDefinition>
  </ContentDefinitions>

  <Localization>
    <LocalizationResources Id="api.localaccountpasswordreset.es">
      <LocalizedStrings>
        <LocalizedString ElementType="UxElement" StringId="verification_code_input_placeholder_text">
          Introduce tu código
        </LocalizedString>
        <LocalizedString ElementType="UxElement" StringId="button_verify">
          Verificar
        </LocalizedString>
      </LocalizedStrings>
    </LocalizationResources>
  </Localization>
</BuildingBlocks>
```

## Debugging Custom Policies

Debugging es complicado. Estas técnicas ayudan:

### 1. Application Insights

```xml
<TrustFrameworkPolicy>
  <UserJourneyRecorderEndpoint>
    urn:journeyrecorder:applicationinsights
  </UserJourneyRecorderEndpoint>
</TrustFrameworkPolicy>

<RelyingParty>
  <UserJourneyBehaviors>
    <JourneyInsights TelemetryEngine="ApplicationInsights"
      InstrumentationKey="tu-instrumentation-key"
      DeveloperMode="true"
      ClientEnabled="true"
      ServerEnabled="true"
      TelemetryVersion="1.0.0" />
  </UserJourneyBehaviors>
</RelyingParty>
```

### 2. Technical Profile debugging

Añadir outputs para ver valores:

```xml
<TechnicalProfile Id="Debug-Claims">
  <DisplayName>Debug Claims</DisplayName>
  <Protocol Name="Proprietary"
    Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine" />
  <OutputClaims>
    <OutputClaim ClaimTypeReferenceId="email" />
    <OutputClaim ClaimTypeReferenceId="isEligible" />
    <!-- Ver valores en pantalla -->
  </OutputClaims>
</TechnicalProfile>
```

## Problemas comunes

### 1. Errores crípticos

```
AADB2C90230: The element 'ClaimsProvider' has incomplete content
```

**Solución**: Validar XML con schema. Hay herramientas online.

### 2. Policy upload falla

```
Validation failed: 1 validation error(s) found in policy "B2C_1A_SIGNUP_SIGNIN"
```

**Solución**: Subir policies en orden correcto:
1. Base
2. Extensions
3. SignUpOrSignIn

### 3. Claims no aparecen en token

**Solución**: Añadir claims a Relying Party:

```xml
<RelyingParty>
  <DefaultUserJourney ReferenceId="SignUpOrSignIn" />
  <TechnicalProfile Id="PolicyProfile">
    <OutputClaims>
      <OutputClaim ClaimTypeReferenceId="email" />
      <OutputClaim ClaimTypeReferenceId="isEligible" />
      <!-- Listar TODOS los claims que quieres en el token -->
    </OutputClaims>
  </TechnicalProfile>
</RelyingParty>
```

## Resultados

Custom Policies nos permitieron:
- Validar emails corporativos automáticamente
- Integrar con sistema de elegibilidad externo
- Flujo de registro multi-step personalizado
- Reducir fraud en un 60%

**Tiempo de implementación**: 3 semanas (vs 2 días con User Flows).
**¿Valió la pena?** Absolutamente. La flexibilidad es invaluable.

¿Has usado Custom Policies de Azure AD B2C? ¿Qué casos de uso has implementado?
