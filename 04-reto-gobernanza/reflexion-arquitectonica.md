# reflexion-arquitectonica.md

## Reto 3: Reflexión Arquitectónica

### 1. Sin Dapr: ¿dónde vive el código de TTL y consumer group?

Sin Dapr, usando el SDK de Kafka directamente, ambas responsabilidades quedan **dentro del código de cada aplicación**:

- **TTL:** Kafka no tiene expiración por mensaje. Solo ofrece retención a nivel de tópico (`retention.ms`), que borra segmentos completos del log y no garantiza que un mensaje vencido no se consuma. Para una regla como "las órdenes expiran a los 2 minutos", habría que programarlo a mano:
  - en el **producer**, agregar un header con el timestamp de creación o de expiración;
  - en el **consumer**, leer ese header antes de procesar cada mensaje y descartar los vencidos.

  Esa lógica se repetiría en cada servicio y en cada lenguaje, con el riesgo de que cada equipo la implemente distinto.
- **Consumer group:** se define en la configuración del consumer dentro de la app (`group.id`), junto con el manejo de offsets (commit automático o manual), los rebalanceos y los reintentos. Para evitar colisiones entre equipos, cada app tendría que construir su propio `group.id` con alguna convención (por ejemplo, `<servicio>-<namespace>`), y nada obliga a cumplirla.

**¿Quién lo mantiene?** El **equipo de la aplicación**. El equipo de plataforma solo administraría el clúster de Kafka (brokers, tópicos, `retention.ms`), pero las reglas de expiración y de aislamiento de consumidores quedarían repartidas en el código de los servicios. Además, cada actualización del SDK, cambio de broker o corrección se tendría que replicar en todas las apps.

Con Dapr, en cambio, el consumer group se declara en el Component (`consumerID: "{appID}-{namespace}"`) y el TTL se pide con un metadato estándar (`ttlInSeconds`). El sidecar aplica la lógica de forma uniforme y la app no conoce el SDK del broker.

### 2. El patrón sidecar como frontera de responsabilidad

El **valor** del TTL (120 s) es una decisión de negocio, mientras que el **mecanismo** que lo hace cumplir es infraestructura. La responsabilidad se puede repartir separando *qué* se decide de *cómo* se aplica:

| Responsabilidad | Equipo de producto | Equipo de plataforma |
|---|---|---|
| Definir el SLA de expiración ("una orden expira a los 2 min") | ✅ Dueño | Consultado |
| Declarar el valor del TTL | ✅ Lo declara en su configuración | Valida que esté dentro de los límites |
| Operar el broker, el Component y los sidecars | — | ✅ Dueño |
| Límites globales (TTL máximo, retención del stream, `consumerID` por namespace) | Consultado | ✅ Dueño |
| Observabilidad (mensajes expirados, lag, alertas) | Consume los dashboards | ✅ Los provee |

En la práctica, la propuesta es:

1. **El equipo de plataforma** publica Components estándar por namespace (broker, credenciales vía Secret Store, `consumerID` con `{namespace}`, `streamTTL` como retención de seguridad) y define guardrails: un TTL máximo permitido y que ningún mensaje viva sin expiración. Estos guardrails se pueden hacer cumplir con revisión en GitOps o con políticas de admisión (OPA/Kyverno).
2. **El equipo de producto** declara el TTL de su dominio como configuración, no como código: un valor en el ConfigMap o en el despliegue de su servicio, que la app envía como `ttlInSeconds` al publicar. Así puede cambiar el SLA sin pedirle nada a plataforma y sin tocar la lógica de negocio.
3. **El cambio pasa por un flujo GitOps** con revisores de ambos equipos (por ejemplo, con `CODEOWNERS`): producto aprueba el valor de negocio y plataforma aprueba que la configuración sea viable y segura.

Lo que se encontró en el laboratorio refuerza esta división: en Dapr el TTL es **por mensaje** y se indica al publicar (ver la pregunta 3), así que el valor queda naturalmente del lado del equipo de producto. El sidecar marca la frontera: el producto dice *cuánto* y la plataforma garantiza *que se cumpla*.

### 3. Limitaciones de la abstracción: ¿migración 100% transparente?

No. Dapr mantiene estable la **API** (`/v1.0/publish/{pubsub}/{topic}`) y el código no depende del SDK del broker, pero **la semántica y las capacidades de cada broker no son iguales**, y esas diferencias se notan en la configuración y a veces en el código.

**Limitación concreta comprobada en este laboratorio: el TTL a nivel de Component no existe para Redis.**

- Se agregó `ttlInSeconds: "120"` en el `spec.metadata` del Component `pubsub.redis`, esperando que todos los mensajes expiraran sin tocar el código.
- Resultado: los 3 mensajes publicados llegaron al subscriber **unos 173 s después** de publicarse, más que los 120 s del TTL. En Redis, el CloudEvent guardado en el stream `defaultorders` no tenía el atributo `expiration`. Dapr ignoró el campo sin mostrar error.
- Según la [documentación de Dapr](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-message-ttl/), el TTL es **por mensaje** y se indica al publicar (`?metadata.ttlInSeconds=120`). El componente Redis no tiene un campo de TTL por mensaje. Solo tiene `streamTTL` y `maxLenApprox`, que recortan el stream de forma aproximada y no garantizan la expiración de un mensaje.
- Por lo tanto, para cumplir el requisito de negocio con Redis **sí hay que modificar el código del publisher**, en contra de la premisa de "solo configuración".

**Diferencias entre brokers:**

| Aspecto | Redis Streams | Kafka |
|---|---|---|
| TTL nativo | No por mensaje. Dapr lo gestiona en el runtime a partir del metadato `ttlInSeconds`; `streamTTL` solo recorta el stream. | Retención por tópico (`retention.ms`), que borra segmentos completos, no mensajes individuales. El TTL por mensaje también lo gestiona Dapr. |
| Consumer group | Campo `consumerID`. | Campo `consumerGroup`, que tiene otro nombre y otras reglas. |
| Creación del tópico | El stream se crea solo al publicar. | El tópico debe existir o el clúster debe permitir la creación automática. |
| Retención e historial | Limitada por memoria; depende de `maxLenApprox`/`streamTTL`. | Persistencia en disco por días; permite volver a leer el historial. |

**Consecuencias para una migración Redis → Kafka:**

- Los Components cambian por completo: `type`, los nombres de la metadata (`redisHost` → `brokers`, `consumerID` → `consumerGroup`) y campos obligatorios nuevos como `authType`.
- Las garantías de orden, de retención y de reentrega cambian, y los consumidores pueden notarlo (por ejemplo, el orden por partición en Kafka).
- Lo que se configuraba en el broker (por ejemplo, `retention.ms`) queda fuera del Component y pasa a la administración del clúster de Kafka.
- Si un consumidor lee del broker **sin Dapr**, los mensajes expirados no se descartan solos, porque la expiración la aplica el sidecar.

**Conclusión:** Dapr hace que la migración sea **mayormente transparente para el código**, pero no para la configuración ni para la semántica. La abstracción cubre el mínimo común entre brokers, y cuando una característica no está en ese mínimo (como el TTL a nivel de componente en Redis), aparecen diferencias que hay que conocer, probar y documentar antes de migrar.
