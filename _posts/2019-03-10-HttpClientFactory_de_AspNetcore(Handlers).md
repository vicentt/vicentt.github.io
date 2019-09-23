---
layout: post
title: HttpClientFactory de Asp.Net Core 2.1 (Handlers)
image: /images/aspnetcore.png
author: Vicente José Moreno Escobar
categories: ASP.NET-Core
published: true 
---
![netcore](/images/aspnetcore.png)
> Handlers con HttpClientFactory 


En los dos últimos post hemos estado hablando sobre HttpClientFactory. Hemos visto sus fundamentos y tambien algunas maneras de usar esta factoría.
En este post vamos ha hablar de una caracteristica que hace realmente potente el uso de HttpClients generados a través de la factoría HttpClientFactory. 

#### DelegatingHandler: ####

Cuando realizamos solicitudes Http es habitual que necesitemos tomar el control sobre ciertas cuestiones. Gestión de errores y resiliencia, registros de actividad o logging, o tal vez el uso de caches para aminorar el numero de peticiones Http.

DelegatingHandlers nos ayuda a resolver este tipo de temas. Con un patron del estilo de los middlewares podremos ir decidiendo lo que va a ocurrir con nuestra peticion saliente.

Definiremos una cadena de manejadores cuyo contenido se ejecutará antes de dejar salir la solicitud.

Al igual que en el patron middleware, cada pieza que compone nuestro pipeline tendrá la oportunidad de ejecutar su código de ida y de vuelta en orden inverso pudiendo realizar un cortocircuito o shortcircuit en caso de que sea necesario.

![netcore](/images/ihttpclientfactory-delegatinghandler-outgoing-middleware-pipeline.png)

Por ejemplo podríamos hacer una solicitud que necesita una determinada cabecera y en un handler establecer dicha cabecera de versión de api o de cualquier otro tipo. En la devolución de la solicitud podemos utilizar un handler para comprobar que la respuesta es un 200 y de no ser así, actuar en consecuencia sin necesidad de llegar hasta el método que realizó la llamada.

#### Creando un Handler: ####

Un handler es una clase que hereda de la clase abstracta DelegatingHandler. Sobreescribiremos el método SendAsync para meter nuestras funcionalidades.

<script src="https://gist.github.com/vicentt/696433482267611c0b7c1718641388fa.js"></script>


