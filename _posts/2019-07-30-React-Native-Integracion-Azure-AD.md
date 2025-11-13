---
layout: post
title: React Native + Azure AD - Autenticación móvil con MSAL
author: Vicente José Moreno Escobar
categories: [React Native, Azure AD, Mobile]
published: true
---

> Integrando autenticación empresarial en apps React Native

# El reto

Necesitábamos una app móvil (iOS + Android) que:
- Autenticara usuarios con Azure AD corporativo
- Accediera a APIs protegidas con OAuth 2.0
- Soportara SSO (Single Sign-On)
- Funcionara offline con token refresh

React Native parecía ideal, pero la integración con Azure AD tenía sus particularidades.

## MSAL vs librerías custom

Opciones para autenticación:

1. **Implementar OAuth manualmente**: Mucho trabajo, fácil meter la pata
2. **react-native-app-auth**: Genérico, funciona pero sin features específicos de Azure AD
3. **react-native-msal**: Librería oficial de Microsoft

Elegimos **react-native-msal** (ahora parte de @azure/msal-react-native).

## Instalación y configuración

```bash
npm install @azure/msal-react-native
npx pod-install # iOS
```

### Configuración iOS

```xml
<!-- ios/MyApp/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>msauth.com.empresa.myapp</string>
        </array>
    </dict>
</array>

<key>LSApplicationQueriesSchemes</key>
<array>
    <string>msauthv2</string>
    <string>msauthv3</string>
</array>
```

### Configuración Android

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity
    android:name="com.microsoft.identity.client.BrowserTabActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="msauth"
            android:host="com.empresa.myapp"
            android:path="/signature_hash" />
    </intent-filter>
</activity>
```

Generar signature hash para Android:

```bash
keytool -exportcert -alias androiddebugkey \
  -keystore ~/.android/debug.keystore | \
  openssl sha1 -binary | \
  openssl base64
```

## Configuración en Azure Portal

### 1. Registrar aplicación

```
Azure AD → App registrations → New registration

Name: MyApp Mobile
Supported account types: Accounts in this organizational directory only
Redirect URI:
  - Platform: Mobile and desktop applications
  - URI: msauth.com.empresa.myapp://auth
```

### 2. Configurar API permissions

```
API permissions → Add a permission
  - Microsoft Graph → Delegated permissions
    - User.Read
    - profile
    - openid
    - offline_access
```

### 3. Obtener Client ID

```
Overview → Application (client) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

## Código de autenticación

```typescript
// src/services/AuthService.ts
import { PublicClientApplication } from '@azure/msal-react-native';

const msalConfig = {
  auth: {
    clientId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
    authority: 'https://login.microsoftonline.com/common',
    redirectUri: 'msauth.com.empresa.myapp://auth',
  },
};

const pca = new PublicClientApplication(msalConfig);

export class AuthService {
  private static instance: AuthService;
  private pca: PublicClientApplication;

  private constructor() {
    this.pca = new PublicClientApplication(msalConfig);
  }

  static getInstance(): AuthService {
    if (!AuthService.instance) {
      AuthService.instance = new AuthService();
    }
    return AuthService.instance;
  }

  async signIn(): Promise<string> {
    try {
      const result = await this.pca.acquireToken({
        scopes: ['User.Read', 'api://backend-api/.default'],
      });

      return result.accessToken;
    } catch (error) {
      if (error.errorCode === 'user_cancelled') {
        throw new Error('Login cancelled by user');
      }
      throw error;
    }
  }

  async signInSilent(): Promise<string | null> {
    try {
      const accounts = await this.pca.getAccounts();

      if (accounts.length === 0) {
        return null;
      }

      const result = await this.pca.acquireTokenSilent({
        scopes: ['User.Read'],
        account: accounts[0],
      });

      return result.accessToken;
    } catch (error) {
      // Token expirado, necesita login interactivo
      return null;
    }
  }

  async signOut(): Promise<void> {
    const accounts = await this.pca.getAccounts();

    if (accounts.length > 0) {
      await this.pca.removeAccount(accounts[0]);
    }
  }

  async getAccessToken(): Promise<string> {
    // Intentar silent primero
    let token = await this.signInSilent();

    if (!token) {
      // Fallback a login interactivo
      token = await this.signIn();
    }

    return token;
  }
}
```

## Componente de Login

```typescript
// src/screens/LoginScreen.tsx
import React, { useState } from 'react';
import { View, Button, Text, ActivityIndicator } from 'react-native';
import { AuthService } from '../services/AuthService';

export const LoginScreen: React.FC = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleLogin = async () => {
    setLoading(true);
    setError(null);

    try {
      const token = await AuthService.getInstance().signIn();
      console.log('Login successful, token:', token.substring(0, 20) + '...');

      // Navegar a app principal
      // navigation.navigate('Home');
    } catch (err) {
      setError(err.message || 'Login failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={{ flex: 1, justifyContent: 'center', padding: 20 }}>
      <Text style={{ fontSize: 24, marginBottom: 20, textAlign: 'center' }}>
        MyApp
      </Text>

      {error && (
        <Text style={{ color: 'red', marginBottom: 10 }}>{error}</Text>
      )}

      {loading ? (
        <ActivityIndicator size="large" />
      ) : (
        <Button title="Sign in with Microsoft" onPress={handleLogin} />
      )}
    </View>
  );
};
```

