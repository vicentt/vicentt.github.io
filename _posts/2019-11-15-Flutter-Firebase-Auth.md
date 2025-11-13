---
layout: post
title: Flutter + Firebase Auth - Autenticación rápida para MVPs
author: Vicente José Moreno Escobar
categories: [Flutter, Firebase, Mobile]
published: true
---

> Cuando necesitas autenticación funcionando en 2 horas, no en 2 semanas

# El contexto

Prototipo de app móvil para validar idea de negocio. Necesitábamos:
- Login con email/password
- Login con Google
- Reset de password
- Funcionar en iOS y Android
- **Timeline: 2 días**

Firebase Authentication era la respuesta obvia.

## Setup inicial

```bash
# Crear proyecto Flutter
flutter create my_app
cd my_app

# Añadir dependencias
flutter pub add firebase_core
flutter pub add firebase_auth
flutter pub add google_sign_in
```

### Configuración Firebase

1. Crear proyecto en https://console.firebase.google.com
2. Añadir app iOS y Android
3. Descargar `google-services.json` (Android) y `GoogleService-Info.plist` (iOS)

```
android/app/google-services.json
ios/Runner/GoogleService-Info.plist
```

### Configuración Android

```gradle
// android/build.gradle
buildscript {
    dependencies {
        classpath 'com.google.gms:google-services:4.3.15'
    }
}

// android/app/build.gradle
apply plugin: 'com.google.gms.google-services'

defaultConfig {
    minSdkVersion 21 // Firebase requiere mínimo 21
}
```

### Configuración iOS

```ruby
# ios/Podfile
platform :ios, '12.0'
```

```bash
cd ios && pod install
```

## Inicializar Firebase

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      home: AuthWrapper(),
    );
  }
}
```

## AuthService

Centralizar lógica de autenticación:

```dart
// services/auth_service.dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';

class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final GoogleSignIn _googleSignIn = GoogleSignIn();

  // Stream de cambios de autenticación
  Stream<User?> get authStateChanges => _auth.authStateChanges();

  // Usuario actual
  User? get currentUser => _auth.currentUser;

  // Sign up con email/password
  Future<UserCredential?> signUpWithEmail(String email, String password) async {
    try {
      UserCredential userCredential = await _auth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );

      // Enviar email de verificación
      await userCredential.user?.sendEmailVerification();

      return userCredential;
    } on FirebaseAuthException catch (e) {
      if (e.code == 'weak-password') {
        throw Exception('La contraseña es muy débil');
      } else if (e.code == 'email-already-in-use') {
        throw Exception('El email ya está registrado');
      }
      throw Exception(e.message);
    }
  }

  // Sign in con email/password
  Future<UserCredential?> signInWithEmail(String email, String password) async {
    try {
      return await _auth.signInWithEmailAndPassword(
        email: email,
        password: password,
      );
    } on FirebaseAuthException catch (e) {
      if (e.code == 'user-not-found') {
        throw Exception('Usuario no encontrado');
      } else if (e.code == 'wrong-password') {
        throw Exception('Contraseña incorrecta');
      }
      throw Exception(e.message);
    }
  }

  // Sign in con Google
  Future<UserCredential?> signInWithGoogle() async {
    try {
      // Trigger Google Sign In
      final GoogleSignInAccount? googleUser = await _googleSignIn.signIn();

      if (googleUser == null) {
        return null; // Usuario canceló
      }

      // Obtener detalles de autenticación
      final GoogleSignInAuthentication googleAuth = await googleUser.authentication;

      // Crear credential para Firebase
      final credential = GoogleAuthProvider.credential(
        accessToken: googleAuth.accessToken,
        idToken: googleAuth.idToken,
      );

      return await _auth.signInWithCredential(credential);
    } catch (e) {
      throw Exception('Error al iniciar sesión con Google: $e');
    }
  }

  // Password reset
  Future<void> sendPasswordResetEmail(String email) async {
    try {
      await _auth.sendPasswordResetEmail(email: email);
    } catch (e) {
      throw Exception('Error al enviar email de recuperación: $e');
    }
  }

  // Sign out
  Future<void> signOut() async {
    await _googleSignIn.signOut();
    await _auth.signOut();
  }
}
```

## AuthWrapper - Routing condicional

```dart
// widgets/auth_wrapper.dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import '../services/auth_service.dart';
import '../screens/login_screen.dart';
import '../screens/home_screen.dart';

