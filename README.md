# TP Kubernetes — parties 1 à 5

Ce dépôt contient les manifests Kubernetes et le compte-rendu du TP, depuis les namespaces et NGINX jusqu'aux probes, au HPA et au débogage éphémère.

## Contenu

- [`compte-rendu.md`](./compte-rendu.md) : réponses, explications et commandes.
- [`TP/`](./TP/) : manifests Kubernetes organisés par service et par partie.
- [`img/`](./img/) : schémas fournis avec le sujet et captures réelles à ajouter après exécution.
- [`partie1.md`](./partie1.md) à [`partie5.md`](./partie5.md) : consignes originales.

## Pré-requis

Installer Docker Desktop, `kubectl` et Minikube, démarrer Docker Desktop puis lancer :

```powershell
minikube start --driver=docker
minikube status
kubectl get nodes
```

Les identifiants MySQL présents dans les manifests sont fictifs et réservés au TP. Un Secret encodé en base64 n'est pas chiffré par cet encodage.

## Déploiement

Créer d'abord les namespaces :

```powershell
kubectl apply -f TP/front/namespace.yml
kubectl apply -f TP/middle/namespace.yml
kubectl apply -f TP/database/namespace.yml
```

Pour suivre les différentes étapes, utiliser les commandes et l'ordre décrits dans [`compte-rendu.md`](./compte-rendu.md). En particulier, déployer MySQL avant WordPress, puis appliquer la ConfigMap et le Deployment NGINX configuré comme reverse proxy.

Pour la partie 5, appliquer les manifests finaux :

```powershell
kubectl apply -f TP/partie5/namespace.yml
kubectl apply -f TP/partie5/app-deployment.yml
kubectl apply -f TP/partie5/app-service.yml
kubectl port-forward -n prod-app svc/hello-kubernetes 8080:8080
```

Ouvrir ensuite `http://localhost:8080`.
