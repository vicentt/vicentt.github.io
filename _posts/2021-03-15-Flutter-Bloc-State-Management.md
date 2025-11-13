---
layout: post
title: Flutter BLoC - Gestión de estado predecible y testeable
author: Vicente José Moreno Escobar
categories: [Flutter, Arquitectura, Mobile]
published: true
---

> Cuando setState() ya no es suficiente para tu app Flutter

# El problema

App Flutter creciendo:
- 30+ pantallas
- Estado compartido entre múltiples widgets
- Lógica de negocio mezclada con UI
- Testing imposible

**Solución**: BLoC pattern (Business Logic Component).

## ¿Por qué BLoC?

Comparativa de state management:

| setState | Provider | BLoC | Riverpod |
|----------|----------|------|----------|
| Simple | Moderado | Complejo | Moderado |
| Local | Global | Global | Global |
| Difícil test | OK | Excelente | Excelente |
| Boilerplate bajo | Medio | Alto | Medio |

BLoC brilla cuando:
- App compleja con mucho estado
- Testing es prioridad
- Equipo grande que necesita patrones claros

## Instalación

```yaml
# pubspec.yaml
dependencies:
  flutter_bloc: ^8.1.0
  equatable: ^2.0.5

dev_dependencies:
  bloc_test: ^9.1.0
  mocktail: ^0.3.0
```

## Conceptos básicos

```
Events → BLoC → States
  ↑               ↓
  UI ← ← ← ← ← ← UI
```

- **Events**: Acciones del usuario (botón clickeado, formulario enviado)
- **BLoC**: Procesa events, emite states
- **States**: Representaciones inmutables de UI

## Ejemplo: Counter (el "Hello World" de BLoC)

### 1. Define Events

```dart
// counter/counter_event.dart
abstract class CounterEvent extends Equatable {
  const CounterEvent();

  @override
  List<Object> get props => [];
}

class CounterIncremented extends CounterEvent {}

class CounterDecremented extends CounterEvent {}
```

### 2. Define States

```dart
// counter/counter_state.dart
class CounterState extends Equatable {
  final int count;

  const CounterState({this.count = 0});

  CounterState copyWith({int? count}) {
    return CounterState(count: count ?? this.count);
  }

  @override
  List<Object> get props => [count];
}
```

### 3. Implementa BLoC

```dart
// counter/counter_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';

class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(const CounterState()) {
    on<CounterIncremented>(_onIncremented);
    on<CounterDecremented>(_onDecremented);
  }

  void _onIncremented(CounterIncremented event, Emitter<CounterState> emit) {
    emit(state.copyWith(count: state.count + 1));
  }

  void _onDecremented(CounterDecremented event, Emitter<CounterState> emit) {
    emit(state.copyWith(count: state.count - 1));
  }
}
```

### 4. UI con BlocProvider y BlocBuilder

```dart
// main.dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: BlocProvider(
        create: (_) => CounterBloc(),
        child: CounterPage(),
      ),
    );
  }
}

class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Counter')),
      body: Center(
        child: BlocBuilder<CounterBloc, CounterState>(
          builder: (context, state) {
            return Text(
              '${state.count}',
              style: TextStyle(fontSize: 48),
            );
          },
        ),
      ),
      floatingActionButton: Column(
        mainAxisAlignment: MainAxisAlignment.end,
        children: [
          FloatingActionButton(
            onPressed: () => context.read<CounterBloc>().add(CounterIncremented()),
            child: Icon(Icons.add),
          ),
          SizedBox(height: 8),
          FloatingActionButton(
            onPressed: () => context.read<CounterBloc>().add(CounterDecremented()),
            child: Icon(Icons.remove),
          ),
        ],
      ),
    );
  }
}
```

## Caso real: Login con llamada API

### Events

```dart
// auth/auth_event.dart
abstract class AuthEvent extends Equatable {
  @override
  List<Object?> get props => [];
}

class AuthLoginRequested extends AuthEvent {
  final String email;
  final String password;

  AuthLoginRequested({required this.email, required this.password});

  @override
  List<Object> get props => [email, password];
}

class AuthLogoutRequested extends AuthEvent {}

class AuthCheckRequested extends AuthEvent {}
```

### States

