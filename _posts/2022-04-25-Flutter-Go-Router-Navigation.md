---
layout: post
title: Flutter go_router - Navegación declarativa y deep linking
author: Vicente José Moreno Escobar
categories: [Flutter, Mobile, Arquitectura]
published: true
---

> Cuando Navigator 1.0 se queda corto y necesitas URLs en tu app móvil

# El problema

App Flutter con navegación compleja:
- Deep links desde web/email
- Navegación condicional (auth required)
- Nested navigation (tabs con sus propios stacks)
- State restoration

Navigator 1.0/2.0 era código imperativo difícil de mantener.

**Solución**: **go_router** - navegación declarativa con URLs.

## Navigator 1.0 (el viejo modo)

```dart
// Imperativo, difícil de mantener
Navigator.push(
  context,
  MaterialPageRoute(builder: (context) => ProductDetail(id: '123'))
);

// Deep linking: manual y complicado
```

## go_router (el nuevo modo)

```dart
// Declarativo, basado en URLs
context.go('/product/123');

// Deep linking: automático
```

## Instalación

```yaml
# pubspec.yaml
dependencies:
  go_router: ^13.0.0
```

## Configuración básica

```dart
// lib/router/app_router.dart
import 'package:go_router/go_router.dart';

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(),
    ),
    GoRoute(
      path: '/product/:id',
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        return ProductDetailScreen(productId: id);
      },
    ),
  ],
);

// main.dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: router,
    );
  }
}
```

## Navegación básica

```dart
// Ir a ruta
context.go('/product/123');

// Ir y reemplazar (no volver atrás)
context.go('/login');
context.pushReplacement('/home');

// Push (mantener en stack)
context.push('/settings');

// Pop (volver)
context.pop();

// Ir a named route con params
context.goNamed(
  'product',
  pathParameters: {'id': '123'},
  queryParameters: {'ref': 'email'},
);
```

## Named routes

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      name: 'home',
      path: '/',
      builder: (context, state) => HomeScreen(),
    ),
    GoRoute(
      name: 'product',
      path: '/product/:id',
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        return ProductDetailScreen(productId: id);
      },
    ),
  ],
);

// Uso
context.goNamed('product', pathParameters: {'id': '123'});
```

## Query parameters

```dart
GoRoute(
  path: '/search',
  builder: (context, state) {
    final query = state.uri.queryParameters['q'] ?? '';
    final category = state.uri.queryParameters['category'];

    return SearchScreen(
      query: query,
      category: category,
    );
  },
)

// Navegación
context.go('/search?q=laptop&category=electronics');
```

## Nested navigation (tabs)

```dart
final router = GoRouter(
  routes: [
    ShellRoute(
      builder: (context, state, child) {
        return ScaffoldWithNavBar(child: child);
      },
      routes: [
        GoRoute(
          path: '/home',
          builder: (context, state) => HomeScreen(),
        ),
        GoRoute(
          path: '/favorites',
          builder: (context, state) => FavoritesScreen(),
        ),
        GoRoute(
          path: '/profile',
          builder: (context, state) => ProfileScreen(),
        ),
      ],
    ),
  ],
);

// ScaffoldWithNavBar mantiene bottom navigation
class ScaffoldWithNavBar extends StatelessWidget {
  final Widget child;

  const ScaffoldWithNavBar({required this.child});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child,
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _calculateSelectedIndex(context),
        onTap: (index) => _onItemTapped(index, context),
        items: [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
          BottomNavigationBarItem(icon: Icon(Icons.favorite), label: 'Favorites'),
          BottomNavigationBarItem(icon: Icon(Icons.person), label: 'Profile'),
        ],
      ),
    );
  }

  int _calculateSelectedIndex(BuildContext context) {
    final location = GoRouterState.of(context).uri.toString();

    if (location.startsWith('/home')) return 0;
    if (location.startsWith('/favorites')) return 1;
    if (location.startsWith('/profile')) return 2;

    return 0;
  }

  void _onItemTapped(int index, BuildContext context) {
    switch (index) {
      case 0:
        context.go('/home');
        break;
      case 1:
        context.go('/favorites');
        break;
      case 2:
        context.go('/profile');
        break;
    }
  }
}
```

## Redirect (auth guard)

```dart
final router = GoRouter(
  redirect: (context, state) {
    final authState = context.read<AuthBloc>().state;
    final isLoggedIn = authState.status == AuthStatus.authenticated;

    final isGoingToLogin = state.matchedLocation == '/login';

    // Si no está logueado y no va a login → redirigir a login
    if (!isLoggedIn && !isGoingToLogin) {
      return '/login';
    }

    // Si está logueado y va a login → redirigir a home
    if (isLoggedIn && isGoingToLogin) {
      return '/';
    }

    // No redirigir
    return null;
  },
  routes: [
    GoRoute(
      path: '/login',
      builder: (context, state) => LoginScreen(),
    ),
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(),
    ),
    GoRoute(
      path: '/profile',
      builder: (context, state) => ProfileScreen(),
    ),
  ],
);
```

## Refresh listenable (reactive redirects)

```dart
final router = GoRouter(
  refreshListenable: GoRouterRefreshStream(
    context.read<AuthBloc>().stream,
  ),
  redirect: (context, state) {
    // Este redirect se re-evalúa cuando AuthBloc emite nuevo state
    final authState = context.read<AuthBloc>().state;
    final isLoggedIn = authState.status == AuthStatus.authenticated;

    if (!isLoggedIn && state.matchedLocation != '/login') {
      return '/login';
    }

    return null;
  },
  routes: [...],
);

