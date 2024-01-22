---
layout: post
title: Conexión mediante Client Credentials en Microsoft Azure AD B2C
image: /images/Azure-B2C.png
author: Vicente José Moreno Escobar
categories: [Azure, B2C, Seguridad]
published: true
---

![Azure B2C](/images/b2c.png)

La autenticación es esencial en el desarrollo de aplicaciones seguras, y Microsoft Azure Active Directory B2C (Azure AD B2C) proporciona una solución robusta para la gestión de identidad. En este artículo, exploraremos cómo establecer una conexión utilizando el flujo de Client Credentials en Azure AD B2C.

## Paso 1: Crear una Aplicación en Azure AD B2C

1. Inicia sesión en el [Portal de Azure](https://portal.azure.com/).

2. Navega a Azure AD B2C y selecciona tu instancia.

3. En "App registrations", crea una nueva aplicación y anota el "Application (client) ID" y el "Directory (tenant) ID".

## Paso 2: Configurar el Flujo de Client Credentials

1. En la configuración de la aplicación, ve a "Certificates & Secrets" y crea un nuevo secreto. Guarda el valor del secreto generado.

2. En "API permissions", agrega los permisos necesarios para tu aplicación, seleccionando "Application permissions".

## Paso 3: Implementar la Autenticación en tu Aplicación

A continuación, presentamos un ejemplo utilizando Node.js y la biblioteca `msal-node`.

```javascript
const { ConfidentialClientApplication } = require('@azure/msal-node');

const config = {
  auth: {
    clientId: 'TU_CLIENT_ID',
    authority: 'https://TENANT_NAME.b2clogin.com/TU_TENANT_NAME.onmicrosoft.com/TU_POLICY_NAME',
    clientSecret: 'TU_CLIENT_SECRET',
  },
};

const cca = new ConfidentialClientApplication(config);

async function getToken() {
  const tokenRequest = {
    scopes: ['https://graph.microsoft.com/.default'],
  };

  try {
    const result = await cca.acquireTokenByClientCredential(tokenRequest);
    console.log('Token adquirido:', result.accessToken);
  } catch (error) {
    console.error('Error al adquirir el token:', error);
  }
}

getToken();
```
Asegúrate de reemplazar 'TU_CLIENT_ID', 'TU_TENANT_NAME', 'TU_POLICY_NAME' y 'TU_CLIENT_SECRET' con los valores correspondientes.

Con estos pasos, has implementado la autenticación mediante el flujo de Client Credentials en Microsoft Azure AD B2C. Este enfoque es útil cuando tu aplicación necesita acceder a recursos protegidos sin la intervención del usuario, proporcionando una capa adicional de seguridad y eficiencia en la gestión de identidades.