```dart
// auth/auth_state.dart
enum AuthStatus { unknown, authenticated, unauthenticated }

class AuthState extends Equatable {
  final AuthStatus status;
  final User? user;
  final String? error;

  const AuthState({
    this.status = AuthStatus.unknown,
    this.user,
    this.error,
  });

  AuthState copyWith({
    AuthStatus? status,
    User? user,
    String? error,
  }) {
    return AuthState(
      status: status ?? this.status,
      user: user ?? this.user,
      error: error,
    );
  }

  @override
  List<Object?> get props => [status, user, error];
}
```

### BLoC con API calls

```dart
// auth/auth_bloc.dart
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final AuthRepository _authRepository;

  AuthBloc({required AuthRepository authRepository})
      : _authRepository = authRepository,
        super(const AuthState()) {
    on<AuthLoginRequested>(_onLoginRequested);
    on<AuthLogoutRequested>(_onLogoutRequested);
    on<AuthCheckRequested>(_onCheckRequested);
  }

  Future<void> _onLoginRequested(
    AuthLoginRequested event,
    Emitter<AuthState> emit,
  ) async {
    try {
      final user = await _authRepository.login(
        email: event.email,
        password: event.password,
      );

      emit(state.copyWith(
        status: AuthStatus.authenticated,
        user: user,
      ));
    } catch (e) {
      emit(state.copyWith(
        status: AuthStatus.unauthenticated,
        error: e.toString(),
      ));
    }
  }

  Future<void> _onLogoutRequested(
    AuthLogoutRequested event,
    Emitter<AuthState> emit,
  ) async {
    await _authRepository.logout();
    emit(const AuthState(status: AuthStatus.unauthenticated));
  }

  Future<void> _onCheckRequested(
    AuthCheckRequested event,
    Emitter<AuthState> emit,
  ) async {
    final user = await _authRepository.getCurrentUser();

    if (user != null) {
      emit(state.copyWith(
        status: AuthStatus.authenticated,
        user: user,
      ));
    } else {
      emit(state.copyWith(status: AuthStatus.unauthenticated));
    }
  }
}
```

### UI con loading/error states

```dart
// login_page.dart
class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Login')),
      body: BlocConsumer<AuthBloc, AuthState>(
        listener: (context, state) {
          if (state.status == AuthStatus.authenticated) {
            Navigator.of(context).pushReplacementNamed('/home');
          }

          if (state.error != null) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text(state.error!)),
            );
          }
        },
        builder: (context, state) {
          if (state.status == AuthStatus.unknown) {
            return Center(child: CircularProgressIndicator());
          }

          return LoginForm();
        },
      ),
    );
  }
}

class LoginForm extends StatefulWidget {
  @override
  _LoginFormState createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.all(16.0),
      child: Column(
        children: [
          TextField(
            controller: _emailController,
            decoration: InputDecoration(labelText: 'Email'),
          ),
          TextField(
            controller: _passwordController,
            decoration: InputDecoration(labelText: 'Password'),
            obscureText: true,
          ),
          SizedBox(height: 16),
          BlocBuilder<AuthBloc, AuthState>(
            builder: (context, state) {
              return ElevatedButton(
                onPressed: () {
                  context.read<AuthBloc>().add(
                        AuthLoginRequested(
                          email: _emailController.text,
                          password: _passwordController.text,
                        ),
                      );
                },
                child: Text('Login'),
              );
            },
          ),
        ],
      ),
    );
  }
}
```

## BlocObserver para debugging

```dart
// app/bloc_observer.dart
class AppBlocObserver extends BlocObserver {
  @override
  void onCreate(BlocBase bloc) {
    super.onCreate(bloc);
    print('onCreate -- ${bloc.runtimeType}');
  }

  @override
  void onChange(BlocBase bloc, Change change) {
    super.onChange(bloc, change);
    print('onChange -- ${bloc.runtimeType}, $change');
  }

  @override
  void onError(BlocBase bloc, Object error, StackTrace stackTrace) {
    print('onError -- ${bloc.runtimeType}, $error');
    super.onError(bloc, error, stackTrace);
  }

  @override
  void onClose(BlocBase bloc) {
    super.onClose(bloc);
    print('onClose -- ${bloc.runtimeType}');
  }
}

// main.dart
void main() {
  Bloc.observer = AppBlocObserver();
  runApp(MyApp());
}
```