## Llamadas autenticadas a APIs

```typescript
// src/services/ApiService.ts
import axios from 'axios';
import { AuthService } from './AuthService';

const apiClient = axios.create({
  baseURL: 'https://api.empresa.com',
});

// Interceptor para añadir token automáticamente
apiClient.interceptors.request.use(
  async (config) => {
    const token = await AuthService.getInstance().getAccessToken();

    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Interceptor para manejar 401 (token expirado)
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Token expirado, intentar renovar
      try {
        const newToken = await AuthService.getInstance().signIn();
        error.config.headers.Authorization = `Bearer ${newToken}`;
        return apiClient.request(error.config);
      } catch (refreshError) {
        // Refresh falló, forzar logout
        await AuthService.getInstance().signOut();
        // Navegar a login
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);

export const ApiService = {
  getUserProfile: async () => {
    const response = await apiClient.get('/me');
    return response.data;
  },

  getDocuments: async () => {
    const response = await apiClient.get('/documents');
    return response.data;
  },
};
```

## Manejo de estado global con Context

```typescript
// src/contexts/AuthContext.tsx
import React, { createContext, useState, useEffect, useContext } from 'react';
import { AuthService } from '../services/AuthService';

interface AuthContextData {
  isAuthenticated: boolean;
  isLoading: boolean;
  signIn: () => Promise<void>;
  signOut: () => Promise<void>;
}

const AuthContext = createContext<AuthContextData>({} as AuthContextData);

export const AuthProvider: React.FC = ({ children }) => {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Check si ya hay sesión al iniciar app
    checkAuth();
  }, []);

  const checkAuth = async () => {
    try {
      const token = await AuthService.getInstance().signInSilent();
      setIsAuthenticated(!!token);
    } catch (error) {
      setIsAuthenticated(false);
    } finally {
      setIsLoading(false);
    }
  };

  const signIn = async () => {
    await AuthService.getInstance().signIn();
    setIsAuthenticated(true);
  };

  const signOut = async () => {
    await AuthService.getInstance().signOut();
    setIsAuthenticated(false);
  };

  return (
    <AuthContext.Provider value={{ isAuthenticated, isLoading, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

## Navegación condicional

```typescript
// App.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';
import { AuthProvider, useAuth } from './contexts/AuthContext';
import { LoginScreen } from './screens/LoginScreen';
import { HomeScreen } from './screens/HomeScreen';

const Stack = createStackNavigator();

const Navigation = () => {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) {
    return <LoadingScreen />;
  }

  return (
    <NavigationContainer>
      <Stack.Navigator>
        {isAuthenticated ? (
          <>
            <Stack.Screen name="Home" component={HomeScreen} />
            {/* Otras pantallas autenticadas */}
          </>
        ) : (
          <Stack.Screen
            name="Login"
            component={LoginScreen}
            options={{ headerShown: false }}
          />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
};

export default function App() {
  return (
    <AuthProvider>
      <Navigation />
    </AuthProvider>
  );
}
```

## Problemas que enfrenté

### 1. Redirect URI no coincide

```
Error: AADSTS50011: The reply URL specified in the request does not match
```

**Solución**: Asegurar que redirect URI en código coincide EXACTAMENTE con Azure Portal:

```typescript
// Debe ser EXACTAMENTE igual
redirectUri: 'msauth.com.empresa.myapp://auth'
```

### 2. Android signature hash incorrecto

Para release builds, el hash es diferente:

```bash
# Release keystore
keytool -exportcert -alias myapp-release \
  -keystore android/app/myapp-release.keystore | \
  openssl sha1 -binary | \
  openssl base64
```

### 3. Tokens expirando demasiado rápido

Configurar token lifetime en Azure AD:

```
Azure AD → App registrations → Token configuration
→ Optional claims → Add groups claim
→ Access token lifetime: 60 minutos (ajustar según necesidad)
```

## Testing

```typescript
// __tests__/AuthService.test.ts
import { AuthService } from '../src/services/AuthService';

jest.mock('@azure/msal-react-native');

describe('AuthService', () => {
  it('should sign in successfully', async () => {
    const service = AuthService.getInstance();
    const token = await service.signIn();

    expect(token).toBeDefined();
    expect(typeof token).toBe('string');
  });

  it('should return null on silent sign-in when no account', async () => {
    const service = AuthService.getInstance();
    const token = await service.signInSilent();

    expect(token).toBeNull();
  });
});
```

## Resultados

- Autenticación corporativa funcionando en iOS y Android
- SSO automático si usuario ya logueado en browser
- Token refresh transparente
- 500+ usuarios activos sin incidencias

¿Has integrado Azure AD en apps móviles? ¿Qué problemas encontraste?
