# Sesión 11: Abstracción de Mensajería y Estandarización con Dapr

Laboratorio de Pub/Sub con **Dapr** sobre Kubernetes (Kind). La idea es que las aplicaciones publiquen y reciban eventos sin conocer el broker: el **sidecar** de Dapr se encarga de hablar con Redis y de envolver los mensajes en **CloudEvents 1.0**.

## Arquitectura de la PoC

| Componente | Rol en la PoC |
|---|---|
| **Dapr Control Plane** | Inyecta el sidecar (`daprd`) y distribuye los componentes declarativos |
| **Redis** | Broker de mensajería subyacente, invisible para el código de la aplicación |
| **Subscriber (Python/Flask)** | Microservicio que escucha el tópico `orders` |
| **Publisher (Python)** | Job que publica órdenes de prueba en `orders` |

Las apps solo hablan con su sidecar en `localhost:3500`. Cambiar de broker implica cambiar el Component YAML, no el código.

## Requisitos previos

```bash
kind version        # >= 0.20.0
kubectl version --client
helm version        # >= 3.x
docker info
python3 --version   # >= 3.9
```

## Estructura del repositorio

```
.
├── 01-arquitectura-sidecar/     # Fase 1: clúster Kind, Dapr y plano de control
├── 02-componentes-declarativos/ # Fase 2: Redis y Component order-pubsub
├── 03-aplicaciones/             # Fase 3: subscriber, publisher y CloudEvents
├── 04-reto-gobernanza/          # Fase 4: TTL y consumerID por namespace
└── evidencias/                  # Capturas de cada fase
```

Cada fase tiene su propio `README.md` con los pasos y su archivo de análisis (o reflexión) con las respuestas.

## Guías de la PoC

Sigue las guías en orden:

1. **[Fase 1: Arquitectura Sidecar, clúster y plano de control Dapr](01-arquitectura-sidecar/README.md)**
2. **[Fase 2: Componentes Declarativos, el contrato de mensajería](02-componentes-declarativos/README.md)**
3. **[Fase 3: Aplicaciones Pub/Sub, subscriber y publisher](03-aplicaciones/README.md)**
4. **[Fase 4: Reto de gobernanza, TTL y multi-tenant namespaces](04-reto-gobernanza/README.md)**

## Resultados principales

- **Fase 1:** Dapr 1.19.0 instalado en `dapr-system`. El webhook `dapr-sidecar-injector` inyecta `daprd` en los pods con `dapr.io/enabled: "true"`.
- **Fase 2:** Component `order-pubsub` (`pubsub.redis`) apuntando a `redis-master.redis.svc.cluster.local:6379`.
- **Fase 3:** el subscriber queda en `2/2` y recibe los tres CloudEvents del publisher con `source: order-publisher`.
- **Fase 4:**
  - **TTL:** en Redis, `ttlInSeconds` en el Component no tuvo efecto. El TTL funciona al publicar con `?metadata.ttlInSeconds=120`, y los mensajes publicados con TTL vencido no llegan al subscriber.
  - **consumerID:** `order-subscriber-{namespace}` crea el consumer group `order-subscriber-default.order-subscriber`.

## Limpieza

```bash
kubectl delete -f 03-aplicaciones/subscriber-deployment.yaml
kubectl delete -f 03-aplicaciones/publisher-job.yaml --ignore-not-found
kubectl delete -f 04-reto-gobernanza/pubsub-redis-challenge.yaml --ignore-not-found
kubectl delete -f 02-componentes-declarativos/pubsub-redis.yaml --ignore-not-found

kind delete cluster --name dapr-lab
```