## Testing BLoCs

```dart
// test/auth/auth_bloc_test.dart
import 'package:bloc_test/bloc_test.dart';
import 'package:mocktail/mocktail.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  group('AuthBloc', () {
    late AuthRepository authRepository;

    setUp(() {
      authRepository = MockAuthRepository();
    });

    test('initial state is unknown', () {
      final bloc = AuthBloc(authRepository: authRepository);
      expect(bloc.state.status, AuthStatus.unknown);
    });

    blocTest<AuthBloc, AuthState>(
      'emits authenticated when login succeeds',
      build: () {
        when(() => authRepository.login(
              email: any(named: 'email'),
              password: any(named: 'password'),
            )).thenAnswer((_) async => User(id: '1', email: 'test@test.com'));

        return AuthBloc(authRepository: authRepository);
      },
      act: (bloc) => bloc.add(
        AuthLoginRequested(email: 'test@test.com', password: 'password'),
      ),
      expect: () => [
        isA<AuthState>()
            .having((s) => s.status, 'status', AuthStatus.authenticated)
            .having((s) => s.user?.email, 'user email', 'test@test.com'),
      ],
    );

    blocTest<AuthBloc, AuthState>(
      'emits unauthenticated with error when login fails',
      build: () {
        when(() => authRepository.login(
              email: any(named: 'email'),
              password: any(named: 'password'),
            )).thenThrow(Exception('Invalid credentials'));

        return AuthBloc(authRepository: authRepository);
      },
      act: (bloc) => bloc.add(
        AuthLoginRequested(email: 'test@test.com', password: 'wrong'),
      ),
      expect: () => [
        isA<AuthState>()
            .having((s) => s.status, 'status', AuthStatus.unauthenticated)
            .having((s) => s.error, 'error', contains('Invalid credentials')),
      ],
    );
  });
}
```

## Múltiples BLoCs

```dart
// App con múltiples BLoCs
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider<AuthBloc>(
          create: (context) => AuthBloc(
            authRepository: context.read<AuthRepository>(),
          )..add(AuthCheckRequested()),
        ),
        BlocProvider<SettingsBloc>(
          create: (context) => SettingsBloc(),
        ),
        BlocProvider<ThemeBloc>(
          create: (context) => ThemeBloc(),
        ),
      ],
      child: MaterialApp(
        home: HomePage(),
      ),
    );
  }
}
```

## BLoC vs Cubit

Cubit es versión simplificada de BLoC sin events:

```dart
// Simple Cubit
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
}

// Uso
context.read<CounterCubit>().increment();
```

**Cuándo usar Cubit**:
- Estado simple
- No necesitas auditoría de events
- Menos boilerplate

**Cuándo usar BLoC completo**:
- Estado complejo
- Necesitas replay/debugging de events
- Testing exhaustivo

## Optimizaciones

### 1. buildWhen para evitar rebuilds innecesarios

```dart
BlocBuilder<AuthBloc, AuthState>(
  buildWhen: (previous, current) {
    // Solo rebuild si status cambió
    return previous.status != current.status;
  },
  builder: (context, state) {
    // ...
  },
)
```

### 2. BlocSelector para subscribirse a parte del state

```dart
BlocSelector<AuthBloc, AuthState, String?>(
  selector: (state) => state.user?.name,
  builder: (context, userName) {
    return Text(userName ?? 'Guest');
  },
)
```

## Mejores prácticas

1. **Un BLoC por feature**: `LoginBloc`, `ProfileBloc`, no `AppBloc` gigante
2. **States inmutables**: Usar `copyWith()` siempre
3. **Equatable para comparaciones**: Evita rebuilds innecesarios
4. **Repository pattern**: BLoC no llama APIs directamente
5. **Testing exhaustivo**: BLoCs son fáciles de testear, aprovecharlo

## Resultados

Con BLoC implementado:
- Lógica separada de UI completamente
- Testing coverage 95%+
- Debugging más fácil con BlocObserver
- Onboarding de nuevos devs más rápido (patrón claro)

¿Usas BLoC en Flutter? ¿O prefieres Riverpod/Provider?
