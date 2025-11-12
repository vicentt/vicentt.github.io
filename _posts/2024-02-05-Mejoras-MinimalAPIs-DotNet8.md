---
layout: post
title: Minimal APIs en .NET 8 - Nuevas capacidades que debes conocer
author: Vicente José Moreno Escobar
categories: [.NET, ASP.NET-Core, APIs]
published: true
---

> .NET 8 trae mejoras significativas a las Minimal APIs que las hacen aún más potentes

# ¿Qué son las Minimal APIs?

Las Minimal APIs llegaron con .NET 6 como una forma simplificada de crear APIs HTTP sin necesidad de controladores. Con .NET 8, Microsoft ha dado un paso más allá mejorando significativamente esta funcionalidad.

## Nuevas características en .NET 8 🚀

### 1. Soporte mejorado para formularios

Ahora podemos trabajar con formularios de manera nativa:

```csharp
app.MapPost("/upload", async (IFormFile file) =>
{
    var filePath = Path.Combine("uploads", file.FileName);
    using var stream = File.OpenWrite(filePath);
    await file.CopyToAsync(stream);

    return Results.Ok(new { fileName = file.FileName, size = file.Length });
});
```

### 2. Filtros de endpoint

Una de las mejoras más esperadas. Ahora podemos añadir filtros personalizados a nuestros endpoints:

```csharp
app.MapGet("/api/products", async (ProductService service) =>
{
    return await service.GetAllAsync();
})
.AddEndpointFilter(async (context, next) =>
{
    // Lógica antes del endpoint
    var result = await next(context);
    // Lógica después del endpoint
    return result;
});
```

### 3. Antiforgery tokens nativos

La seguridad mejora con soporte incorporado para tokens antiforgery:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAntiforgery();

var app = builder.Build();
app.UseAntiforgery();

app.MapPost("/api/data", (DataModel data) =>
{
    // El token se valida automáticamente
    return Results.Ok();
})
.RequireAntiforgeryToken();
```

## Comparativa con controladores tradicionales

Las Minimal APIs ahora están al mismo nivel de funcionalidad que los controladores tradicionales pero con menos ceremonía:

| Característica | Controllers | Minimal APIs (.NET 8) |
|---------------|-------------|----------------------|
| Routing | ✅ | ✅ |
| Model Binding | ✅ | ✅ |
| Validation | ✅ | ✅ |
| Filters | ✅ | ✅ |
| Código necesario | 🟡 Alto | 🟢 Bajo |

## ¿Cuándo usar Minimal APIs?

En mi experiencia, las Minimal APIs son ideales para:

- **Microservicios** donde la simplicidad es clave
- **APIs pequeñas** con pocos endpoints
- **Prototipos rápidos** que luego se pueden migrar
- **Backends simples** para aplicaciones móviles o SPAs

Para aplicaciones grandes con muchos endpoints, los controladores tradicionales pueden seguir siendo más organizados.

# Migración desde .NET 6/7

La migración es prácticamente transparente. La mayor parte del código de Minimal APIs de versiones anteriores funciona sin cambios, pero ahora tienes acceso a estas nuevas capacidades.

## Conclusión

.NET 8 ha cerrado la brecha entre Minimal APIs y controladores tradicionales. La elección ahora depende más del estilo de código que prefieras que de las capacidades técnicas.

Si aún no has probado las Minimal APIs en .NET 8, te recomiendo encarecidamente que lo hagas. La productividad que se gana es considerable.

¿Ya estás usando .NET 8? ¿Qué te parecen estas mejoras? Me encantaría conocer tu opinión.

¡Hasta la próxima! 👋
