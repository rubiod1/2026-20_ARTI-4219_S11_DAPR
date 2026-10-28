# analisis-control-plane.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 1.

## TODO 1: Análisis del Plano de Control

### 1. ¿Qué tipo de objeto de Kubernetes es el `dapr-sidecar-injector`?

Es un Deployment, que crea un ReplicaSet y este a su vez un Pod (`dapr-sidecar-injector-76d4cb6cf4-v4pmp`). Lo que lo hace especial es que actúa como webhook: cuando alguien crea un Pod con la anotación `dapr.io/enabled: "true"`, el API server le manda la petición al injector antes de crear el Pod, y el injector le agrega el contenedor `daprd`. Por eso se registra como MutatingWebhookConfiguration (en el clúster aparece como `dapr-sidecar-injector`). Tiene que ser "mutating" porque necesita modificar el Pod, no solo validarlo; si fuera de validación no podría agregar el contenedor.

### 2. Réplicas del `dapr-operator` y comparación con Istio

Hay una sola réplica (`dapr-operator-7c8bb589b7-qcsdq`). Para un laboratorio tiene sentido, y en producción se pueden subir réplicas. Respecto a Istio, la diferencia de peso viene de que Istio usa un proxy Envoy por Pod con toda la lógica de malla, mientras que Dapr deja en el sidecar solo lo que la app necesita (pub/sub, state, invocación) y el plano de control se queda con tareas pequeñas: distribuir componentes, inyectar sidecars, emitir certificados (sentry) y coordinar. Por eso el control plane de Dapr pesa bastante menos.

### 3. Flujo de inyección del sidecar

Cuando hago `kubectl apply` de un Deployment con `dapr.io/enabled: "true"`, el API server crea el ReplicaSet y, al intentar crear cada Pod, consulta a los webhooks registrados. El webhook de Dapr recibe el Pod, ve la anotación y le añade el contenedor `daprd` con sus variables (dirección de sentry, imagen, certificados montados desde `dapr-trust-bundle`). Luego el Pod se programa en un nodo, arranca los dos contenedores, `daprd` pide su identidad a sentry y, cuando se conecta, recibe los Components del operator. Al final, el Pod queda con 2/2 listo.

---

## TODO 2: Exploración de los CRDs de Dapr

### 4. CRDs instalados por Dapr

En mi clúster aparecen, entre otros:

- `components.dapr.io`: la configuración de los building blocks, por ejemplo el pubsub de Redis.
- `subscriptions.dapr.io`: declara a qué tópico se suscribe una app.
- `configurations.dapr.io`: ajustes generales del sidecar, como tracing o métricas.
- `resiliencies.dapr.io`: políticas de reintentos y timeouts.
- `httpendpoints.dapr.io`: endpoints HTTP externos a los que las apps pueden llamar a través de Dapr.

### 5. Relación entre `components.dapr.io` y el concepto de Building Block

El building block es la capacidad abstracta (por ejemplo, Pub/Sub), y el Component es la implementación concreta que la conecta con una tecnología (`pubsub.redis`). Así que el Component no es la abstracción en sí, sino la forma de enchufar una implementación a esa abstracción. Por eso la app solo pide "pubsub" con el nombre `order-pubsub`, y el Component decide si detrás hay Redis, Kafka u otro.
