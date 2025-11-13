---
layout: post
title: React Native Offline-First - Apps que funcionan sin conexión
author: Vicente José Moreno Escobar
categories: [React Native, Mobile, Performance]
published: true
---

> Cuando tu app móvil necesita funcionar en el metro sin WiFi

# El problema

App de gestión de tareas para trabajadores de campo. El problema:
- Trabajan en zonas sin cobertura
- Necesitan registrar datos sobre la marcha
- Sincronizar cuando vuelven a tener conexión

**Solución**: Arquitectura offline-first.

## Offline-First vs Online-First

**Online-First** (tradicional):
```
Usuario hace acción → API request → Actualizar UI
Si no hay internet → Error
```

**Offline-First**:
```
Usuario hace acción → Actualizar local storage → Actualizar UI
Sincronizar con API cuando haya conexión
```

## Stack tecnológico

```json
{
  "dependencies": {
    "react-native": "0.68.0",
    "@react-native-async-storage/async-storage": "^1.17.0",
    "@reduxjs/toolkit": "^1.8.0",
    "redux-persist": "^6.0.0",
    "@react-native-community/netinfo": "^9.0.0",
    "react-native-queue": "^1.0.0"
  }
}
```

## Detección de conectividad

```typescript
// services/NetworkService.ts
import NetInfo from '@react-native-community/netinfo';

class NetworkService {
  private isConnected: boolean = true;
  private listeners: Array<(connected: boolean) => void> = [];

  constructor() {
    this.init();
  }

  private init() {
    NetInfo.addEventListener(state => {
      const wasConnected = this.isConnected;
      this.isConnected = state.isConnected ?? false;

      // Si cambió el estado, notificar listeners
      if (wasConnected !== this.isConnected) {
        this.notifyListeners();

        if (this.isConnected) {
          console.log('🟢 Conexión restaurada, iniciando sincronización...');
          // Trigger sync
          SyncService.syncAll();
        } else {
          console.log('🔴 Conexión perdida, modo offline');
        }
      }
    });
  }

  isOnline(): boolean {
    return this.isConnected;
  }

  onConnectionChange(callback: (connected: boolean) => void) {
    this.listeners.push(callback);
  }

  private notifyListeners() {
    this.listeners.forEach(listener => listener(this.isConnected));
  }
}

export default new NetworkService();
```

## Redux con persistencia

```typescript
// store/store.ts
import { configureStore } from '@reduxjs/toolkit';
import {
  persistStore,
  persistReducer,
  FLUSH,
  REHYDRATE,
  PAUSE,
  PERSIST,
  PURGE,
  REGISTER,
} from 'redux-persist';
import AsyncStorage from '@react-native-async-storage/async-storage';
import tasksReducer from './slices/tasksSlice';
import syncReducer from './slices/syncSlice';

const persistConfig = {
  key: 'root',
  storage: AsyncStorage,
  whitelist: ['tasks', 'sync'], // Solo persistir estos reducers
};

const persistedReducer = persistReducer(persistConfig, tasksReducer);

export const store = configureStore({
  reducer: {
    tasks: persistedReducer,
    sync: syncReducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
      },
    }),
});

export const persistor = persistStore(store);
```

## Tasks Slice con optimistic updates

```typescript
// store/slices/tasksSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { v4 as uuidv4 } from 'react-native-uuid';

interface Task {
  id: string;
  title: string;
  completed: boolean;
  localOnly?: boolean; // Indica que no se ha sincronizado
  updatedAt: number;
}

interface TasksState {
  items: Task[];
  pendingSync: string[]; // IDs de tasks que necesitan sincronizar
}

const initialState: TasksState = {
  items: [],
  pendingSync: [],
};

const tasksSlice = createSlice({
  name: 'tasks',
  initialState,
  reducers: {
    addTask: (state, action: PayloadAction<Omit<Task, 'id' | 'localOnly'>>) => {
      const task: Task = {
        ...action.payload,
        id: uuidv4(),
        localOnly: true,
        updatedAt: Date.now(),
      };

      state.items.push(task);
      state.pendingSync.push(task.id);
    },

    updateTask: (state, action: PayloadAction<{ id: string; updates: Partial<Task> }>) => {
      const index = state.items.findIndex(t => t.id === action.payload.id);

      if (index !== -1) {
        state.items[index] = {
          ...state.items[index],
          ...action.payload.updates,
          localOnly: true,
          updatedAt: Date.now(),
        };

        // Añadir a pendingSync si no está ya
        if (!state.pendingSync.includes(action.payload.id)) {
          state.pendingSync.push(action.payload.id);
        }
      }
    },

    deleteTask: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter(t => t.id !== action.payload);
      state.pendingSync = state.pendingSync.filter(id => id !== action.payload);
    },

    markTaskSynced: (state, action: PayloadAction<string>) => {
      const task = state.items.find(t => t.id === action.payload);
      if (task) {
        task.localOnly = false;
      }
      state.pendingSync = state.pendingSync.filter(id => id !== action.payload);
    },

    // Usado cuando recibimos tasks del servidor
    setTasks: (state, action: PayloadAction<Task[]>) => {
      state.items = action.payload;
      state.pendingSync = [];
    },
  },
});

export const { addTask, updateTask, deleteTask, markTaskSynced, setTasks } = tasksSlice.actions;
export default tasksSlice.reducer;
```

