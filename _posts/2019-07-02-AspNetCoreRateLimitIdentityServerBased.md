---
layout: post
title: AspNetCoreRateLimit con límites basados en claims de identidad
image: /images/aspnetcore.png
author: Vicente José Moreno Escobar
categories: ASP.NET-Core IdentityServer4
published: true 
---
![netcore](/images/ratelimitsIS.jpg)
> Extendiendo la librería AspNetCoreRateLimits para establecer límites de request basados en claims de la identidad del usuario

#### Conociendo le problema y su solución: ####

En muy habitual que en el desarrollo de APIs modernas necesitemos **establecer límites sobre las solicitudes que son realizadas por un cliente.**
Ya sea por necesidades de negocio o para estar a salvo de hotlinkers o de atacantes debemos proteger siempre nuestras APIs con alguna solución de límite de solicitudes.

Atendiendo a esta necesidad existen múltiples opciones que podemos implementar. Una de ellas es la que presento hoy y a la que voy a intentar dar un cuarto de vuelta. 

#### AspNetCoreRateLimits de Stefan Prodan ####

Se trata del proyecto **AspNetCoreRateLimit**. Es un proyecto disponible en GitHub y también como Nuget. Este proyecto ha sido desarrollado por [Stefan Prodan](https://stefanprodan.com/)

[Repo GitHub de AspNetCoreRateLimit](https://github.com/stefanprodan/AspNetCoreRateLimit)

En mi opinión es un buen producto y cumple con lo que promete pero tiene una cuestión que creo que puede hacer **difícil** su implementación en proyectos reales.
**Toda la configuración a cerca de limites parte del fichero de configuración appsettings.json.** Esto puede ser problemático debido a que lo más probable es que necesitemos que esa configuración sea más dinámica.

#### Extendiendo la funcionalidad de AspNetCoreRateLimits ####
Nuestra necesidad es la de identificar los límites establecidos a partir de la información de identidad que hemos adquirido cuando se han identificado para usar nuestra Api.

He creado un proyecto de ejemplo muy sencillo para ilustrar...

[Repo GitHub del Ejemplo](https://github.com/vicentt/AspNetCoreRateLimitWhitIdentity)

El proyecto de ejemplo se compone de una solución en la que tenemos una **Api**, un **Cliente** y un **IdentityServer4** que levantaremos en memoria

En la Api se usa **AspNetCoreRateLimit** para establecer límites de solicitudes. Vamos a tener **dos EndPoints:** Uno en el que necesitaremos autorización y otro con acceso anónimo. De la autorización se va a encargar una instancia de IdentityServer4 levantado en memoria. El flow de autorización elegido ha sido ClientCredentials por motivos de sencillez y la claridad del ejemplo.

Gracias a una de las grandes evoluciones que nos trae **.Net Core** que es la facilidad en la extensión de las funcionalidades que nos ofrecen las librerías, vamos a poder extender su funcionalidad del producto

En las propias instrucciones de implementación del proyecto Stefan nos indica cómo podemos hacerlo.

Para conseguir nuestro objetivo debemos proveer **AspNetCoreRateLimit** de una manera de identificar al objeto de limitación. En este caso va a ser el cliente **(Client_Id)** pero podríamos utilizar otros criterios
Esto vamos a hacerlo proveyendo a la librería de una configuración creada por nosotros para tal efecto.


<script src="https://gist.github.com/vicentt/1c0af52924285866a0a6f3b01f6beacc.js"></script>


Utilizamos **IHttpContextAccessor** inyectado mediante Dependency Injection en startup.cs y se lo pasamos a una clase llamada **IdentityResolveContributor**. En esta clase existe un método **ResolveClient** descrito en la interface **IClientResolveContributor** de **AspNetCoreRateLimit** que será el usado por la librería para obtener la identidad en la que se basarán las políticas de límites. 

En este método usaremos **HttpContext** de de la solicitud para extraer las características(Claims) del cliente que se ha identificado. En este caso usaremos el **Client_Id** que es el claim que se usa en **ClientCredentials** para identificar al cliente. En caso de que no tengamos cliente identificado, devolvemos “anon”.


<script src="https://gist.github.com/vicentt/92b2466e33826d222f992fe2dcac3564.js"></script>


Por otra parte vamos a crear un **Middleware** en el cual interceptaremos cada request para comprobar los límites e incrementarlos si es necesario. 


<script src="https://gist.github.com/vicentt/42d933da8d93b2dd4f4a72222e79112b.js"></script>


Nuestro **Middleware** recibe de DI el **equestDelegate** que le va a permitir llamar al siguiente middleware, la colección de opciones configuradas en appsettings.json, y el almacén de políticas que utiliza **AspNetCoreRateLimit**.

En el método **Invoke**, en primer lugar, vamos a sacar del **HttpContex**t el **Claim Client_Id**. 
Con él vamos a comprobar si en el almacén de políticas de cliente existe una política para este cliente. En caso de que no exista, creamos una estableciendo el límite que sacamos del **Claim client_AspNetRateLimit24h** y finalmente seteamos esta política. 
A partir de este momento nuestro cliente podrá hacer las llamadas establecidas en el claim durante las siguientes 24 horas.

El objetivo de este post es mostrar un ejemplo funcional de una extensión de **AspNetCoreRateLimit**. A partir de aquí y según las necesidades de nuestro proyecto podemos imaginar múltiples escenarios de uso.

Cualquier sugerencia, crítica o duda será recibida con anímo y alborozo.


