---
layout: post
title: Azure Container Apps en producción - Lecciones aprendidas
image: /images/azure-developer-associate-600x600.png
author: Vicente José Moreno Escobar
categories: [Azure, Containers, DevOps]
published: true
---
![Azure Container Apps](/images/azure-developer-associate-600x600.png)

> Mi experiencia llevando Azure Container Apps a un entorno de producción real

# El contexto 🎯

Hace unos meses tuve la oportunidad de migrar una aplicación de Azure App Services a Azure Container Apps. La promesa era clara: mejor control, escalado automático más fino y costes optimizados.

Después de varios meses en producción, quiero compartir lo que he aprendido.

## ¿Qué son Azure Container Apps?

Para quien no las conozca, Azure Container Apps es un servicio serverless que permite ejecutar contenedores sin gestionar infraestructura. Está construido sobre Kubernetes pero abstrae toda esa complejidad.

### Ventajas que he encontrado 👍

**1. Escalado basado en eventos**

El escalado KEDA (Kubernetes Event-Driven Autoscaling) es una maravilla:

```yaml
scale:
  minReplicas: 0
  maxReplicas: 10
  rules:
  - name: http-rule
    http:
      metadata:
        concurrentRequests: '50'
```

Poder escalar hasta 0 réplicas cuando no hay tráfico ha reducido nuestros costes un 40%.

**2. Revisiones inmutables**

Cada despliegue crea una nueva revisión. Hacer rollback es instantáneo:

```bash
az containerapp revision set-active \
  --name myapp \
  --resource-group mygroup \
  --revision myapp--revision01
```

**3. Traffic splitting**

Dividir tráfico entre versiones para hacer canary deployments es trivial:

```yaml
traffic:
- revisionName: myapp--v1
  weight: 90
- revisionName: myapp--v2
  weight: 10
```

### Desafíos encontrados 🤔

**1. Cold starts**

Si escalas a 0, el primer request después de inactividad puede tardar 10-15 segundos. Para APIs públicas, esto puede ser problemático.

**Solución**: Mantener al menos 1 réplica mínima en horario de uso.

**2. Límites de timeout**

Por defecto hay un timeout de 240 segundos. Para procesos largos, hay que usar Dapr o dividir el trabajo.

**3. Debugging complejo**

Los logs están en Log Analytics, lo cual está bien, pero debuggear problemas puede ser más complejo que en App Services.

## Comparativa con otras opciones

| Servicio | Complejidad | Flexibilidad | Coste |
|----------|-------------|--------------|-------|
| App Services | 🟢 Baja | 🟡 Media | 🟡 Medio |
| Container Apps | 🟡 Media | 🟢 Alta | 🟢 Bajo* |
| AKS | 🔴 Alta | 🟢 Muy Alta | 🔴 Alto |

*Con escalado a 0

## Consejos prácticos 💡

### 1. Usa managed identities desde el día 1

```bicep
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  identity: {
    type: 'SystemAssigned'
  }
}
```

Evita secretos hardcodeados y usa Azure AD para todo.

### 2. Configura health probes correctamente

```yaml
probes:
- type: liveness
  httpGet:
    path: "/health/live"
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

Un health probe mal configurado puede causar reinicios constantes.

### 3. Monitoriza Application Insights desde el contenedor

No confíes solo en los logs de Azure. Instrumenta tu aplicación:

```csharp
builder.Services.AddApplicationInsightsTelemetry();
```

## ¿Lo recomendaría? ✅

Sí, absolutamente. Para aplicaciones modernas basadas en contenedores, Azure Container Apps es un sweet spot perfecto entre simplicidad y potencia.

**Casos de uso ideales:**
- APIs REST con tráfico variable
- Workers procesando colas
- Aplicaciones con múltiples servicios comunicándose
- Prototipos que necesitan escalar

**Evitar si:**
- Necesitas persistencia local (usa volúmenes Azure Files)
- Tienes procesos que tardan más de 4 minutos por request
- Tu equipo no está familiarizado con contenedores

# Conclusión

Azure Container Apps ha madurado mucho desde su lanzamiento. Es una opción sólida que considero seria para cualquier proyecto nuevo en Azure.

El equilibrio entre facilidad de uso y capacidades es excelente. Si ya usas contenedores, la curva de aprendizaje es mínima.

¿Alguien más tiene experiencia con Container Apps? Me encantaría conocer otros casos de uso.

¡Nos vemos en el próximo post! 🚀
