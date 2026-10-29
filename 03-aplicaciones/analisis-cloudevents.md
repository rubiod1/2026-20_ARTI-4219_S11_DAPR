# analisis-cloudevents.md
# Completa este archivo con tus respuestas al TODO 4 de la Fase 3.

## TODO 4: Análisis de Trazabilidad (CloudEvents)

### 1. El atributo `id`: ¿quién lo genera y para qué sirve?

El `id` lo genera el sidecar de Dapr del publisher en el momento en que recibe la petición de publicación, no la app. En los logs del subscriber se ve, por ejemplo, `4e42e45e-af1a-48e8-a9ee-5e9ceb1022bf` para la orden 1003. Cada mensaje tiene su propio UUID, así que sirve para identificar un evento concreto. Es importante para la idempotencia: si el broker reenvía el mismo mensaje (por un reintento, por ejemplo), el subscriber puede guardar los `id` que ya procesó y descartar los repetidos, para no procesar dos veces la misma orden.

### 2. El atributo `source`: ¿qué valor tiene y de dónde lo toma Dapr?

En los logs aparece `source: order-publisher`. Ese valor sale de la anotación `dapr.io/app-id: "order-publisher"` que pusimos en el Job del publisher. Dapr no lo inventa: toma el identificador de la aplicación que publica, así que el subscriber sabe qué servicio originó el evento sin que el código del publisher lo indique.

### 3. El atributo `traceparent` y su relación con OpenTelemetry

`traceparent` viene del estándar W3C Trace Context. Tiene el formato `versión-traceId-spanId-flags` y sirve para encadenar las trazas de una misma operación a través de varios servicios. OpenTelemetry usa ese mismo estándar para propagar el contexto. En nuestro caso el campo llega vacío (`traceparent :` en los logs) porque no configuramos tracing en Dapr. Si lo configuráramos con un exportador (Zipkin, OTLP...), Dapr llenaría el campo y podríamos seguir una orden desde el publisher, pasando por Redis, hasta el subscriber, en una sola traza.

### 4. Valor de negocio para auditoría: atributos más útiles del sobre CloudEvents

Para rastrear eventos en las últimas 24 horas, los atributos más útiles serían:

- **`id`:** identifica cada evento de forma única, así que sirve para buscar un mensaje concreto y detectar duplicados.
- **`source`:** indica qué servicio publicó el evento, lo que permite saber de qué equipo o componente viene.
- **`type`:** dice qué tipo de evento es, para filtrar por ejemplo solo eventos de órdenes.
- **`traceparent`:** conecta el evento con el resto de la operación en la traza, útil para reconstruir qué pasó después.

Además, para auditoría completa haría falta el `data` (con el `orderId` y el `status`), porque es lo que dice qué cambió. Los atributos del sobre sirven para localizar el evento, y el `data` para entender su contenido.
