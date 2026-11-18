# DitStock — déploiement (CD Kubernetes)

Dépôt de **déploiement continu** de DitStock, application de gestion de stock pour les petits commerçants
et PME sénégalais. Il contient les **manifests Kubernetes** et le **pipeline CD** qui déploie les images
publiées sur Docker Hub par les deux dépôts de CI.

## Architecture multi-repositories

```
 ditstock_backend ──push──> CI : tests ─> image ─> Docker Hub  <compte>/ditstock-backend:sha-xxxxxxx
        │                                                 │
        └──── repository_dispatch « image-publiee » ──┐   │
                                                      ▼   ▼
 ditstock_deployment (ce dépôt) ──> CD : kubectl apply -f k8s/ ─> rollout status ─> vérifications
                                                      ▲   ▲
        ┌──── repository_dispatch « image-publiee » ──┘   │
        │                                                 │
 ditstock_frontend ──push──> CI : tests ─> image ─> Docker Hub  <compte>/ditstock-frontend:sha-xxxxxxx
```

| Dépôt | Contenu | Rôle |
|---|---|---|
| [ditstock_backend](https://github.com/Mj2k-styles/ditstock_backend) | API Flask, Dockerfile, CI, tests | CI : tests, image publiée sur Docker Hub, déclenche le CD |
| [ditstock_frontend](https://github.com/Mj2k-styles/ditstock_frontend) | React, Dockerfile, CI, tests | CI : tests, image publiée sur Docker Hub, déclenche le CD |
| **ditstock_deployment** | Manifests Kubernetes, pipeline CD | CD : déploie les images et vérifie le déploiement |

## Dans le cluster

```
                 NodePort 30080
 navigateur ──> Service frontend ──> Deployment frontend (2 pods Nginx + React)
                                            │ /api/
                                            ▼
                Service backend:5000 ──> Deployment backend (2 pods Flask/Gunicorn, initContainer migrations)
                                            │
                                            ▼
                Service db:5432 ──────> Deployment database (PostgreSQL 16) ──> PVC database-data (1 Gi)
```

| Fichier | Ressource |
|---|---|
| `k8s/namespace.yaml` | Namespace `ditstock` |
| `k8s/configmap.yaml` | Configuration de l'API (journaux, workers, données de démonstration) |
| `k8s/database-pvc.yaml` | Volume persistant de PostgreSQL |
| `k8s/database-deployment.yaml`, `database-service.yaml` | PostgreSQL 16 (stratégie `Recreate`, volume ReadWriteOnce) |
| `k8s/backend-deployment.yaml`, `backend-service.yaml` | API : 2 répliques, migrations en initContainer, sondes `/health/` |
| `k8s/frontend-deployment.yaml`, `frontend-service.yaml` | Frontend : 2 répliques, NodePort 30080 |
| `kind/cluster.yaml` | Cluster kind utilisé par le pipeline |

Les secrets (mot de passe PostgreSQL, clés Flask et JWT) ne sont **pas** dans le dépôt : le pipeline crée
le Secret `ditstock-secrets` avec des valeurs aléatoires s'il n'existe pas.

### Mise à jour sans interruption

Les Deployments `backend` et `frontend` utilisent le **rolling update** natif de Kubernetes :

- `replicas: 2`, `maxUnavailable: 0`, `maxSurge: 1` : un nouveau pod est créé, et un ancien n'est arrêté
  que lorsque le nouveau est **prêt** ;
- `readinessProbe` (`/health/` pour l'API, `/` pour le frontend) : un pod ne reçoit du trafic que s'il répond ;
- `preStop: sleep 5` : un pod qui s'arrête laisse au Service le temps de le retirer avant de couper.

### Persistance

PostgreSQL écrit dans le PVC `database-data` : supprimer le pod de la base ou redéployer l'application
ne fait pas perdre les données.

## Pipeline CD (GitHub Actions)

[`.github/workflows/cd.yml`](.github/workflows/cd.yml) se déclenche :

- **automatiquement après chaque CI réussie** : la CI du backend ou du frontend envoie l'événement
  `image-publiee` avec le tag de l'image qu'elle vient de publier ;
- à chaque push sur `main` de ce dépôt (modification des manifests) ;
- à la main (Actions → CD Kubernetes → Run workflow), en choisissant les tags.

Étapes :

1. **récupération des manifests** et **accès au cluster** : un cluster Kubernetes **kind** est créé dans le runner ;
2. **`kubectl apply -f`** des manifests, avec les images Docker Hub au tag demandé ;
3. **vérifications post-déploiement** : `kubectl rollout status` des trois Deployments, `/health/` de l'API,
   page d'accueil, API à travers Nginx ;
4. **persistance** : un produit est créé, le pod PostgreSQL est supprimé et l'API redéployée, puis on vérifie
   que le produit est toujours là ;
5. **mise à jour sans interruption** : `kubectl rollout restart` du backend et du frontend pendant qu'une
   boucle envoie des requêtes en continu ; le pipeline échoue si une seule requête échoue ;
6. **notification** : résumé du déploiement dans GitHub Actions (images, persistance, nombre de requêtes
   pendant la mise à jour, état des pods).

Pourquoi **GitHub Actions** : les trois dépôts sont sur GitHub, l'outil est intégré (pas de serveur Jenkins
à installer et maintenir), gratuit pour ce volume, et les dépôts se déclenchent entre eux nativement
(`repository_dispatch`).

Pourquoi **kind** : un vrai cluster Kubernetes (mêmes API, mêmes manifests) créé à chaque exécution, sans
serveur ni identifiants de cluster à gérer. Les mêmes manifests s'appliquent sur n'importe quel cluster
(k3s, minikube, cloud) : seule l'étape « Créer le cluster » change.

### Configuration

Dans **ce dépôt** (Settings → Secrets and variables → Actions) :

| Type | Nom | Valeur |
|---|---|---|
| Variable | `DOCKERHUB_USERNAME` | compte Docker Hub des images |
| Secret (facultatif) | `DEMO_PASSWORD` | mot de passe des comptes de démonstration (défaut `Demo12345!`) |

Dans **ditstock_backend** et **ditstock_frontend** : `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, la variable
`DEPLOYMENT_REPO` = `Mj2k-styles/ditstock_deployment` et le secret `DEPLOYMENT_TOKEN` (jeton GitHub
fine-grained, accès à ce dépôt, permission *Contents : Read and write*, exigée par `repository_dispatch`).

## Déployer à la main

Avec un cluster et `kubectl` configurés :

```bash
# Secrets (une seule fois)
kubectl apply -f k8s/namespace.yaml
kubectl -n ditstock create secret generic ditstock-secrets \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=SECRET_KEY="$(openssl rand -hex 32)" \
  --from-literal=JWT_SECRET_KEY="$(openssl rand -hex 32)" \
  --from-literal=DEMO_PASSWORD='Demo12345!'

# Manifests avec les images voulues
mkdir -p rendu
for f in k8s/*.yaml; do
  sed -e 's#DOCKERHUB_USERNAME/#moncompte/#' -e 's#:BACKEND_TAG#:latest#' -e 's#:FRONTEND_TAG#:latest#' \
      "$f" > "rendu/$(basename "$f")"
done
kubectl apply -f rendu/

# Vérifications
kubectl -n ditstock rollout status deployment/backend
kubectl -n ditstock get pods,svc,pvc
```

L'application est alors servie sur le port 30080 des nœuds (`kubectl -n ditstock port-forward svc/frontend 8080:80`
pour y accéder depuis son poste : http://localhost:8080).

Comptes de démonstration : `admin@ditstock.local`, `gestionnaire@ditstock.local`, `magasinier@ditstock.local`
(mot de passe `DEMO_PASSWORD`).

### Commandes utiles

```bash
kubectl -n ditstock get all,pvc                         # état
kubectl -n ditstock rollout history deployment/backend  # versions déployées
kubectl -n ditstock rollout undo deployment/backend     # revenir à la version précédente
kubectl -n ditstock logs deploy/backend -c migrations   # journal des migrations
kubectl -n ditstock logs -f deploy/backend              # journal de l'API
```
