# gitops-config

Repo de configuracion de ArgoCD, separado del codigo de cada
microservicio. Aqui vive el registro de que se despliega y desde donde;
cada microservicio (por ejemplo
[hello-world-argocd](https://github.com/juanfranciscofernandezherreros/hello-world-argocd))
solo aporta su codigo y sus manifiestos `k8s/`, sin saber nada de Argo.

- `applications/` — un `Application` de ArgoCD por microservicio.

## Instalar ArgoCD en el cluster

```powershell
kind create cluster --name argocd-demo
kubectl config use-context kind-argocd-demo

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

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

Con `syncPolicy.automated`, cualquier cambio en el repo del microservicio
(carpeta `k8s/`) se aplica solo, sin `argocd app sync` manual.

## Anadir un microservicio nuevo

Copia `applications/hello-world.yaml`, cambia `metadata.name`,
`spec.source.repoURL` y `spec.destination.namespace`, y aplica el fichero.
Cuando haya varios servicios con el mismo patron, vale la pena sustituir
esta carpeta por un unico `ApplicationSet` (ver
`crud-automation/argocd/applicationset.yaml` como referencia).
