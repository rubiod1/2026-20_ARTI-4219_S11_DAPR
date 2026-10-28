# analisis-abstraccion.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 2.

## TODO 2: Análisis de la Abstracción

### 1. Migración de broker: ¿qué archivos cambiarían?

Si migráramos de Redis a Kafka, el cambio principal sería el `pubsub-redis.yaml`: cambiar `spec.type` a `pubsub.kafka` y ajustar la metadata (`brokers`, `authType`, etc.) según la configuración de Kafka. También habría que tocar el manifiesto de infraestructura para desplegar Kafka en lugar de Redis, y el `pubsub-redis-challenge.yaml` si ya lo tenemos en uso.

El código Python del subscriber y del publisher no cambiaría. Ambos usan `order-pubsub` como nombre del componente y el endpoint `localhost:3500/v1.0/publish/...`, así que no saben qué broker hay detrás. Lo único que cambia de cara a la app es que el nombre del componente se mantenga igual.

### 2. Inventario de 5 brokers soportados por Dapr

| Broker | `spec.type` en el YAML |
|---|---|
| Redis | `pubsub.redis` |
| Apache Kafka | `pubsub.kafka` |
| RabbitMQ | `pubsub.rabbitmq` |
| Azure Service Bus (Topics) | `pubsub.azure.servicebus.topics` |
| GCP Pub/Sub | `pubsub.gcp.pubsub` |

En todos los casos lo que cambia es `spec.type`, y después la metadata específica de cada broker (por ejemplo, `brokers` en Kafka o `connectionString` en Azure Service Bus).

### 3. Manejo seguro de secretos en producción

Dapr tiene el building block de **Secret Store**. En lugar de escribir la contraseña en el YAML, el Component la pide a un almacén de secretos (Kubernetes Secrets, Azure Key Vault, HashiCorp Vault, etc.) mediante `secretKeyRef`. Así el YAML que va a Git solo tiene una referencia al nombre del secreto, no el valor. En el taller el `redisPassword: ""` sirve porque Redis no tiene autenticación, pero en un entorno real esa línea debería apuntar a un secreto.
