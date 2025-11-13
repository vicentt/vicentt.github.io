---
layout: post
title: .NET 6 Minimal APIs en producción - Experiencia real
author: Vicente José Moreno Escobar
categories: [.NET, APIs, Performance]
published: true
---

> Cuando decidimos reescribir nuestra API REST con Minimal APIs

# El contexto

API REST tradicional con ASP.NET Core 5:
- 40+ controllers
- Startup.cs con 300 líneas
- Program.cs básico
- Mucho boilerplate

.NET 6 trajo **Minimal APIs**. Prometían menos código, mejor performance.

¿Valía la pena migrar? Spoiler: **Sí**.

## API tradicional vs Minimal API

### Antes (ASP.NET Core 5)

```csharp
// Startup.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddDbContext<AppDbContext>();
        services.AddScoped<IProductService, ProductService>();
        // ... 50 líneas más
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}

// ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    [HttpGet]
    public async Task<ActionResult<List<Product>>> GetProducts()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        if (product == null) return NotFound();
        return Ok(product);
    }

    [HttpPost]
    public async Task<ActionResult<Product>> CreateProduct(Product product)
    {
        await _productService.CreateAsync(product);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
}
```

### Después (Minimal API .NET 6)

```csharp
// Program.cs (todo en un archivo)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>();
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Endpoints directos
app.MapGet("/api/products", async (IProductService service) =>
{
    return await service.GetAllAsync();
});

app.MapGet("/api/products/{id:int}", async (int id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is not null ? Results.Ok(product) : Results.NotFound();
});

app.MapPost("/api/products", async (Product product, IProductService service) =>
{
    await service.CreateAsync(product);
    return Results.Created($"/api/products/{product.Id}", product);
});

app.Run();
```

**60% menos código** para el mismo resultado.

## Organización: Route Groups

Para APIs grandes, agrupar endpoints:

```csharp
// Products/ProductEndpoints.cs
public static class ProductEndpoints
{
    public static void MapProductEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/products")
            .WithTags("Products")
            .RequireAuthorization();

        group.MapGet("/", GetAllProducts);
        group.MapGet("/{id:int}", GetProductById);
        group.MapPost("/", CreateProduct);
        group.MapPut("/{id:int}", UpdateProduct);
        group.MapDelete("/{id:int}", DeleteProduct);
    }

    private static async Task<IResult> GetAllProducts(IProductService service)
    {
        var products = await service.GetAllAsync();
        return Results.Ok(products);
    }

    private static async Task<IResult> GetProductById(int id, IProductService service)
    {
        var product = await service.GetByIdAsync(id);
        return product is not null ? Results.Ok(product) : Results.NotFound();
    }

    private static async Task<IResult> CreateProduct(
        Product product,
        IProductService service,
        IValidator<Product> validator)
    {
        var validationResult = await validator.ValidateAsync(product);
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(validationResult.ToDictionary());
        }

        await service.CreateAsync(product);
        return Results.Created($"/api/products/{product.Id}", product);
    }

    private static async Task<IResult> UpdateProduct(
        int id,
        Product product,
        IProductService service)
    {
        if (id != product.Id) return Results.BadRequest();

        var exists = await service.ExistsAsync(id);
        if (!exists) return Results.NotFound();

        await service.UpdateAsync(product);
        return Results.NoContent();
    }

    private static async Task<IResult> DeleteProduct(int id, IProductService service)
    {
        var exists = await service.ExistsAsync(id);
        if (!exists) return Results.NotFound();

        await service.DeleteAsync(id);
        return Results.NoContent();
    }
}

// Program.cs
var app = builder.Build();

app.MapProductEndpoints();
app.MapOrderEndpoints();
app.MapCustomerEndpoints();
```

## Filters para cross-cutting concerns

```csharp
// Validation filter
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (validator is not null)
        {
            var entity = context.Arguments.OfType<T>().FirstOrDefault();
            if (entity is not null)
            {
                var validationResult = await validator.ValidateAsync(entity);
                if (!validationResult.IsValid)
                {
                    return Results.ValidationProblem(validationResult.ToDictionary());
                }
            }
        }

        return await next(context);
    }
}

// Logging filter
public class LoggingFilter : IEndpointFilter
{
    private readonly ILogger<LoggingFilter> _logger;

    public LoggingFilter(ILogger<LoggingFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var stopwatch = Stopwatch.StartNew();

        var result = await next(context);

        stopwatch.Stop();

        _logger.LogInformation(
            "Endpoint {Endpoint} executed in {Duration}ms",
            context.HttpContext.Request.Path,
            stopwatch.ElapsedMilliseconds);

        return result;
    }
}

// Aplicar filters
group.MapPost("/", CreateProduct)
    .AddEndpointFilter<ValidationFilter<Product>>()
    .AddEndpointFilter<LoggingFilter>();
```

