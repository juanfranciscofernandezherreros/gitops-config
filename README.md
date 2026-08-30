# gitops-config

Repo unico de configuracion de despliegue: manifiestos Kubernetes y
`Application` de ArgoCD, separados del codigo de cada microservicio. Los
repos de codigo (por ejemplo
[hello-world-argocd](https://github.com/juanfranciscofernandezherreros/hello-world-argocd))
solo tienen la aplicacion, sin nada de Kubernetes ni de Argo.

- `applications/` — un `Application` de ArgoCD por microservicio.
- `manifests/<servicio>/` — Namespace, Deployment, Service de ese
  microservicio.

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

## Anadir un microservicio nuevo

1. Crea `manifests/<servicio>/` con sus manifiestos.
2. Copia `applications/hello-world.yaml`, cambia `metadata.name`, `path` y
   `destination.namespace`.
3. `kubectl apply -f applications/<servicio>.yaml`.

Cuando haya varios servicios con el mismo patron, vale la pena sustituir
`applications/` por un unico `ApplicationSet` que genere las Applications
a partir de `manifests/*` (ver
`crud-automation/argocd/applicationset.yaml` como referencia).

## Prueba OOM de GodbyeMicroservices

El Deployment de `godbyemicroservices` limita el contenedor a `100Mi` y activa
la carga de memoria durante el arranque. La JVM de la imagen limita su heap a
64 MiB para lanzar `java.lang.OutOfMemoryError: Java heap space` antes de que el
kernel termine el contenedor como `OOMKilled`, ya que el proceso tambien usa
memoria nativa y metaspace fuera del heap.
