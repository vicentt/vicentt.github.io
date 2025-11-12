---
layout: post
title: C# 12 Primary Constructors - Simplificando clases y records
image: /images/aspnetcore.png
author: Vicente José Moreno Escobar
categories: [C#, .NET, Programming]
published: true
---
![C# 12](/images/aspnetcore.png)

> C# 12 trae Primary Constructors a clases normales, no solo records. Y cambia mucho.

# ¿Qué son los Primary Constructors?

Los Primary Constructors existían en C# desde la versión 9, pero solo para `record` types. C# 12 los extiende a `class` y `struct`.

## El antes y el después 📝

### Antes (C# 11)

```csharp
public class ProductService
{
    private readonly ILogger<ProductService> _logger;
    private readonly IProductRepository _repository;
    private readonly IMapper _mapper;

    public ProductService(
        ILogger<ProductService> logger,
        IProductRepository repository,
        IMapper mapper)
    {
        _logger = logger;
        _repository = repository;
        _mapper = mapper;
    }

    public async Task<ProductDto> GetProductAsync(int id)
    {
        _logger.LogInformation("Fetching product {Id}", id);
        var product = await _repository.GetByIdAsync(id);
        return _mapper.Map<ProductDto>(product);
    }
}
```

### Después (C# 12)

```csharp
public class ProductService(
    ILogger<ProductService> logger,
    IProductRepository repository,
    IMapper mapper)
{
    public async Task<ProductDto> GetProductAsync(int id)
    {
        logger.LogInformation("Fetching product {Id}", id);
        var product = await repository.GetByIdAsync(id);
        return mapper.Map<ProductDto>(product);
    }
}
```

**13 líneas menos**. Y más legible.

## Cómo funcionan realmente 🔍

Los parámetros del primary constructor están disponibles en:

1. **Inicializadores de campos y propiedades**
2. **Cuerpo de la clase**
3. **Métodos y propiedades**

```csharp
public class OrderProcessor(
    ILogger<OrderProcessor> logger,
    decimal taxRate)
{
    // Se pueden usar en inicializadores
    private readonly decimal _taxRate = taxRate > 0 ? taxRate : 0.21m;

    // Se pueden usar en propiedades
    public decimal TaxRate => taxRate;

    // Se pueden usar en métodos
    public void ProcessOrder(Order order)
    {
        logger.LogInformation("Processing order {OrderId}", order.Id);
        order.TotalAmount *= (1 + taxRate);
    }
}
```

## Casos de uso prácticos 💼

### 1. Servicios con Dependency Injection

Esto es oro para ASP.NET Core:

```csharp
public class WeatherForecastController(
    IWeatherService weatherService,
    ILogger<WeatherForecastController> logger) : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> Get(string city)
    {
        logger.LogInformation("Getting weather for {City}", city);
        var forecast = await weatherService.GetForecastAsync(city);
        return Ok(forecast);
    }
}
```

### 2. Configuración con Options pattern

```csharp
public class EmailService(
    IOptions<EmailSettings> options,
    ISmtpClient smtpClient)
{
    private readonly EmailSettings _settings = options.Value;

    public async Task SendAsync(string to, string subject, string body)
    {
        await smtpClient.SendMailAsync(
            _settings.FromAddress,
            to,
            subject,
            body);
    }
}
```

### 3. Combinación con field keyword (C# 13)

En C# 13 llegará `field`, que complementa esto perfectamente:

```csharp
public class Product(string name, decimal price)
{
    public string Name { get; set; } = name;

    public decimal Price
    {
        get => field;
        set => field = value >= 0 ? value : 0;
    } = price;
}
```

## Diferencias con records 🤔

Los primary constructors en `class` y `record` no son idénticos:

```csharp
// Record: genera propiedades automáticas
public record PersonRecord(string Name, int Age);

var person = new PersonRecord("John", 30);
Console.WriteLine(person.Name); // "John"

// Class: NO genera propiedades automáticas
public class PersonClass(string name, int age);

var person = new PersonClass("John", 30);
// Console.WriteLine(person.Name); // ❌ No compila
```

Para exponer parámetros como propiedades en clases:

```csharp
public class Person(string name, int age)
{
    public string Name { get; } = name;
    public int Age { get; } = age;
}
```

## Gotchas y consideraciones ⚠️

### 1. Captura de parámetros

Los parámetros se capturan como campos privados implícitos:

```csharp
public class Counter(int initialValue)
{
    private int _count = initialValue;

    // initialValue aún está disponible
    public void Reset() => _count = initialValue;
}
```

Esto tiene implicaciones de memoria si los parámetros son objetos grandes.

### 2. Constructores adicionales deben delegar

```csharp
public class DatabaseConnection(string connectionString)
{
    // ✅ Correcto
    public DatabaseConnection() : this("DefaultConnection")
    {
    }

    // ❌ No compila
    // public DatabaseConnection() { }
}
```

### 3. Serialización

Algunos serializadores (JSON.NET, System.Text.Json) pueden tener problemas si no defines propiedades explícitas:

```csharp
// ❌ Puede no serializarse correctamente
public class Product(string name, decimal price);

// ✅ Mejor para serialización
public class Product(string name, decimal price)
{
    public string Name { get; } = name;
    public decimal Price { get; } = price;
}
```

## Mi opinión personal 💭

Después de usar primary constructors en varios proyectos, mi take:

**Pros:**
- Código mucho más limpio en servicios con DI
- Reduce boilerplate significativamente
- Mejora legibilidad en clases pequeñas

**Contras:**
- Puede ser confuso cuando se mezcla con constructores tradicionales
- La captura implícita puede llevar a memory leaks si no se tiene cuidado
- Tooling aún está madurando (refactorings, analyzers)

**Mi regla:**
- ✅ Úsalos en: Services, Controllers, pequeñas clases de infraestructura
- ❌ Evítalos en: Entities, DTOs que se serializan, clases con lógica compleja de inicialización

## Migración desde código existente

Rider y Visual Studio ofrecen refactorings automáticos:

1. Click derecho en el constructor
2. "Convert to primary constructor"
3. Review los cambios (no siempre es trivial)

Para proyectos grandes, hazlo incrementalmente. No hay prisa.

## Compatibilidad

- **Requiere**: C# 12 + .NET 8 (o .NET 7 con langVersion=12)
- **Funciona con**: Todos los frameworks (.NET Framework 4.8+, .NET Standard 2.0+)

```xml
<PropertyGroup>
  <LangVersion>12</LangVersion>
</PropertyGroup>
```

## Conclusión

Primary constructors son una adición bienvenida a C#. Reducen ceremonia sin perder expresividad.

No son revolucionarios, pero para quien escribe servicios con DI todo el día (como yo), son un cambio de calidad de vida notable.

¿Ya los estás usando? ¿Qué opinas? Me encantaría saber si has encontrado casos de uso interesantes o problemas que yo no haya mencionado.

¡Feliz coding! 🚀
