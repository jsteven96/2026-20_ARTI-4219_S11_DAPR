# analisis-cloudevents.md

## Análisis de Trazabilidad (CloudEvents)

### 1. El atributo `id`: ¿quién lo genera y para qué sirve?

El UUID lo genera el sidecar darpd del publisher. Pasa en el momento en el que recibe la petición de publicación y arma el sobre CloudEvent. Dapr garantiza la entrega at-least-once: si el subscriber no responde ```200/SUCCESS```, el mensaje se reintenta y puede llegar más de una vez. Con el ```id``` como llave, el subscriber puede guardar los ids que ya procesó y descartar los duplicados. Así procesar el mismo evento dos veces no tiene efectos dobles, como cobrar dos veces una orden.

### 2. El atributo `source`: ¿qué valor tiene y de dónde lo toma Dapr?

El atributo ```source```contiene el valor ```order-publisher``` El valor lo toma del ```app-id``` del publisher, que viene de la anotación del Job.

### 3. El atributo `traceparent` y su relación con OpenTelemetry

El atributo ```traceparent``` aparece vacío en los CloudEvents revisados. De acuerdo a la documentación sale vacío por que el tracing de Dapr no está habilitado para las aplicaciones. Este atributo viene de la especificación ```W3C Trace Context```, el estándar del W3C para propagar contexto de trazas distribuídas entre servicios. Tiene un formato fijo de 4 partes:
```yaml
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```
Relación con OpenTelemetry

  OpenTelemetry usa W3C Trace Context como su formato de propagación por defecto. Dapr crea internamente los spans con OpenTelemetry y propaga
  el contexto con traceparent:
  - en llamadas HTTP y gRPC, como header
  - en Pub/Sub, dentro del sobre CloudEvent, porque un broker como Redis no transporta headers HTTP

  Así, una traza puede ir de publisher → daprd → Redis → daprd → subscriber sin que tu código haga nada, y la puedes ver como una sola línea de
  tiempo en Zipkin, Jaeger o cualquier backend compatible con OTLP.

### 4. Valor de negocio para auditoría: atributos más útiles del sobre CloudEvents


Para auditar los eventos de las últimas 24 horas, el sobre CloudEvents permite responder cuatro preguntas: **qué evento, quién lo envió, 
  cuándo y qué recorrido hizo**. Los atributos más útiles son:

  | Atributo | Utilidad para la auditoría |
  | --- | --- |
  | `id` | Identificador único de cada evento (UUID generado por el sidecar del publisher). Permite contar eventos sin duplicados, aunque haya reintentos por la entrega *at-least-once*, y referenciar un evento concreto. |
  | `source` | Indica **quién** originó el evento. Dapr lo toma del `app-id` del workload, no del código de la aplicación, por lo que es un dato confiable para atribuir responsabilidad. |
  | `time` | Indica **cuándo** se generó el evento. Es el atributo clave para filtrar la ventana de 24 horas. |
  | `type` | Indica **qué clase** de evento es, para agrupar o filtrar. En el lab es el genérico `com.dapr.event.sent`; en producción conviene un tipo de negocio como `com.empresa.order.created`. |
  | `traceparent` | Implementa W3C Trace Context y permite **correlacionar el evento de punta a punta** con la petición que lo originó y con lo que ocurrió después en otros servicios. |