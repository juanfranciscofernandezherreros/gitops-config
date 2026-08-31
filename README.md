# gitops-config

Repo unico de configuracion de despliegue: manifiestos Kubernetes y
`Application` de ArgoCD, separados del codigo de cada microservicio. Los
repos de codigo (por ejemplo
[hello-world-argocd](https://github.com/juanfranciscofernandezherreros/hello-world-argocd))
solo tienen la aplicacion, sin nada de Kubernetes ni de Argo.

- `applications/` — un `Application` de ArgoCD por microservicio.
- `manifests/<servicio>/` — Namespace, Deployment, Service de ese
  microservicio.

`applications/devtools-kafka.yaml` despliega en el namespace `devtools-kafka`
la traduccion Kubernetes del stack `rft-devtools-kafka-cucumber-main/local-dev`:
ZooKeeper, Kafka, Confluent Schema Registry, Kafka REST Proxy 7.4.4, Kafka UI
y Redpanda Console. Las dos interfaces visuales usan el mismo broker y Schema
Registry; Kafka UI tambien muestra la conexion con ZooKeeper. Un hook
PostSync registra automaticamente los contratos `test-topic-key` y
`test-topic-value` que utilizan la aplicacion demo y sus samples.

La arquitectura, las conexiones, el funcionamiento de cada componente y las
tareas de operacion se explican en
[Plataforma Kafka local administrada con Argo CD](docs/plataforma-kafka-con-argocd.md).

## Instalar ArgoCD en el cluster

```powershell
kind create cluster --name hello-world
kubectl config use-context kind-hello-world

kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

`--server-side` es necesario: con `apply` normal, el CRD de `ApplicationSet`
supera el limite de 262144 bytes en la anotacion `last-applied-configuration`
y falla.

Acceso a la UI/CLI:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8081:443
```

En otra terminal, password inicial (usuario `admin`):

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
argocd login localhost:8081 --username admin --insecure
```

## Registrar un microservicio

```powershell
kubectl apply -f applications/hello-world.yaml
```

Con `syncPolicy.automated`, cualquier cambio en `manifests/hello-world/`
se aplica solo, sin `argocd app sync` manual.

## Registrar la plataforma Kafka compartida

```powershell
kubectl apply -f applications/devtools-kafka.yaml
kubectl get application devtools-kafka -n argocd
kubectl get pods,svc -n devtools-kafka
```

Los microservicios del cluster se conectan al broker interno mediante
`kafka.devtools-kafka.svc.cluster.local:29092`. Schema Registry queda en
`http://schema-registry.devtools-kafka.svc.cluster.local:8081` y REST Proxy en
`http://rest-proxy.devtools-kafka.svc.cluster.local:8082`.

Acceso temporal desde el equipo local (un comando por terminal):

```powershell
kubectl port-forward -n devtools-kafka svc/kafka 9092:9092
kubectl port-forward -n devtools-kafka svc/schema-registry 18081:8081
kubectl port-forward -n devtools-kafka svc/rest-proxy 18082:8082
kubectl port-forward -n devtools-kafka svc/kafka-ui 18080:8080
kubectl port-forward -n devtools-kafka svc/redpanda-console 18083:8080
```

Con esos tuneles, Schema Registry responde en `http://localhost:18081` y
REST Proxy en `http://localhost:18082`. Kafka UI queda en
`http://localhost:18080` y Redpanda Console en `http://localhost:18083`. El
listener externo de Kafka se anuncia como `localhost:9092`, igual que en el
Compose original.

## Anadir un microservicio nuevo

1. Crea `manifests/<servicio>/` con sus manifiestos.
2. Copia `applications/hello-world.yaml`, cambia `metadata.name`, `path` y
   `destination.namespace`.
3. `kubectl apply -f applications/<servicio>.yaml`.

Cuando haya varios servicios con el mismo patron, vale la pena sustituir
`applications/` por un unico `ApplicationSet` que genere las Applications
a partir de `manifests/*` (ver
`crud-automation/argocd/applicationset.yaml` como referencia).