// Helper class
class GoRouterRefreshStream extends ChangeNotifier {
  GoRouterRefreshStream(Stream stream) {
    notifyListeners();
    _subscription = stream.asBroadcastStream().listen((_) {
      notifyListeners();
    });
  }

  late final StreamSubscription _subscription;

  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
}
```

## Error handling

```dart
final router = GoRouter(
  errorBuilder: (context, state) => ErrorScreen(error: state.error),
  routes: [
    GoRoute(
      path: '/product/:id',
      builder: (context, state) {
        final id = state.pathParameters['id'];

        if (id == null || id.isEmpty) {
          throw Exception('Product ID is required');
        }

        return ProductDetailScreen(productId: id);
      },
    ),
  ],
);

class ErrorScreen extends StatelessWidget {
  final Exception? error;

  const ErrorScreen({this.error});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Error')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.error, size: 100, color: Colors.red),
            SizedBox(height: 16),
            Text(error?.toString() ?? 'Unknown error'),
            SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => context.go('/'),
              child: Text('Go Home'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Deep linking

### Android

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />

    <data
        android:scheme="https"
        android:host="tusitio.com" />
</intent-filter>
```

### iOS

```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>https</string>
        </array>
    </dict>
</array>

<key>FlutterDeepLinkingEnabled</key>
<true/>
```

### Handling deep links

```dart
// go_router maneja automáticamente!
// Si usuario clickea https://tusitio.com/product/123
// → Abre app en /product/123

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/product/:id',
      builder: (context, state) {
        final id = state.pathParameters['id']!;

        // Log analytics
        analytics.logEvent(
          name: 'deep_link_opened',
          parameters: {'product_id': id},
        );

        return ProductDetailScreen(productId: id);
      },
    ),
  ],
);
```

## State restoration

```dart
final router = GoRouter(
  // Restaurar navegación después de reinicio
  restorationScopeId: 'app',
  routes: [...],
);

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: router,
      restorationScopeId: 'app',
    );
  }
}
```

## Transiciones custom

```dart
GoRoute(
  path: '/product/:id',
  pageBuilder: (context, state) {
    final id = state.pathParameters['id']!;

    return CustomTransitionPage(
      key: state.pageKey,
      child: ProductDetailScreen(productId: id),
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        return FadeTransition(
          opacity: animation,
          child: child,
        );
      },
    );
  },
)
```

## Passing complex objects

```dart
// NO hacer esto (no funciona con deep links):
// context.go('/product', extra: Product(...));

// HACER esto:
// 1. Pasar ID y fetch en destino
context.go('/product/123');

// En ProductDetailScreen
class ProductDetailScreen extends StatefulWidget {
  final String productId;

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<Product>(
      future: fetchProduct(productId),
      builder: (context, snapshot) {
        if (!snapshot.hasData) return CircularProgressIndicator();
        return ProductView(product: snapshot.data!);
      },
    );
  }
}

// O usar provider/state management
context.read<ProductBloc>().add(LoadProduct(productId));
```

## Testing

```dart
// test/router_test.dart
void main() {
  testWidgets('Navigate to product detail', (tester) async {
    final router = GoRouter(
      routes: [
        GoRoute(
          path: '/',
          builder: (context, state) => HomeScreen(),
        ),
        GoRoute(
          path: '/product/:id',
          builder: (context, state) {
            final id = state.pathParameters['id']!;
            return ProductDetailScreen(productId: id);
          },
        ),
      ],
    );

    await tester.pumpWidget(MaterialApp.router(routerConfig: router));

    // Navigate
    router.go('/product/123');
    await tester.pumpAndSettle();

    // Verify
    expect(find.byType(ProductDetailScreen), findsOneWidget);
  });

  test('Redirect if not authenticated', () {
    final router = GoRouter(
      redirect: (context, state) {
        if (state.matchedLocation != '/login') {
          return '/login';
        }
        return null;
      },
      routes: [
        GoRoute(path: '/', builder: (c, s) => HomeScreen()),
        GoRoute(path: '/login', builder: (c, s) => LoginScreen()),
      ],
    );

    expect(router.routeInformationProvider.value.uri.path, '/login');
  });
}
```

## Debugging

```dart
final router = GoRouter(
  debugLogDiagnostics: true, // Print routing info
  routes: [...],
);

// Logs:
// [GoRouter] known full paths for routes:
// [GoRouter]   => /
// [GoRouter]   => /product/:id
// [GoRouter]   => /search
```

## Best practices

1. **Usa named routes** para refactorings fáciles
2. **Evita pasar objetos complejos** en extra
3. **Implementa redirects** para auth
4. **Test deep links** en device real
5. **Usa ShellRoute** para nested navigation
6. **Monitor analytics** en deep links

## Migración desde Navigator 1.0

```dart
// Antes
Navigator.of(context).push(
  MaterialPageRoute(builder: (context) => ProductDetailScreen(id: '123'))
);

// Después
context.push('/product/123');

// Antes
Navigator.of(context).pushNamed('/product', arguments: '123');

// Después
context.goNamed('product', pathParameters: {'id': '123'});
```

## Resultados

Con go_router implementado:

- Deep links funcionando en iOS y Android
- Navegación condicional (auth) centralizada
- URLs compartibles desde la app
- State restoration automático
- Código de navegación 60% más limpio

¿Usas go_router en Flutter? ¿O prefieres otro paquete de routing?
