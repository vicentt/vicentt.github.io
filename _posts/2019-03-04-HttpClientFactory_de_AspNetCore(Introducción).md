---
layout: post
title: HttpClientFactory de Asp.Net Core 2.1 (Introducción)
image: /images/cover-promocion-blogpocket-6.png
author: Vicente José Moreno Escobar
categories: ASP.NET Core
published: true 
---

> HttpClientFactory es una fábrica bien fundamentada, disponible desde .NET Core 2.1, para crear instancias de HttpClient con el fin de usarlas en las aplicaciones. 

Nos ayudará resolver algunos problemas al utilizar HttpClient.

### Problemas de la clase original HttpClient: ###

Aunque es una clase extremadamente común y muy fácil de utilizar, en muchas ocasiones, no la estamos usando correctamente:

Debemos partir del hecho de que el sistema asigna un distinto socket a cada instancia HttpClient. Así que el crear múltiples instancias de HttpClient según las vamos necesitando no es buena idea debido a que los sockets subyacentes no se liberan de forma inmediata cuando las instancias dejan de ser usadas. 
Aunque la clase HttpClient es disposable, tampoco es buena idea usarla mediant using ya que el socket tampoco es liberando cuando abandonamos el bloque. Podemos estar  causando un problema denominado "agotamiento de socket"
