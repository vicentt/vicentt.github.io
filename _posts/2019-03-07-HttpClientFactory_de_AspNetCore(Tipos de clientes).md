---
layout: post
title: HttpClientFactory de Asp.Net Core 2.1 (Tipos de Clientes)
image: /images/aspnetcore.png
author: Vicente José Moreno Escobar
categories: ASP.NET-Core
published: true 
---
![netcore](/images/aspnetcore.png)
> Tipos de uso de HttpClientFactory 

En mi post [HttpClientFactory de Asp.Net Core 2.1 (Introducción)](https://vicentt.github.io/HttpClientFactory_de_AspNetCore(Introducción)) hablamos la incorporacion de la factoría de clientes http en asp.net core. Las razones por las que el equipo de Microsoft a introducido esta factoría, la gran potencia que tiene y como empezar a utilizarla de forma básica. Seguimos con el tema y en este post vamos a hablar a cerca de los dos sabores que nos ofrece la factoría: Clientes con nombre(Named Clients) o clientes tipados(Typed Clients)

#### Clientes con nombre(Named Clients): ####

Es el tipo de cliente usado en el primer post de esta serie.
Con un cliente con nombre, podremos establecer ciertas caracteristicas para nuestros clientes en la ConfigureServices usando services.AddHttpClient()
Gracias a que hemos nombrado nuestro cliente, podremos acceder a él en disintas partes de nuestro código usando su nombre y sus caracteristicas vendrán con él. Evidentemente podremos registrar todos los clientes con nombre que queramos siempre y cuando su nombre sea único.

AddHttpClient recibe una string como primer argumento que servirá como identificador del cliente. El segundo argumento será una lambda con la configuración del cliente. 

<script src="https://gist.github.com/vicentt/2aac3a77bde7a19a2bb38a5c9da17bdf.js"></script>

En las ubicaciones del código donde queramos utilizar este cliente, lo crearemos mediante inserción de dependencias solicitandolo a través de su identificador.

<script src="https://gist.github.com/vicentt/1df5d58cc38c3c2cf718505c7c8c1015.js"></script>

Un problema con estos clientes es la necesidad de usar magic strings para identificarlos. Reconocido esto como una mala práctica, podemos bordear el problema creando una clase estática con constantes con los nombres de nuestros clientes y luego usar esta clase para identificarlos.

#### Clientes tipados(Typed Clients): ####

Los clientes tipados nos permiten definir una clase que espera un HttpClient insertado en su constructor. Una vez que tenemos nuestra clase, podemos exponer el HttpClient directamente(cosa que no tiene mucho sentido) o encapsular el uso de HttpClient para que toda la lógica relacionada con él esté en un mismo lugar.

En este ejemplo defino el cliente mediante HttpClient y le asigno un tipo MyGitHubClient

<script src="https://gist.github.com/vicentt/6101ebe1df0303e58fd4058741eb88a6.js"></script>


En este otro, expongo la clase MyGitHubClient

<script src="https://gist.github.com/vicentt/4d684ac3da999549b8af6ffe918be224.js"></script>

Recibe un HttpClient en su constructor mnediante DI y encapsula un endpoint de dicho HttpClient en un método.

En este controlador usamos nuestra clase MyGitHubClient

<script src="https://gist.github.com/vicentt/c3833c4d666c92d08366285c07980649.js"></script>

Recibimos una instancia de dicha clase mediante DI(No tenemos que hacer nada ya que viene gracias a AddHttpClient) y usamos el método que expusimos en ella.

[Docu Microsoft](https://docs.microsoft.com/es-es/dotnet/standard/microservices-architecture/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests)



