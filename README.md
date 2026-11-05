# gitops-argocd-apps

Capa de **Argo CD** del circuito **QuickTask GitOps**: define _quién_ reconcilia y
_cómo se descubren_ las aplicaciones. Combina dos patrones clásicos de Argo CD:
**App of Apps** + **ApplicationSet**.

> - [`helm-chart-generic-app`](https://github.com/slashispy/helm-chart-generic-app) → **cómo** se despliega (templates).
> - [`gitops-app-values`](https://github.com/slashispy/gitops-app-values) → **qué** se despliega y **con qué values**.
> - **`gitops-argocd-apps`** (este repo) → **quién** reconcilia (Argo CD).

## Modelo

```
 kubectl apply -f app-of-apps.yaml
        │
        ▼
 Application "root" (App of Apps)
        │  sincroniza este repo: projects/ + apps/
        ├─▶ AppProject "quicktask"          (projects/quicktask-project.yaml)
        └─▶ ApplicationSet "quicktask-apps" (apps/applicationset.yaml)
                   │  git files generator sobre gitops-app-values/apps/*/metadata.yaml
                   ▼
            Application "quicktask"  (una por cada metadata.yaml)
                   │  sources: chart (helm-chart-generic-app) + values (gitops-app-values)
                   ▼
            namespace quicktask: Deployment + Service + CM/Secret + MongoDB
```

### 1. App of Apps (`app-of-apps.yaml`)

La **única** pieza que se aplica a mano. Una `Application` raíz llamada `root` que
apunta a este repo y sincroniza, con un _directory generator_ recursivo, los
manifiestos de `projects/` y `apps/`. A partir de ahí, todo lo demás lo gestiona
Argo CD desde Git.

### 2. AppProject (`projects/quicktask-project.yaml`)

Frontera de seguridad: restringe los repos usables como source, los destinos
(cluster + namespaces `quicktask`/`argocd`) y los tipos de recurso permitidos.
Todas las Applications del circuito pertenecen a este proyecto.

### 3. ApplicationSet (`apps/applicationset.yaml`)

El motor de descubrimiento. Un **git files generator** recorre
`gitops-app-values/apps/*/metadata.yaml` y, por cada archivo, genera una
`Application`. Usa **multiple sources** para combinar dos repos distintos:

| Source | Rol |
|---|---|
| `helm-chart-generic-app` | El chart (templates). Toma los values vía `$values`. |
| `gitops-app-values` (`ref: values`) | Expone el `values.yaml` de la app; no despliega nada por sí mismo. |

Las claves del `metadata.yaml` (`name`, `namespace`, `chart.*`) se interpolan en la
plantilla con Go templating (`goTemplate: true`).

## Uso

```bash
# Requiere Argo CD instalado en el namespace argocd.
kubectl apply -f app-of-apps.yaml

# Ver el estado
kubectl -n argocd get applications
argocd app list           # si tenés la CLI de argocd
```

> En el circuito completo esto lo hace el script `03-bootstrap-apps.sh` de
> [`kind-gitops-utility-belt`](https://github.com/slashispy/kind-gitops-utility-belt).

## Verificación local (sin cluster)

```bash
# Validación de sintaxis de los manifiestos (client-side)
kubectl apply --dry-run=client -f app-of-apps.yaml
kubectl apply --dry-run=client -f projects/quicktask-project.yaml
kubectl apply --dry-run=client -f apps/applicationset.yaml
```

> Nota: las CRDs `Application`/`ApplicationSet`/`AppProject` las aporta Argo CD. La
> validación contra el API server real (con esas CRDs instaladas) se hace end-to-end
> en el repo de tooling, al levantar el cluster KIND.

## Agregar una nueva app

No se toca este repo: basta con añadir `apps/<nueva-app>/{metadata.yaml,values.yaml}`
en [`gitops-app-values`](https://github.com/slashispy/gitops-app-values). El
ApplicationSet la detecta y crea su `Application` automáticamente.

## Parte del circuito QuickTask GitOps

1. [`quicktask`](https://github.com/slashispy/quicktask) — la aplicación (API Go + MongoDB).
2. [`helm-chart-generic-app`](https://github.com/slashispy/helm-chart-generic-app) — chart genérico reutilizable.
3. [`gitops-app-values`](https://github.com/slashispy/gitops-app-values) — values por app.
4. **`gitops-argocd-apps`** — App of Apps + ApplicationSet. ← estás aquí
5. [`kind-gitops-utility-belt`](https://github.com/slashispy/kind-gitops-utility-belt) — bootstrap del circuito en KIND.
