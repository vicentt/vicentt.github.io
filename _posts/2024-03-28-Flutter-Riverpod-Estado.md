---
layout: post
title: Flutter y Riverpod - Gestión de estado que realmente funciona
image: /images/codeMan.jpg
author: Vicente José Moreno Escobar
categories: [Flutter, Mobile, State Management]
published: true
---
![Flutter Riverpod](/images/codeMan.jpg)

> Después de probar Provider, Bloc y GetX, Riverpod es mi elección definitiva para Flutter

# El problema de la gestión de estado en Flutter 🤯

Flutter tiene un problema: demasiadas opciones para gestión de estado. Provider, Bloc, GetX, MobX, Redux... La lista es interminable.

He usado varios en producción. Riverpod es el que menos problemas me ha dado.

## ¿Por qué Riverpod?

Riverpod es una reescritura completa de Provider que soluciona sus principales problemas:

1. **Type-safe** - El compilador detecta errores
2. **No depende de BuildContext** - Más testeable
3. **Composición de providers** - Puedes combinarlos fácilmente
4. **Scope automático** - No más problemas de scope

## Setup inicial 📦

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.4.10

dev_dependencies:
  riverpod_generator: ^2.3.9
  build_runner: ^2.4.8
```

Wrappea tu app:

```dart
void main() {
  runApp(
    ProviderScope(
      child: MyApp(),
    ),
  );
}
```

## Ejemplos prácticos 💡

### 1. Provider simple

```dart
// Estado simple (contador)
final counterProvider = StateProvider<int>((ref) => 0);

// Uso en widget
class CounterWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);

    return Column(
      children: [
        Text('Count: $count'),
        ElevatedButton(
          onPressed: () => ref.read(counterProvider.notifier).state++,
          child: Text('Increment'),
        ),
      ],
    );
  }
}
```

### 2. Provider asíncrono (API call)

```dart
// Modelo
class User {
  final String id;
  final String name;
  final String email;

  User({required this.id, required this.name, required this.email});

  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['id'],
    name: json['name'],
    email: json['email'],
  );
}

// Provider
final userProvider = FutureProvider.family<User, String>((ref, userId) async {
  final response = await http.get(
    Uri.parse('https://api.example.com/users/$userId'),
  );

  if (response.statusCode == 200) {
    return User.fromJson(jsonDecode(response.body));
  }

  throw Exception('Failed to load user');
});

// Uso en widget
class UserProfile extends ConsumerWidget {
  final String userId;

  const UserProfile({required this.userId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProvider(userId));

    return userAsync.when(
      data: (user) => Column(
        children: [
          Text(user.name),
          Text(user.email),
        ],
      ),
      loading: () => CircularProgressIndicator(),
      error: (err, stack) => Text('Error: $err'),
    );
  }
}
```

El método `.when()` es brillante. Maneja todos los estados posibles sin boilerplate.

### 3. StateNotifier para lógica compleja

```dart
// Estado
class TodosState {
  final List<Todo> todos;
  final bool isLoading;
  final String? error;

  TodosState({
    this.todos = const [],
    this.isLoading = false,
    this.error,
  });

  TodosState copyWith({
    List<Todo>? todos,
    bool? isLoading,
    String? error,
  }) {
    return TodosState(
      todos: todos ?? this.todos,
      isLoading: isLoading ?? this.isLoading,
      error: error,
    );
  }
}

// Notifier
class TodosNotifier extends StateNotifier<TodosState> {
  TodosNotifier(this.ref) : super(TodosState()) {
    loadTodos();
  }

  final Ref ref;

  Future<void> loadTodos() async {
    state = state.copyWith(isLoading: true, error: null);

    try {
      final todos = await ref.read(todoRepositoryProvider).getAll();
      state = state.copyWith(todos: todos, isLoading: false);
    } catch (e) {
      state = state.copyWith(error: e.toString(), isLoading: false);
    }
  }

  Future<void> addTodo(String title) async {
    final newTodo = await ref.read(todoRepositoryProvider).create(title);
    state = state.copyWith(
      todos: [...state.todos, newTodo],
    );
  }

  void toggleTodo(String id) {
    state = state.copyWith(
      todos: [
        for (final todo in state.todos)
          if (todo.id == id)
            todo.copyWith(completed: !todo.completed)
          else
            todo,
      ],
    );
  }
}

