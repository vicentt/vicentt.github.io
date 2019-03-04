---
layout: post
title: HttpClientFactory de Asp.Net Core 2.1 (Introducción)
image: /images/aspnetcore.png
author: Vicente José Moreno Escobar
categories: ASP.NET-Core
published: true 
---

> HttpClientFactory es una fábrica bien fundamentada, disponible desde .NET Core 2.1, para crear instancias de HttpClient con el fin de usarlas en las aplicaciones. 

Nos ayudará resolver algunos problemas al utilizar HttpClient.

### Problemas de la clase original HttpClient: ###
![netcore](/images/aspnetcore.png)

Aunque **HttpClient** es una clase extremadamente común y muy fácil de utilizar, en muchas ocasiones no la usamos correctamente:

Debemos partir del hecho de que **el sistema asigna un distinto socket a cada instancia HttpClient**. Así que el crear múltiples instancias de **HttpClient** según las vamos necesitando no es buena idea debido a que los sockets subyacentes no se liberan de forma inmediata cuando las instancias dejan de ser usadas.

Aunque la clase HttpClient es disposable, tampoco es buena idea usarla mediant ***using*** ya que el socket tampoco es liberando cuando abandonamos el bloque. 

En ambos casos podemos estar causando un problema denominado "agotamiento de socket" [You're using HttpClient wrong and it is destabilizing your software](https://aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/)

Incluso en el caso de estar utilizando **HttpClient** mediante la creación de objetos singleton o estaticos, seguimos teniendo ciertos problemas como que dichos objetos no serán permeables a los cambios de DNS.

Por todo esto y por mas cosas, .NET Core 2.1 nos ofrece una nueva factoría llamada **HttpClientFactory** mediante la cual vamos a poder manejar nuestros dichosos **HttpClient**

Hay un detalle que es remarcable en cuanto al uso de la factoría:

Si antes decíamos que crear uno cada vez esta mal y que crearlos de manera *“estática”* tambien esta mal ¿Como y que hace esta factoría?

La respuesta es que cada vez que se obtiene un objeto **HttpClient** de la factoría **IHttpClientFactory**, esta devuelve una instancia nueva. Pero todas estas instancias usan un controlador que las agrupa del tipo **IHttpMessageHandler** y que es el encargado de reducir y optimizar el uso de los recursos a la hora de que nuestras instancias se comuniquen con los recursos externos y por tanto abstraernos de los problemas antes mencionados.

Este IHttpMessageHandlers que agrupará varias instancias de **HttpClient** asociadas a un servicio concreto tienen un tiempo de refresco de 2 minutos. Después de los cuales renovarán las conexiones de las instancias de **HttpClient** que agrupen.
Esta configuración es manejable por nosotros si así lo deseamos pero eso llegará mas adelante.

Y sin mas pasamos a una pequeña parte de código con la que ilustrar el ejemplo mas básico de uso:

SPOILER:Nunca la vamos a usar así

Registraremos un servicio de tipo **IHttpClientFactory** en nuestra colección de servicios que representa nuestro Inyector de Dependencias mediante: 

`services.AddHttpClient();`

Esta invocación registrara (detras del telon) unos cuantos servicios entre los que está una implementación de IClientFactory

En nuestro ValuesController lo usaremos así:

<script src="https://gist.github.com/vicentt/09872bd8e3f892b4238c9b3ae823dcd7.js"></script>

Aquí estamos añadiendo una dependencia a IHttpClientFactory en nuestro constructor que será inyectada por el sistema de inyección de dependencias. Con IHttpClientFactory que nos es inyectada podemos pedir nuevos HttpClients.

En nuestra acción Get estamos usando HttpClientFactory para crear un nuevo cliente y a partir de aquí podremos hacer uso de él.

En siguientes post veremos maneras mas avanzadas de usar esta factoría asi como las cosas tan interesantes que nos permite realizar.

[Docu Microsoft](https://docs.microsoft.com/es-es/dotnet/standard/microservices-architecture/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests)