## Servicio de sincronización

```typescript
// services/SyncService.ts
import { store } from '../store/store';
import { markTaskSynced, setTasks } from '../store/slices/tasksSlice';
import NetworkService from './NetworkService';
import api from './api';

class SyncService {
  private isSyncing: boolean = false;
  private syncQueue: string[] = [];

  async syncAll() {
    if (!NetworkService.isOnline()) {
      console.log('⏸️ Offline, sync postponed');
      return;
    }

    if (this.isSyncing) {
      console.log('⏸️ Sync already in progress');
      return;
    }

    this.isSyncing = true;
    console.log('🔄 Starting sync...');

    try {
      // 1. Pull: Obtener tasks del servidor
      await this.pullTasks();

      // 2. Push: Enviar cambios locales al servidor
      await this.pushTasks();

      console.log('✅ Sync completed');
    } catch (error) {
      console.error('❌ Sync failed:', error);
    } finally {
      this.isSyncing = false;
    }
  }

  private async pullTasks() {
    try {
      const response = await api.get('/tasks');
      const serverTasks = response.data;

      const state = store.getState();
      const localTasks = state.tasks.items;

      // Merge: Resolver conflictos
      const merged = this.mergeTasks(localTasks, serverTasks);

      store.dispatch(setTasks(merged));
    } catch (error) {
      console.error('Pull failed:', error);
      throw error;
    }
  }

  private async pushTasks() {
    const state = store.getState();
    const pendingIds = state.tasks.pendingSync;

    for (const taskId of pendingIds) {
      const task = state.tasks.items.find(t => t.id === taskId);

      if (!task) continue;

      try {
        if (task.localOnly) {
          // Crear en servidor
          await api.post('/tasks', task);
        } else {
          // Actualizar en servidor
          await api.put(`/tasks/${taskId}`, task);
        }

        store.dispatch(markTaskSynced(taskId));
      } catch (error) {
        console.error(`Failed to sync task ${taskId}:`, error);
        // Continuar con las demás
      }
    }
  }

  private mergeTasks(local: Task[], server: Task[]): Task[] {
    const merged = new Map<string, Task>();

    // Añadir tasks del servidor
    server.forEach(task => merged.set(task.id, task));

    // Merge con tasks locales (last-write-wins)
    local.forEach(localTask => {
      const serverTask = merged.get(localTask.id);

      if (!serverTask) {
        // Solo existe local, mantenerlo
        merged.set(localTask.id, localTask);
      } else if (localTask.updatedAt > serverTask.updatedAt) {
        // Local más reciente, usar local
        merged.set(localTask.id, localTask);
      }
      // Si server más reciente, ya está en el map
    });

    return Array.from(merged.values());
  }
}

export default new SyncService();
```

## API client con retry y queue

```typescript
// services/api.ts
import axios from 'axios';
import NetworkService from './NetworkService';

const api = axios.create({
  baseURL: 'https://api.ejemplo.com',
  timeout: 10000,
});

// Queue para requests fallidos
const requestQueue: Array<() => Promise<any>> = [];

api.interceptors.request.use(
  async config => {
    // Verificar conexión antes de cada request
    if (!NetworkService.isOnline()) {
      // Si offline, poner en queue
      throw new Error('No internet connection');
    }

    return config;
  },
  error => Promise.reject(error)
);

api.interceptors.response.use(
  response => response,
  async error => {
    if (error.message === 'No internet connection') {
      // Guardar para retry cuando haya conexión
      console.log('📥 Request queued for later');
      return Promise.reject(error);
    }

    // Retry en errores de red
    if (error.code === 'ECONNABORTED' || !error.response) {
      const config = error.config;

      if (!config._retryCount) {
        config._retryCount = 0;
      }

      if (config._retryCount < 3) {
        config._retryCount += 1;
        console.log(`🔄 Retry ${config._retryCount}/3`);

        // Exponential backoff
        const delay = Math.pow(2, config._retryCount) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));

        return api.request(config);
      }
    }

    return Promise.reject(error);
  }
);

export default api;
```

## UI: Indicador de sync