// Provider
final todosProvider = StateNotifierProvider<TodosNotifier, TodosState>((ref) {
  return TodosNotifier(ref);
});
```

### 4. Composición de providers

Aquí es donde Riverpod brilla:

```dart
// Provider de autenticación
final authProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) {
  return AuthNotifier();
});

// Provider que depende de auth
final userTodosProvider = FutureProvider<List<Todo>>((ref) async {
  // Lee el estado de auth
  final auth = ref.watch(authProvider);

  if (auth.user == null) {
    return [];
  }

  // Si hay usuario, carga sus todos
  return ref.read(todoRepositoryProvider).getTodosForUser(auth.user!.id);
});

// Provider que filtra todos completados
final completedTodosProvider = Provider<List<Todo>>((ref) {
  final todos = ref.watch(userTodosProvider).value ?? [];
  return todos.where((todo) => todo.completed).toList();
});

// Provider que cuenta todos pendientes
final pendingTodosCountProvider = Provider<int>((ref) {
  final todos = ref.watch(userTodosProvider).value ?? [];
  return todos.where((todo) => !todo.completed).length;
});
```

Cuando `authProvider` cambia, todo el árbol de dependencias se recalcula automáticamente. Magia.

## Code generation con riverpod_generator 🪄

Para proyectos grandes, usa code generation:

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'todos.g.dart';

@riverpod
class Todos extends _$Todos {
  @override
  FutureOr<List<Todo>> build() async {
    return await ref.read(todoRepositoryProvider).getAll();
  }

  Future<void> addTodo(String title) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final newTodo = await ref.read(todoRepositoryProvider).create(title);
      final todos = await future;
      return [...todos, newTodo];
    });
  }
}
```

Genera el provider automáticamente:

```bash
dart run build_runner watch
```

## Testing 🧪

Lo mejor de Riverpod es lo testeable que es:

```dart
void main() {
  test('counter increments', () {
    final container = ProviderContainer();

    expect(container.read(counterProvider), 0);

    container.read(counterProvider.notifier).state++;

    expect(container.read(counterProvider), 1);

    container.dispose();
  });

  test('user loading', () async {
    final container = ProviderContainer(
      overrides: [
        // Mock del repository
        userRepositoryProvider.overrideWithValue(
          MockUserRepository(),
        ),
      ],
    );

    final user = await container.read(userProvider('123').future);

    expect(user.name, 'Test User');

    container.dispose();
  });
}
```

No necesitas BuildContext, no necesitas widgets, solo lógica pura.

## Riverpod vs otras soluciones 📊

| Feature | Riverpod | Provider | Bloc | GetX |
|---------|----------|----------|------|------|
| Type safety | ✅ | ⚠️ | ✅ | ❌ |
| Testeable | ✅ | ⚠️ | ✅ | ⚠️ |
| Boilerplate | 🟡 | 🟢 | 🔴 | 🟢 |
| Composición | ✅ | ⚠️ | ❌ | ⚠️ |
| Documentación | ✅ | ✅ | ✅ | ⚠️ |
| Code gen | ✅ | ❌ | ✅ | ❌ |

## Mi setup recomendado 🎯

Para proyectos nuevos, esta es mi estructura:

```
lib/
  ├── features/
  │   ├── auth/
  │   │   ├── data/
  │   │   │   ├── repositories/
  │   │   │   └── providers.dart
  │   │   ├── domain/
  │   │   │   └── models/
  │   │   └── presentation/
  │   │       ├── screens/
  │   │       └── providers.dart
  │   └── todos/
  │       └── ...
  └── shared/
      ├── providers/
      └── utils/
```

Cada feature tiene sus providers, repositories están inyectados via Riverpod.

## Recursos útiles 📚

- [Documentación oficial](https://riverpod.dev/)
- [Cookbook con ejemplos](https://riverpod.dev/docs/cookbooks/authentication)
- [riverpod_lint](https://pub.dev/packages/riverpod_lint) - Linter rules útiles

## Conclusión

Riverpod no es perfecto, tiene una curva de aprendizaje. Pero una vez lo entiendes, es difícil volver atrás.

Si estás empezando un proyecto Flutter en 2024, dale una oportunidad a Riverpod. Tu yo del futuro te lo agradecerá.

¿Qué solución de estado usas en Flutter? ¿Has probado Riverpod? Me encantaría conocer tu experiencia.

¡Happy fluttering! 🦋
