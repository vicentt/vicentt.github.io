---
layout: post
title: HTTP API Problem Details en ASP.NET Core
image: /images/codeMan.jpg
author: Vicente José Moreno Escobar
categories: ASP.NET-Core 3.1
published: true 
---
![code](/images/codeMan.jpg)

#### Un poco de Background ####

Si alguna vez te has visto envuelto en el desarrollo de una Api HTTP sabrás que una parte crucial y que a veces puede dar bastante trabajo, es la devolución estandarizada de códigos HTTP no exitosos. 
Elegir la estrategia y mantenerla durante todo el desarrollo. ¿devuelvo el código sin mas contenido?

 -> ¿devuelvo el código sin mas contenido?
 -> Probablemente te estas quedando corto y estás haciendo que tu Api sea dificil de utilizar

 -> ¿devuelvo una excepción?
 -> No es buena idea ya que no es estandar, enseñas demasiado de las "tripas" de tu API y debes gestionar excepciones para casos que en las que no aplican como un 401 o un 403

 -> ¿Uso una solución propia que personalice en cada caso?
 -> Vas  a tener mucho trabajo a la hora de gestionar y personalizar los objetos que representan las respuestas. Va a ser código que vas a tener que mantener. Vas a tener que exponer ese modelo que tampoco es estandar y que probablemente en algun momento o en algun caso no sea del agrado de tus clientes o consumidores. 

 La cuestión es que este problema común está abordado en una especificación desde hace algún tiempo:
 Esta especificación es: [RFC 7807 Problem Details for HTTP APIs ](https://tools.ietf.org/html/rfc7807)

 Este podría ser un ejemplo básico de una respuestas ProblemDetails:
 ```json
{
    "status": 404,
    "type": "https://httpstatuses.com/404",
    "title": "Not Found",
    "detail": "No content on this category"
}
```

#### Problem Details en .Net ####

El equipo de .Net a incluido una implementación de esta especificación en la clase: [ProblemDetails](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.problemdetails?view=aspnetcore-2.2)

Un ejemplo de uso en un controlador de nuestra Api Http podría ser:
 ```c#
 [HttpGet]
public IActionResult GetProducts(long? category = null)
{
    try
    {
        /// Get Produtos of catégory id
    }
    catch (Exception ex)
    {
        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status404NotFound,
            Type = "https://example.com/probs/out-of-credit",
            Title = "Not Found",
            Detail = "No products found in this category",
            Instance = HttpContext.Request.Path
        };

        return new ObjectResult(problemDetails)
        {
            ContentTypes = { "application/problem+json" },
            StatusCode = 404,
        };
    }

    return Ok();
}
```

Existen infinidad de aproximaciones que podemos utilizar para sacarle partido a ProblemDetails. Utilizar una factoría para su creación y añadirla a un middleware de control de errores:

 ```c#
 public void ConfigureServices(IServiceCollection serviceCollection)
{
    services.AddControllers();
    services.AddTransient<ProblemDetailsFactory, CustomProblemDetailsFactory>();
}
 ```

 Utilizar Problem Details para manejar a nuestro antojo errores de validación:

 ```c#
 [HttpPost]
public ActionResult UpdateProduct(Product model)
{
    if (!ModelState.IsValid)
    {
        var problemDetails = new ValidationProblemDetails(ModelState);

        return new ObjectResult(problemDetails)
        {
            ContentTypes = { "application/problem+json" },
            StatusCode = 403,
        };
    }

    return Ok();
}
 ```

 
#### Hellang.Middleware.ProblemDetails ####

Ahora que ya sabemos lo que es y como usar ProblemDetails vamos a conocer una implementación que hará las delicias de los amantes del "Keep It Simple".
Gracias a [Hellang.Middleware.ProblemDetails](https://www.nuget.org/packages/Hellang.Middleware.ProblemDetails) vamos a utilizar todo el potencial de ProblemDetails con solo algunas lineas de código.

Despues de instalar el paquete en nuestro proyecto, solo tendremos que añadir el middleware a nuestro Pipeline para que la librería empiece a trabajar:
 
 ```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddProblemDetails(ConfigureProblemDetails)
        .AddControllers()
        .AddJsonOptions(x => x.JsonSerializerOptions.IgnoreNullValues = true);
}

 public void Configure(IApplicationBuilder app)
{
    app.UseProblemDetails();
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}

```
Solo con estas dos lineas de código hemos conseguido que todas las solitudes a nuestra Api Http que se pueden considerar problematicas sean respondidas con un objerto ProblemDetail. 
Como suele ser habitual, existen múltiples opciones de configuración y personalización en el uso de esta librería. 

Os invito a pasaros por el Repo de GitHub para descubrirlas [GitHub Hellang.Middleware.ProblemDetails](https://github.com/khellang/Middleware) 



