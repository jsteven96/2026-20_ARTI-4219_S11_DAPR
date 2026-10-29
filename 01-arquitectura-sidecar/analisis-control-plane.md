# analisis-control-plane.md
## Análisis del Plano de Control

### 1. ¿Qué tipo de objeto de Kubernetes es el `dapr-sidecar-injector`?

```dapr-sidecar-injector``` es un Deployment. Este mantiene vivo el pod del injector. Si el pod muere, el ReplicaSet lo vuelve a crear. La MutatingWebhookConfiguration le pide al API Server de k8s que, cada vez que se crea un pod, consulte al injector a través del Service (por el puerto 443) antes de guardarlo. Si el pod tiene la anotación ```dapr.io/enabled: "true"```, el injector lo modifica y le adiciona el contenedor daprd. Ese contenedor es un sidecar.

### 2. Réplicas del `dapr-operator` y comparación con Istio

De acuerdo al API de k8s está corriendo solo una réplica del ```dapr-operator``` Esto debido a que se está utilizando un cuadro de helm para desarrollo. Ahora bien, haciendo la comparación con Istio, este es una malla de red. Su plano de control sincroniza continuamente la topología completa del clúster con cada proxy. Dapr es un runtime de APIs de aplicación. Su plano de control solo inyecta sidecars, entrega configuración y certificados, y casi no participa en las peticiones.

### 3. Flujo de inyección del sidecar

> _Tu respuesta aquí_

---

## Exploración de los CRDs de Dapr

### 4. CRDs instalados por Dapr

Dapr instaló 7 CRDs, bajo el grupo ```dapr.io``` y todos con namespace.

| Kind | apiVersion | Para qué sirve |
| - | - | - |
| Component | v1alpha1 | Es el más importante. Declara un bloque de construcción y su backend: pubsub, state store, bindings, secrets stores, entre otros |
| Configuration | v1alpha1 | Configura el comportamiento del sidecar: tracing, métricas mTLS, control de acceso a la invocación de servicios, features en preview |
| Subscription | v2alpha1 | Suscripciones declarativas a pub/sub (topic -> ruta de la aplicación) en lugar de que la aplicación las exponga en ```GET /dapr/suscribe``` |
| Resiliency | v1alpha1 | Políticas de timeouts, reintentos y circuit breakers aplicadas a apps, componentes o actores |

### 5. Relación entre `components.dapr.io` y el concepto de Building Block

El **Building Block es la abstracción y el Component es la implementación concreta**. El CRD ```components.dapr.io``` es la forma declarativa de decirle a Dapr qué implementación usar para cada building block.