```typescript
// components/SyncIndicator.tsx
import React, { useState, useEffect } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { useSelector } from 'react-redux';
import NetworkService from '../services/NetworkService';

export const SyncIndicator: React.FC = () => {
  const [isOnline, setIsOnline] = useState(NetworkService.isOnline());
  const pendingCount = useSelector(state => state.tasks.pendingSync.length);

  useEffect(() => {
    NetworkService.onConnectionChange(setIsOnline);
  }, []);

  if (isOnline && pendingCount === 0) {
    return null; // Todo sincronizado
  }

  return (
    <View style={[styles.container, isOnline ? styles.syncing : styles.offline]}>
      <Text style={styles.text}>
        {isOnline
          ? `⏳ Sincronizando ${pendingCount} cambios...`
          : `📴 Offline - ${pendingCount} cambios pendientes`}
      </Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    padding: 8,
    alignItems: 'center',
  },
  offline: {
    backgroundColor: '#ff6b6b',
  },
  syncing: {
    backgroundColor: '#ffa500',
  },
  text: {
    color: '#fff',
    fontSize: 12,
  },
});
```

## Componente de lista optimista

```typescript
// screens/TasksScreen.tsx
import React from 'react';
import { FlatList, View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useSelector, useDispatch } from 'react-redux';
import { addTask, updateTask, deleteTask } from '../store/slices/tasksSlice';
import { SyncIndicator } from '../components/SyncIndicator';

export const TasksScreen: React.FC = () => {
  const tasks = useSelector(state => state.tasks.items);
  const dispatch = useDispatch();

  const handleAddTask = () => {
    dispatch(addTask({
      title: 'Nueva tarea',
      completed: false,
      updatedAt: Date.now(),
    }));
  };

  const handleToggleTask = (id: string, completed: boolean) => {
    dispatch(updateTask({
      id,
      updates: { completed: !completed },
    }));
  };

  return (
    <View style={styles.container}>
      <SyncIndicator />

      <FlatList
        data={tasks}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={styles.task}
            onPress={() => handleToggleTask(item.id, item.completed)}
          >
            <Text style={item.completed ? styles.completed : styles.pending}>
              {item.title}
            </Text>
            {item.localOnly && (
              <Text style={styles.badge}>⏳ No sincronizado</Text>
            )}
          </TouchableOpacity>
        )}
      />

      <TouchableOpacity style={styles.addButton} onPress={handleAddTask}>
        <Text style={styles.addButtonText}>+ Nueva Tarea</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  task: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  pending: { fontSize: 16 },
  completed: {
    fontSize: 16,
    textDecorationLine: 'line-through',
    color: '#999',
  },
  badge: {
    fontSize: 10,
    color: '#ffa500',
  },
  addButton: {
    backgroundColor: '#007AFF',
    padding: 16,
    margin: 16,
    borderRadius: 8,
    alignItems: 'center',
  },
  addButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: 'bold',
  },
});
```

## Sincronización automática

```typescript
// App.tsx
import React, { useEffect } from 'react';
import { Provider } from 'react-redux';
import { PersistGate } from 'redux-persist/integration/react';
import { store, persistor } from './store/store';
import NetworkService from './services/NetworkService';
import SyncService from './services/SyncService';
import { TasksScreen } from './screens/TasksScreen';

export default function App() {
  useEffect(() => {
    // Sync cuando la app se abre
    SyncService.syncAll();

    // Sync cuando recupera conexión
    NetworkService.onConnectionChange(isOnline => {
      if (isOnline) {
        SyncService.syncAll();
      }
    });

    // Sync periódico cada 5 minutos (si hay conexión)
    const interval = setInterval(() => {
      if (NetworkService.isOnline()) {
        SyncService.syncAll();
      }
    }, 5 * 60 * 1000);

    return () => clearInterval(interval);
  }, []);

  return (
    <Provider store={store}>
      <PersistGate loading={null} persistor={persistor}>
        <TasksScreen />
      </PersistGate>
    </Provider>
  );
}
```

## Manejo de conflictos

Para casos complejos, usar timestamps o CRDTs:

```typescript
// Conflict resolution: Last-Write-Wins
function resolveConflict(local: Task, server: Task): Task {
  if (local.updatedAt > server.updatedAt) {
    return local; // Local más reciente
  }
  return server; // Server más reciente
}

// O más sofisticado: Merge de campos individuales
function mergeFields(local: Task, server: Task): Task {
  return {
    id: local.id,
    title: local.updatedAt > server.updatedAt ? local.title : server.title,
    completed: local.updatedAt > server.updatedAt ? local.completed : server.completed,
    updatedAt: Math.max(local.updatedAt, server.updatedAt),
  };
}
```

## Resultados

- App funciona perfectamente offline
- Sincronización automática transparente
- UX fluida con optimistic updates
- 0 datos perdidos
- Trabajadores felices (pueden trabajar sin cobertura)

¿Has implementado apps offline-first? ¿Qué estrategias de sync usas?
