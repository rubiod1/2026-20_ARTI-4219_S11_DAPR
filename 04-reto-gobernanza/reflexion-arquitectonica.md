# reflexion-arquitectonica.md
# Completa este archivo con tus respuestas al Reto 3 de la Fase 4.

## Reto 3: Reflexión Arquitectónica

### 1. Sin Dapr: ¿dónde vive el código de TTL y consumer group?

Sin Dapr, el TTL estaría en el código de cada aplicación. El publisher tendría que calcular la fecha de expiración y ponerla en el mensaje, y el subscriber tendría que revisarla antes de procesarlo y descartar lo vencido. Con Kafka, por ejemplo, habría que usar el SDK, manejar los timestamps y escribir la lógica de descarte en cada servicio que consume. El consumer group también se configura en el código: cada servicio define su `group.id` al crear el consumidor de Kafka. Si dos equipos usan el mismo nombre de servicio, el problema de colisión aparece solo si alguien se acuerda de cambiar el `group.id`.

Quien mantiene esto es el equipo de la app, porque el código vive en su repositorio. Pero el equipo de plataforma queda con un problema: tiene que revisar que cada equipo haya implementado bien el TTL y los consumer groups, y no tiene forma de imponerlo en todos los servicios.

### 2. El patrón sidecar como frontera de responsabilidad

El valor de 120 segundos es una decisión de negocio: el equipo de producto sabe cuánto tiempo sigue siendo válida una orden. Pero en nuestro caso el número lo ponemos en el publisher, que es código de la aplicación. Por eso creo que la frontera debería quedar así: el equipo de producto define el SLA y lo documenta en la configuración del servicio, mientras que el equipo de plataforma se encarga de que el mecanismo funcione (en Dapr, el sidecar y el Component). Idealmente el TTL viviría en una política compartida, para que el valor se pueda cambiar sin tocar cada publisher.

En la práctica encontramos que el TTL no funcionaba en el Component de Redis y tuvimos que ponerlo en cada publicación. Eso muestra que la frontera no es tan limpia como parece: el equipo que cambia el broker tiene que conocer detalles del broker, y el equipo de producto termina tocando código de infraestructura.

### 3. Limitaciones de la abstracción: ¿migración 100% transparente?

No es 100 % transparente. Dapr unifica la API de Pub/Sub, pero cada broker tiene su propio comportamiento. En nuestra prueba, el `ttlInSeconds` del Component de Redis no tuvo efecto y el TTL solo funcionó al publicar con `metadata.ttlInSeconds`. Si migráramos a un broker con TTL a nivel de entidad, como Azure Service Bus, el TTL se configuraría en otro lugar, y el publisher tendría que cambiar o quedar con dos mecanismos.

Lo mismo pasa con los consumer groups. Redis agrupa los consumidores por stream y `consumerID`, mientras que Kafka los maneja por partición y `group.id`, y la forma de nombrar y balancear los grupos no es igual. El marcador `{namespace}` funcionó en Redis, pero no sé si el resultado sería idéntico en otro broker, y eso habría que probarlo antes de migrar.