## OpenAPI/Swagger

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Products API",
        Version = "v1"
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Endpoints con metadata para Swagger
app.MapGet("/api/products/{id:int}", GetProductById)
    .WithName("GetProductById")
    .WithDescription("Get a product by ID")
    .Produces<Product>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status404NotFound)
    .WithTags("Products");
```

## Autenticación y autorización

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://auth.ejemplo.com";
        options.Audience = "products-api";
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Endpoint público
app.MapGet("/api/products", GetAllProducts);

// Endpoint requiere autenticación
app.MapPost("/api/products", CreateProduct)
    .RequireAuthorization();

// Endpoint requiere rol específico
app.MapDelete("/api/products/{id:int}", DeleteProduct)
    .RequireAuthorization("AdminOnly");
```

## Dependency Injection

Funciona igual que antes:

```csharp
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<IValidator<Product>, ProductValidator>();

// Los parámetros se inyectan automáticamente
app.MapGet("/api/products", async (
    IProductService productService,
    ILogger<Program> logger,
    HttpContext context) =>
{
    logger.LogInformation("Getting all products");
    return await productService.GetAllAsync();
});
```

## Binding de parámetros

```csharp
// Route parameters
app.MapGet("/api/products/{id:int}", (int id) => { });

// Query string
app.MapGet("/api/products", ([FromQuery] int page, [FromQuery] int size) => { });

// Body
app.MapPost("/api/products", ([FromBody] Product product) => { });

// Header
app.MapGet("/api/products", ([FromHeader(Name = "X-API-Key")] string apiKey) => { });

// Services (inyección automática)
app.MapGet("/api/products", (IProductService service) => { });

// HttpContext
app.MapGet("/api/products", (HttpContext context) => { });
```

## Validación con FluentValidation

```csharp
// ProductValidator.cs
public class ProductValidator : AbstractValidator<Product>
{
    public ProductValidator()
    {
        RuleFor(p => p.Name).NotEmpty().MaximumLength(100);
        RuleFor(p => p.Price).GreaterThan(0);
    }
}

// Extension method para validación
public static class ValidationExtensions
{
    public static async Task<IResult> ValidateAndExecute<T>(
        this T entity,
        IValidator<T> validator,
        Func<Task<IResult>> action)
    {
        var validationResult = await validator.ValidateAsync(entity);
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(validationResult.ToDictionary());
        }

        return await action();
    }
}

// Uso
app.MapPost("/api/products", async (Product product, IValidator<Product> validator, IProductService service) =>
{
    return await product.ValidateAndExecute(validator, async () =>
    {
        await service.CreateAsync(product);
        return Results.Created($"/api/products/{product.Id}", product);
    });
});
```

## Rate Limiting (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.Identity?.Name ?? context.Request.Headers.Host.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
});

var app = builder.Build();

app.UseRateLimiter();

app.MapGet("/api/products", GetAllProducts)
    .RequireRateLimiting("fixed");
```

## Testing

```csharp
// ProductEndpointsTests.cs
public class ProductEndpointsTests
{
    [Fact]
    public async Task GetAllProducts_ReturnsOk()
    {
        // Arrange
        await using var application = new WebApplicationFactory<Program>();
        var client = application.CreateClient();

        // Act
        var response = await client.GetAsync("/api/products");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task CreateProduct_WithInvalidData_ReturnsBadRequest()
    {
        await using var application = new WebApplicationFactory<Program>();
        var client = application.CreateClient();

        var product = new Product { Name = "", Price = -10 };

        var response = await client.PostAsJsonAsync("/api/products", product);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

## Performance benchmarks

Comparamos API tradicional vs Minimal API:

```
BenchmarkDotNet Results:

Traditional API:
- Startup time: 1200ms
- Memory: 45MB
- Requests/sec: 12,000

Minimal API:
- Startup time: 350ms ⚡ 71% faster
- Memory: 28MB ⚡ 38% less
- Requests/sec: 15,000 ⚡ 25% more throughput
```

## Migración gradual

No migramos todo de golpe:

```csharp
var app = builder.Build();

// Minimal APIs nuevos
app.MapProductEndpoints();
app.MapOrderEndpoints();

// Controllers viejos (siguen funcionando)
app.MapControllers();
```

Migramos endpoint por endpoint durante 2 meses.

## Cuándo NO usar Minimal APIs

- **APIs muy complejas** con mucha lógica en controllers
- **Teams grandes** que prefieren estructura tradicional
- **Mucho model binding custom**
- **Si ya tienes controllers funcionando bien** (no migrar por migrar)

## Cuándo SÍ usar Minimal APIs

- **APIs nuevas**
- **Microservicios pequeños**
- **Prototipos rápidos**
- **Performance es crítico**
- **Menos boilerplate preferido**

## Resultados después de 6 meses

- Startup time: -70%
- Memory usage: -35%
- Throughput: +25%
- Líneas de código: -40%
- Tiempo de desarrollo nuevos endpoints: -50%
- Satisfacción del equipo: 9/10

**¿Lo haríamos de nuevo?** Absolutamente.

¿Has probado Minimal APIs en producción? ¿Qué experiencia tuviste?
