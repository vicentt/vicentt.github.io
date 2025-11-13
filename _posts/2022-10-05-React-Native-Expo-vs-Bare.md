---
layout: post
title: Expo vs React Native Bare - Cuándo usar cada uno
author: Vicente José Moreno Escobar
categories: [React Native, Mobile, Expo]
published: true
---

> La eterna pregunta: ¿Expo o React Native puro?

# El dilema

Cada proyecto React Native empieza con la misma pregunta: **¿Expo o bare workflow?**

Después de 5 proyectos con Expo y 3 con bare, tengo opiniones claras.

## ¿Qué es Expo?

Framework sobre React Native que simplifica:
- Setup (0 config de Xcode/Android Studio)
- Updates OTA (Over The Air)
- APIs comunes (camera, location, notifications)
- Build en la nube

## ¿Qué es bare workflow?

React Native puro:
- Control total
- Acceso a native code
- Cualquier librería native
- Más complejo de configurar

## Proyecto 1: App de delivery (elegimos Expo)

### Requisitos

- Geolocalización
- Push notifications
- Camera para fotos
- Updates frecuentes sin app store review

### Por qué Expo

```bash
# Setup en 2 minutos
npx create-expo-app delivery-app
cd delivery-app
npm start
```

vs bare workflow:

```bash
npx react-native init DeliveryApp
cd DeliveryApp
# Instalar Xcode, CocoaPods
# Configurar Android Studio
# Configurar Gradle
# ... 2 horas después
```

### APIs que usamos

```typescript
// Location - funciona out of the box
import * as Location from 'expo-location';

const location = await Location.getCurrentPositionAsync({});

// Camera
import { Camera } from 'expo-camera';

const { status } = await Camera.requestCameraPermissionsAsync();

// Push notifications
import * as Notifications from 'expo-notifications';

const token = await Notifications.getExpoPushTokenAsync();
```

### Updates OTA

```bash
# Publicar update sin pasar por App Store
eas update --branch production --message "Fix delivery bug"

# Usuarios reciben update automáticamente
```

**Resultado**: App en prod en 2 semanas. Expo fue perfecta.

## Proyecto 2: App bancaria (elegimos bare)

### Requisitos

- Seguridad máxima
- Integración con SDKs bancarios nativos
- Biometría avanzada
- Sin updates OTA (regulación)

### Por qué NO Expo

```typescript
// SDK bancario requiere native code
import BankSDK from 'bank-proprietary-sdk';  // ❌ No funciona en Expo

// Necesitábamos modificar AndroidManifest.xml
// Necesitábamos agregar códigos nativos en iOS
```

### Migración a bare

```bash
expo eject  # Ahora deprecado
# O mejor:
npx expo prebuild  # Genera ios/ y android/
```

Pero perdimos:
- OTA updates
- Expo Go para testing
- Simplicidad de builds

### Lo que ganamos

```java
// android/app/src/main/java/.../MainActivity.java
// Código Java nativo para integración bancaria

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    BankSDK.initialize(this, "api-key");
}
```

```swift
// ios/DeliveryApp/AppDelegate.m
// Código Swift nativo

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [BankSDK initializeWithApiKey:@"api-key"];
    return YES;
}
```

**Resultado**: Más trabajo pero necesario para el proyecto.

## Comparativa técnica

| Feature | Expo | Bare |
|---------|------|------|
| Setup time | 2 min | 2 hours |
| Native code | ❌ No | ✅ Sí |
| OTA updates | ✅ Sí | Con CodePush |
| Build | EAS Build (cloud) | Local |
| Size de app | Más grande | Optimizado |
| Librerias | Solo compatibles con Expo | Cualquiera |

## Expo Application Services (EAS)

Gran mejora sobre Expo clásico:

### EAS Build

```bash
# Build iOS sin Mac
eas build --platform ios

# Build Android
eas build --platform android

# Builds en la nube de Expo
```

### EAS Submit

```bash
# Subir a App Store automáticamente
eas submit --platform ios

# Subir a Google Play
eas submit --platform android
```

### EAS Update

```bash
# OTA update
eas update --branch production
```

### Configuración

```json
// eas.json
{
  "build": {
    "production": {
      "android": {
        "buildType": "apk"
      },
      "ios": {
        "bundleIdentifier": "com.empresa.app",
        "buildConfiguration": "Release"
      }
    },
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "developer@empresa.com",
        "ascAppId": "123456789"
      },
      "android": {
        "serviceAccountKeyPath": "./google-play-key.json",
        "track": "internal"
      }
    }
  }
}
```

## Development Build (lo mejor de ambos mundos)

Expo ahora permite usar librerías nativas:

```bash
# Crear development build
npx expo install expo-dev-client

# Agregar librería con native code
npx expo install react-native-ble-plx

# Rebuild
eas build --profile development
```

Ahora puedes:
- ✅ Usar librerías con código nativo
- ✅ Mantener simplicidad de Expo
- ✅ OTA updates
- ✅ EAS services

**Esto cambia todo**. Ya no necesitas elegir.

## Cuándo usar Expo (con dev build)

✅ **Casi siempre**

Excepto si:
- Necesitas modificar native build scripts extensivamente
- Tienes SDKs propietarios muy complejos
- Regulación prohíbe OTA updates

## Cuándo usar bare workflow

❌ **Raramente**

Solo si:
- Tienes código legacy native
- Build process muy customizado
- No quieres depender de Expo

## Proyecto 3: E-commerce (Expo + dev build)

Mejor de ambos mundos:

```bash
npx create-expo-app ecommerce-app

# Instalar librerías (incluso con native code)
npx expo install @stripe/stripe-react-native
npx expo install react-native-vision-camera
npx expo install react-native-reanimated

# Create development build
eas build --profile development --platform ios

# Uso normal
npm start
```

**Resultado**:
- Setup rápido ✅
- Librerías nativas ✅
- OTA updates ✅
- EAS services ✅

## Costo

### Expo Free

- OTA updates ilimitados
- EAS Build: 30 builds/mes
- Perfecto para proyectos pequeños

### Expo Production ($99/mes)

- EAS Build ilimitado
- Priority build queue
- Support

### Bare workflow

- Gratis
- Pero necesitas Mac para builds iOS
- Tiempo de setup vale dinero

## Migración de bare a Expo

¿Ya tienes app en bare? Puedes migrar:

```bash
# Instalar Expo en proyecto existente
npx install-expo-modules

# Configurar
npx pod-install  # iOS
```

Ahora puedes usar Expo modules sin reescribir todo.

## Recomendación 2024

**Usar Expo con development builds** para 95% de proyectos.

Bare workflow solo si tienes razón MUY específica.

## Nuestros resultados

**Apps con Expo**:
- Time to market: -60%
- Developer experience: 9/10
- Issues con librerías: 5% de casos

**Apps con bare**:
- Control total: ✅
- Complejidad: 3x
- Solo valió la pena en 1 de 3 proyectos

¿Usas Expo o bare? ¿Por qué?
