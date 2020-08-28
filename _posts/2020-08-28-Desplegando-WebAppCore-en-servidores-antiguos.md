---
layout: post
title: Trucos desplegando WebApp .NetCore 3.1 en un servidor antiguo
image: /images/Old-Server.png
author: Vicente José Moreno Escobar
categories: ASP.NET-Core 
published: true 
---
![Old Server](/images/Old-Server.png)

Es absolutamente impresionante lo que ha avanzado el desarrollo de software en los últimos 15 años. Hemos pasado de copiar carpetas directamente desde nuestra máquina a un único servidor, que era nuestro único nexo con internet a desplegar de manera automatizada en miles de contenedores, por anillos, a través del control de código y monitorizando todo el ciclo de entrega de valor de cada uno de nuestros desplieges, entre otras cosas... 

Lamentablemente, sobre todo en entornos de desarrollo temprano, seguimos teniendo que mantener servidores OnPrem que no siempre están en las mejores condiciones. Ya sea por cuestiones de licencia o por que nuestros amigos de sistemas no se preocupan mucho de ellos...

Es de lo que voy a hablar hoy. Daré algunos trucos para publicar una aplicación realizada con lo que a día de hoy es casi lo último en el ecosistema .Net, en un servidor cuyas versiones instaladas de SO, IIS, Frameworks son antiguas.

Por lo menos podemos contar con que, gracias a las características de .Net Core, esto es posible.

Para confeccionar esta lista, he trabajado publicando en un Windows Server 2008, con un IIS 7.0


# Tip 1 Self-Contained:

Gracias a las características (benditas) que hacen de .Net Core Cross-Platform, podemos seleccionar a la hora de crear nuestro perfil de publicación la opción "Self Contained".
Esta opción creará en nuestro destino de publicación todo lo necesario para que la aplicación funcione sin depender de ningung .Net Framework. 

[.NET Core application publishing overview](https://docs.microsoft.com/en-us/dotnet/core/deploying/) 


# Tip 2 Application Pools:

Como hemos publicado con "Self Contained" configuraremos los Application Pools de IIS como "No Managed Code"
Otra consideración al respecto de los Application Pools es que deberémos crear uno por cada aplicación. Varias aplicaciones no pueden compartir un Application Pool.


# Tip 3 Web.Config:

Aunque creíamos que nos habíamos despedido para siempre del fichero web.config, esa breva no ha caido. El sistema de publicación crea un fichero de configuración. Por lo menos ahora no tenemos que guardar ahi nuestras configuraciones. 

Una vez publicada la aplicación debemos editar el fichero web.config para cambiar el valor 

`modules="AspNetCoreModuleV2"` 

por 

`modules="AspNetCoreModule"`



¡Hasta la próxima!





