# analisis-abstraccion.md

## Análisis de la Abstracción

### 1. Migración de broker: ¿qué archivos cambiarían?

El código Python no cambia. Lo que cambia es la configuración e infraestructura. Los archivos que si cambian son los siguientes:

1. 02-componentes-declarativos/pubsub-redis.yaml: es el cambio principal. Se cambia el type y se reemplaza la metadata específica de Redis por
  la de Kafka. El metadata.name se mantiene igual:

  ```yaml
  apiVersion: dapr.io/v1alpha1
  kind: Component
  metadata:
    name: order-pubsub          # ← igual que antes, es el contrato con las apps
    namespace: default
  spec:
    type: pubsub.kafka          # ← antes: pubsub.redis
    version: v1
    metadata:
      - name: brokers           # ← antes: redisHost
        value: "kafka.kafka.svc.cluster.local:9092"
      - name: consumerGroup
        value: "order-subscriber"
      - name: authType          # ← obligatorio en el componente de Kafka
        value: "none"
  ```
  2. 04-reto-gobernanza/pubsub-redis-challenge.yaml: tiene la misma metadata de Redis (redisHost, redisPassword), así que también hay que adaptarlo.

### 2. Inventario de 5 brokers soportados por Dapr

  | Broker | `spec.type` en el YAML | Metadata de conexión principal |
  |---|---|---|
  | Redis Streams | `pubsub.redis` | `redisHost`, `redisPassword` |
  | Apache Kafka | `pubsub.kafka` | `brokers`, `authType`, `consumerGroup` |
  | RabbitMQ | `pubsub.rabbitmq` | `connectionString` (ej: `amqp://user:pass@host:5672`) |
  | Azure Service Bus (Topics) | `pubsub.azure.servicebus.topics` | `connectionString` o `namespaceName` (con Managed Identity) |
  | AWS SNS/SQS | `pubsub.aws.snssqs` | `region`, `accessKey`, `secretKey` |

  ¿Qué campo cambia? El campo que indica el broker es ```spec.type```, con la forma ```pubsub.<implementación>```. Le dice al sidecar daprd qué
  implementación del building block Pub/Sub cargar.

  Junto con ```spec.type``` también cambia ```spec.metadata```, porque cada broker tiene sus propios parámetros de conexión. Lo que no cambia es
  ```metadata.name``` (order-pubsub). Ese nombre es el que usan las apps, y por eso el código queda igual.

### 3. Manejo seguro de secretos en producción

 El manejo sugerido en la documentación es a través del componente Secret Stores + secretKeyRef

  En lugar de escribir el valor en el YAML, el Component referencia un secreto guardado en un Secret Store. Un Secret Store es otro building block de Dapr, también configurado como Component.

  1. Se crea el secreto fuera de Git:
        ```bash
        kubectl create secret generic redis-secret \
        --from-literal=redis-password='S3cr3t0!' \
        -n default
        ```

  2. El Component lo referencia con secretKeyRef:
        ```yaml
        apiVersion: dapr.io/v1alpha1
        kind: Component
        metadata:
            name: order-pubsub
            namespace: default
        spec:
            type: pubsub.redis
            version: v1
            metadata:
            - name: redisHost
                value: "redis-master.redis.svc.cluster.local:6379"
            - name: redisPassword
                secretKeyRef:           # ← en vez de "value"
                name: redis-secret    # nombre del secreto
                key: redis-password   # llave dentro del secreto
        auth:
            secretStore: kubernetes     # ← de qué Secret Store leerlo
        ```
  Cuando daprd carga el Component, resuelve la referencia contra el Secret Store y recibe el valor real. En el repositorio nunca aparece la
  contraseña.