class AuthWrapper extends StatelessWidget {
  final AuthService _authService = AuthService();

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<User?>(
      stream: _authService.authStateChanges,
      builder: (context, snapshot) {
        // Mostrar loading mientras verifica
        if (snapshot.connectionState == ConnectionState.waiting) {
          return Scaffold(
            body: Center(child: CircularProgressIndicator()),
          );
        }

        // Si está autenticado, ir a Home
        if (snapshot.hasData) {
          return HomeScreen();
        }

        // Si no, mostrar Login
        return LoginScreen();
      },
    );
  }
}
```

## Login Screen

```dart
// screens/login_screen.dart
import 'package:flutter/material.dart';
import '../services/auth_service.dart';

class LoginScreen extends StatefulWidget {
  @override
  _LoginScreenState createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final AuthService _authService = AuthService();
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  bool _isLoading = false;
  String? _errorMessage;

  Future<void> _signInWithEmail() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() {
      _isLoading = true;
      _errorMessage = null;
    });

    try {
      await _authService.signInWithEmail(
        _emailController.text.trim(),
        _passwordController.text,
      );
      // AuthWrapper se encarga de la navegación
    } catch (e) {
      setState(() {
        _errorMessage = e.toString();
      });
    } finally {
      setState(() {
        _isLoading = false;
      });
    }
  }

  Future<void> _signInWithGoogle() async {
    setState(() {
      _isLoading = true;
      _errorMessage = null;
    });

    try {
      await _authService.signInWithGoogle();
    } catch (e) {
      setState(() {
        _errorMessage = e.toString();
      });
    } finally {
      setState(() {
        _isLoading = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Iniciar Sesión')),
      body: Padding(
        padding: EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              if (_errorMessage != null)
                Container(
                  padding: EdgeInsets.all(8.0),
                  color: Colors.red[100],
                  child: Text(
                    _errorMessage!,
                    style: TextStyle(color: Colors.red[900]),
                  ),
                ),
              SizedBox(height: 16),
              TextFormField(
                controller: _emailController,
                decoration: InputDecoration(
                  labelText: 'Email',
                  border: OutlineInputBorder(),
                ),
                keyboardType: TextInputType.emailAddress,
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Ingresa tu email';
                  }
                  if (!value.contains('@')) {
                    return 'Email inválido';
                  }
                  return null;
                },
              ),
              SizedBox(height: 16),
              TextFormField(
                controller: _passwordController,
                decoration: InputDecoration(
                  labelText: 'Contraseña',
                  border: OutlineInputBorder(),
                ),
                obscureText: true,
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Ingresa tu contraseña';
                  }
                  if (value.length < 6) {
                    return 'Mínimo 6 caracteres';
                  }
                  return null;
                },
              ),
              SizedBox(height: 24),
              _isLoading
                  ? CircularProgressIndicator()
                  : Column(
                      children: [
                        ElevatedButton(
                          onPressed: _signInWithEmail,
                          child: Text('Iniciar Sesión'),
                          style: ElevatedButton.styleFrom(
                            minimumSize: Size(double.infinity, 50),
                          ),
                        ),
                        SizedBox(height: 16),
                        OutlinedButton.icon(
                          onPressed: _signInWithGoogle,
                          icon: Icon(Icons.login),
                          label: Text('Continuar con Google'),
                          style: OutlinedButton.styleFrom(
                            minimumSize: Size(double.infinity, 50),
                          ),
                        ),
                        SizedBox(height: 16),
                        TextButton(
                          onPressed: () {
                            Navigator.push(
                              context,
                              MaterialPageRoute(
                                builder: (context) => PasswordResetScreen(),
                              ),
                            );
                          },
                          child: Text('¿Olvidaste tu contraseña?'),
                        ),
                      ],
                    ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## Password Reset Screen

```dart
// screens/password_reset_screen.dart
import 'package:flutter/material.dart';
import '../services/auth_service.dart';

class PasswordResetScreen extends StatefulWidget {
  @override
  _PasswordResetScreenState createState() => _PasswordResetScreenState();
}

class _PasswordResetScreenState extends State<PasswordResetScreen> {
  final AuthService _authService = AuthService();
  final _emailController = TextEditingController();

  bool _isLoading = false;
  bool _emailSent = false;
  String? _errorMessage;

  Future<void> _sendResetEmail() async {
    setState(() {
      _isLoading = true;
      _errorMessage = null;
    });

    try {
      await _authService.sendPasswordResetEmail(_emailController.text.trim());
      setState(() {
        _emailSent = true;
      });
    } catch (e) {
      setState(() {
        _errorMessage = e.toString();
      });
    } finally {
      setState(() {
        _isLoading = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Recuperar Contraseña')),
      body: Padding(
        padding: EdgeInsets.all(16.0),
        child: _emailSent
            ? Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(Icons.email, size: 100, color: Colors.green),
                  SizedBox(height: 16),
                  Text(
                    'Email enviado',
                    style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
                  ),
                  SizedBox(height: 8),
                  Text('Revisa tu correo para restablecer tu contraseña'),
                  SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: () => Navigator.pop(context),
                    child: Text('Volver'),
                  ),
                ],
              )
            : Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  if (_errorMessage != null)
                    Container(
                      padding: EdgeInsets.all(8.0),
                      color: Colors.red[100],
                      child: Text(_errorMessage!),
                    ),
                  SizedBox(height: 16),
                  TextField(
                    controller: _emailController,
                    decoration: InputDecoration(
                      labelText: 'Email',
                      border: OutlineInputBorder(),
                    ),
                    keyboardType: TextInputType.emailAddress,
                  ),
                  SizedBox(height: 24),
                  _isLoading
                      ? CircularProgressIndicator()
                      : ElevatedButton(
                          onPressed: _sendResetEmail,
                          child: Text('Enviar Email'),
                          style: ElevatedButton.styleFrom(
                            minimumSize: Size(double.infinity, 50),
                          ),
                        ),
                ],
              ),
      ),
    );
  }
}
```

## Home Screen (ejemplo)

```dart
// screens/home_screen.dart
import 'package:flutter/material.dart';
import '../services/auth_service.dart';

class HomeScreen extends StatelessWidget {
  final AuthService _authService = AuthService();

  @override
  Widget build(BuildContext context) {
    final user = _authService.currentUser;

    return Scaffold(
      appBar: AppBar(
        title: Text('Home'),
        actions: [
          IconButton(
            icon: Icon(Icons.logout),
            onPressed: () async {
              await _authService.signOut();
            },
          ),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Bienvenido!', style: TextStyle(fontSize: 24)),
            SizedBox(height: 16),
            Text('Email: ${user?.email}'),
            SizedBox(height: 8),
            Text('UID: ${user?.uid}'),
            if (!user!.emailVerified)
              Padding(
                padding: EdgeInsets.all(16.0),
                child: Card(
                  color: Colors.orange[100],
                  child: Padding(
                    padding: EdgeInsets.all(16.0),
                    child: Column(
                      children: [
                        Text('Tu email no está verificado'),
                        SizedBox(height: 8),
                        ElevatedButton(
                          onPressed: () async {
                            await user.sendEmailVerification();
                            ScaffoldMessenger.of(context).showSnackBar(
                              SnackBar(content: Text('Email de verificación enviado')),
                            );
                          },
                          child: Text('Reenviar email'),
                        ),
                      ],
                    ),
                  ),
                ),
              ),
          ],
        ),
      ),
    );
  }
}
```

## Configurar Google Sign In en Firebase Console

```
Firebase Console → Authentication → Sign-in method
→ Google → Enable
→ Configurar pantalla de consentimiento OAuth
```

Para iOS, añadir URL Scheme:

```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>com.googleusercontent.apps.TU-CLIENT-ID-REVERSED</string>
        </array>
    </dict>
</array>
```

## Problemas comunes

### 1. Google Sign In falla en Android

**Solución**: Añadir SHA-1 fingerprint en Firebase Console:

```bash
# Debug
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey

# Release
keytool -list -v -keystore android/app/release.keystore -alias mykey
```

Copiar SHA-1 y añadir en Firebase Console → Project Settings → Your apps → Android app.

### 2. "PlatformException(sign_in_failed)"

Firebase Console → Authentication → Sign-in method → Google → Asegurar que está habilitado

### 3. Email verification no funciona

```dart
// Verificar antes de permitir acceso completo
if (user != null && !user.emailVerified) {
  // Mostrar mensaje o redirigir a pantalla de verificación
}
```

## Resultados

- Autenticación completa implementada en **6 horas**
- Email/password + Google Sign In funcionando
- Password reset automatizado
- Verificación de email
- 100% funcional en iOS y Android

Firebase Authentication es perfecto para MVPs y productos que no necesitan autenticación custom compleja.

¿Has usado Firebase Auth en Flutter? ¿Qué otros providers has integrado?
