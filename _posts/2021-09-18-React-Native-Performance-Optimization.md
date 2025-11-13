---
layout: post
title: React Native Performance - De lag insoportable a 60 FPS
author: Vicente José Moreno Escobar
categories: [React Native, Performance, Mobile]
published: true
---

> Cuando tu app React Native se siente lenta y los usuarios se quejan

# El problema

App con 50+ pantallas. Usuarios reportaban:
- Lag al scrollear listas
- Animaciones tartamudeando
- Tiempo de carga lento
- Crashes por memoria

Profiler mostraba: **15-20 FPS** en pantallas críticas. Inaceptable.

## Medición inicial

```typescript
// Habilitar performance monitor
// Android: shake device → Show Perf Monitor
// iOS: Cmd+D → Show Perf Monitor

// React DevTools Profiler
import { Profiler } from 'react';

<Profiler id="ProductList" onRender={onRenderCallback}>
  <ProductList />
</Profiler>

function onRenderCallback(
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime,
) {
  console.log(`${id} took ${actualDuration}ms to render`);
}
```

## Optimización 1: FlatList virtualization

### Antes (render todo)

```typescript
// MAL: Renderiza 1000 items
const ProductList = ({ products }) => {
  return (
    <ScrollView>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </ScrollView>
  );
};
```

### Después (virtualizado)

```typescript
// BIEN: Solo renderiza items visibles
import { FlatList } from 'react-native';

const ProductList = ({ products }) => {
  const renderItem = useCallback(({ item }) => (
    <ProductCard product={item} />
  ), []);

  const keyExtractor = useCallback((item) => item.id.toString(), []);

  return (
    <FlatList
      data={products}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      removeClippedSubviews={true} // Android optimization
      maxToRenderPerBatch={10}
      updateCellsBatchingPeriod={50}
      initialNumToRender={10}
      windowSize={5}
      getItemLayout={(data, index) => ({
        length: ITEM_HEIGHT,
        offset: ITEM_HEIGHT * index,
        index,
      })} // Si todos los items tienen misma altura
    />
  );
};
```

**Resultado**: 20 FPS → 55 FPS

## Optimización 2: Memoization

### Problema: Re-renders innecesarios

```typescript
// ProductCard se re-renderiza aunque product no cambió
const ProductCard = ({ product, onPress }) => {
  console.log('Rendering:', product.id);

  return (
    <TouchableOpacity onPress={() => onPress(product.id)}>
      <Text>{product.name}</Text>
      <Text>${product.price}</Text>
    </TouchableOpacity>
  );
};
```

### Solución: React.memo

```typescript
const ProductCard = React.memo(({ product, onPress }) => {
  console.log('Rendering:', product.id);

  return (
    <TouchableOpacity onPress={() => onPress(product.id)}>
      <Text>{product.name}</Text>
      <Text>${product.price}</Text>
    </TouchableOpacity>
  );
}, (prevProps, nextProps) => {
  // Solo re-render si product cambió
  return prevProps.product.id === nextProps.product.id &&
         prevProps.product.price === nextProps.product.price;
});
```

### useCallback para funciones

```typescript
const ProductList = ({ products }) => {
  // MAL: Nueva función en cada render
  // const handlePress = (id) => { ... };

  // BIEN: Función memoizada
  const handlePress = useCallback((id) => {
    navigation.navigate('ProductDetail', { id });
  }, [navigation]);

  return (
    <FlatList
      data={products}
      renderItem={({ item }) => (
        <ProductCard product={item} onPress={handlePress} />
      )}
    />
  );
};
```

## Optimización 3: Imágenes

### Antes: Imágenes sin optimizar

```typescript
<Image
  source={{ uri: product.imageUrl }}
  style={{ width: 300, height: 300 }}
/>
```

Problemas:
- Carga imágenes en resolución completa
- No hay caching
- Consume mucha memoria

### Después: react-native-fast-image

```bash
npm install react-native-fast-image
```

```typescript
import FastImage from 'react-native-fast-image';

<FastImage
  source={{
    uri: product.imageUrl,
    priority: FastImage.priority.normal,
    cache: FastImage.cacheControl.immutable,
  }}
  style={{ width: 300, height: 300 }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

### Lazy loading de imágenes

```typescript
const LazyImage = ({ source, style }) => {
  const [loaded, setLoaded] = useState(false);

  return (
    <View style={style}>
      {!loaded && (
        <View style={[style, styles.placeholder]}>
          <ActivityIndicator />
        </View>
      )}
      <FastImage
        source={source}
        style={style}
        onLoad={() => setLoaded(true)}
      />
    </View>
  );
};
```

## Optimización 4: Animaciones nativas

### Antes: JS-based animations (laggy)

```typescript
// Corre en JS thread, se traba fácil
const opacity = useState(new Animated.Value(0))[0];

Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: false, // MAL
}).start();
```

### Después: Native driver

```typescript
const opacity = useState(new Animated.Value(0))[0];

Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // BIEN - corre en UI thread
}).start();
```

### react-native-reanimated para animaciones complejas

```bash
npm install react-native-reanimated
```

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

const AnimatedBox = () => {
  const offset = useSharedValue(0);

  const animatedStyles = useAnimatedStyle(() => {
    return {
      transform: [{ translateX: offset.value }],
    };
  });

  const handlePress = () => {
    offset.value = withSpring(offset.value + 50);
  };

  return (
    <>
      <Animated.View style={[styles.box, animatedStyles]} />
      <Button onPress={handlePress} title="Move" />
    </>
  );
};
```

## Optimización 5: Reducir JS bundle size

### Analizar bundle

```bash
npx react-native-bundle-visualizer
```

### Lazy import de pantallas

```typescript
// Antes: Import todo upfront
import HomeScreen from './screens/HomeScreen';
import ProfileScreen from './screens/ProfileScreen';
import SettingsScreen from './screens/SettingsScreen';

// Después: Lazy load
const HomeScreen = React.lazy(() => import('./screens/HomeScreen'));
const ProfileScreen = React.lazy(() => import('./screens/ProfileScreen'));
const SettingsScreen = React.lazy(() => import('./screens/SettingsScreen'));
```

### Remover librerías pesadas

```bash
# Antes
moment.js (67kB)

# Después
date-fns (12kB, tree-shakeable)
```

```typescript
// Antes
import moment from 'moment';
const formattedDate = moment(date).format('DD/MM/YYYY');

// Después
import { format } from 'date-fns';
const formattedDate = format(date, 'dd/MM/yyyy');
```

## Optimización 6: Navigation optimizations

### Lazy screens

```typescript
import { createStackNavigator } from '@react-navigation/stack';

const Stack = createStackNavigator();

<Stack.Navigator>
  <Stack.Screen
    name="Home"
    component={HomeScreen}
    options={{
      lazy: true, // No montar hasta que se navegue aquí
    }}
  />
</Stack.Navigator>
```

### Freeze inactive screens

```bash
npm install react-native-screens
```

```typescript
import { enableScreens } from 'react-native-screens';

enableScreens(); // En index.js

// Screens que no están activos se "freezan" (no ejecutan JS)
```

## Optimización 7: Redux selectors

### Antes: Re-render en cada cambio de state

```typescript
const ProductList = () => {
  // Re-render si CUALQUIER COSA en store cambia
  const products = useSelector(state => state.products.items);

  return <FlatList data={products} ... />;
};
```

### Después: Selectores memoizados

```bash
npm install reselect
```

```typescript
import { createSelector } from 'reselect';

// Selector memoizado
const selectVisibleProducts = createSelector(
  state => state.products.items,
  state => state.filters.category,
  (products, category) => {
    if (!category) return products;
    return products.filter(p => p.category === category);
  }
);

const ProductList = () => {
  const products = useSelector(selectVisibleProducts);

  return <FlatList data={products} ... />;
};
```

## Optimización 8: Hermes Engine

Hermes: JS engine optimizado para React Native

### Habilitar Hermes

```javascript
// android/app/build.gradle
project.ext.react = [
    enableHermes: true
]
```

```ruby
# ios/Podfile
use_react_native!(
  :hermes_enabled => true
)
```

```bash
cd ios && pod install
```

**Resultados**:
- App size: -50%
- Startup time: -30%
- Memory usage: -40%

## Optimización 9: InteractionManager

Diferir trabajo pesado hasta que interacciones terminen:

```typescript
const HeavyScreen = () => {
  const [data, setData] = useState(null);

  useEffect(() => {
    InteractionManager.runAfterInteractions(() => {
      // Esperar a que animaciones de navegación terminen
      fetchHeavyData().then(setData);
    });
  }, []);

  if (!data) {
    return <ActivityIndicator />;
  }

  return <HeavyComponent data={data} />;
};
```

## Optimización 10: Profiling en producción

### Flipper para debugging

```bash
# Instalar Flipper
brew install flipper

# Conectar app
npx react-native doctor
```

Flipper plugins:
- React DevTools
- Network inspector
- Layout inspector
- Performance monitor

### Sentry para crash reporting

```bash
npm install @sentry/react-native
```

```typescript
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'your-dsn',
  enableAutoSessionTracking: true,
  tracesSampleRate: 0.1, // 10% de traces
});

// Wrap app
export default Sentry.wrap(App);
```

## Checklist de performance

- [ ] FlatList con virtualization
- [ ] React.memo para componentes pesados
- [ ] useCallback para funciones en props
- [ ] useMemo para cálculos costosos
- [ ] FastImage para imágenes
- [ ] useNativeDriver: true en animaciones
- [ ] Hermes habilitado
- [ ] Bundle size optimizado
- [ ] Navigation lazy loading
- [ ] react-native-screens habilitado
- [ ] Redux selectors memoizados
- [ ] InteractionManager para trabajo pesado
- [ ] Profiling con Flipper
- [ ] Crash reporting con Sentry

## Resultados finales

**Antes**:
- FPS: 15-20
- Startup: 3.5s
- Memory: 280MB
- Crashes: 5%

**Después**:
- FPS: 55-60
- Startup: 1.2s
- Memory: 120MB
- Crashes: 0.5%

**Impacto en negocio**:
- User retention +35%
- App Store rating: 3.2 → 4.6
- 80% menos quejas de performance

¿Qué optimizaciones te han dado mejores resultados en React Native?
