# TP Kubernetes — compte-rendu des parties 1 à 5

> Les valeurs de mot de passe de ces manifests sont des valeurs fictives pour le TP. Un Secret Kubernetes encodé en base64 n'est pas chiffré par ce seul encodage : ne pas réutiliser ces valeurs en production.

## Préparation et conventions

Le dossier `TP` contient les manifests à appliquer. Les ressources namespacées indiquent explicitement leur namespace dans leur metadata. Le PV WordPress est une ressource globale au cluster et n'a donc pas de namespace.

Commandes de préparation et de contrôle :

```powershell
docker info
minikube start --driver=docker
minikube status
kubectl version
kubectl get nodes
```

Le cluster et Docker doivent être démarrés avant l'application des manifests. Pour supprimer une ressource créée par un manifest, utiliser `kubectl delete -f <fichier.yml>`; pour supprimer un namespace de test, `kubectl delete namespace <nom>`.

## Partie 1 — namespaces et reverse proxy NGINX

### Déploiement

Les trois namespaces sont définis dans `TP/front/namespace.yml`, `TP/middle/namespace.yml` et `TP/database/namespace.yml`. Les ressources front sont dans `TP/front`. Le Service `nginx` reçoit sur le port TCP 8383 et sélectionne les pods `app: reverse-proxy` sur le port 80. Le Deployment utilise deux réplicas de `nginx:1.27.1-alpine`.

```powershell
kubectl get namespaces
kubectl apply -f TP/front/namespace.yml
kubectl apply -f TP/middle/namespace.yml
kubectl apply -f TP/database/namespace.yml
kubectl describe namespace front
kubectl get namespaces
kubectl apply -f TP/front/nginx-deployment-partie1.yml
kubectl apply -f TP/front/nginx-service.yml
kubectl get events -n front --sort-by=.metadata.creationTimestamp
kubectl get pods -n front
kubectl describe pods -n front
kubectl port-forward -n front svc/nginx 8080:8383
```

Les deux commandes pour consulter les événements et l'état de démarrage sont `kubectl get events -n front` et `kubectl describe pods -n front` (ou `kubectl get pods -n front` pour une vue synthétique). Ouvrir ensuite `http://localhost:8080`.

Dans cette première étape, le manifeste `nginx-deployment-partie1.yml` laisse NGINX servir sa page par défaut. Pour la partie 2, le Deployment final `nginx-deployment.yml` ajoute le montage de la ConfigMap et le reverse proxy WordPress; il remplace le Deployment de l'étape 1 :

```powershell
kubectl apply -f TP/front/nginx-configmap.yml
kubectl apply -f TP/front/nginx-deployment.yml
```

### Questions

- **Pointillés violets (schéma Q7)** : ils regroupent les pods gérés ensemble par le contrôleur de déploiement (Deployment/ReplicaSet); ce n'est pas un réseau séparé.
- **Pointillés rouges (schéma Q8)** : ils délimitent le namespace logique dans lequel les ressources de l'application sont isolées. Les namespaces ne constituent pas, à eux seuls, une barrière réseau.

### Captures à fournir

| No | Preuve attendue | Nom du fichier prévu |
|---|---|---|
| 1 | `kubectl get namespaces` avant la création des namespaces du TP | `img/01-namespaces-avant.png` |
| 2 | `kubectl get namespaces` après leur création | `img/02-namespaces-apres.png` |
| 3 | Page `Welcome to nginx!` sur `http://localhost:8080`, barre d'adresse visible | `img/03-nginx-localhost.png` |
| 4 | Variables MySQL affichées depuis le pod MySQL | `img/04-mysql-environnement.png` |
| 5 | Variables WordPress affichées depuis le pod WordPress | `img/05-wordpress-environnement.png` |
| 6 | Page d'installation WordPress sur `http://localhost:8080`, barre d'adresse visible | `img/06-wordpress-installation.png` |
| 7 | CronJob créé et état initial de l'HPA | `img/07-cronjob-hpa.png` |
| 8 | Charge BusyBox et `kubectl get hpa -w` visibles côte à côte; l'HPA doit montrer les réplicas supplémentaires s'ils sont déclenchés | `img/08-hpa-charge.png` |
| 9 | Pods, PVC et Services prêts après le déploiement complet de WordPress | `img/09-ressources-wordpress.png` |
| 10 | Page Hello Kubernetes sur `http://localhost:8080`, nom du pod et barre d'adresse visibles | `img/10-hello-kubernetes.png` |
| 11 | Événements de la readiness probe dans `kubectl describe pod` | `img/11-probes-evenements.png` |
| 12 | Remplacement des pods observé avec `kubectl get pods -n prod-app -w` | `img/12-rolling-update.png` |
| 13 | État `Terminating` pendant l'exécution du hook `preStop` | `img/13-prestop-terminating.png` |
| 14 | Échec attendu de `curl` absent de l'image applicative | `img/14-curl-absent.png` |
| 15 | Réponse `wget` depuis le conteneur de débogage éphémère | `img/15-debug-wget.png` |

Pour les captures 3, 6 et 10, laisser tourner la commande `kubectl port-forward` correspondante et conserver la barre d'adresse visible. Les images actuelles de `img/` sont les schémas fournis par le sujet, pas des preuves d'exécution. Ne marquer une capture comme obtenue qu'après l'avoir enregistrée depuis une exécution réelle.

## Partie 2 — MySQL, WordPress et reverse proxy

### MySQL

MySQL est déployé dans `database`. Le StatefulSet possède un volume persistant créé par son `volumeClaimTemplates` (1 Gio); le Service headless `mysql` assure son identité réseau stable. Les mêmes identifiants fictifs sont définis dans le Secret `mysql-secret` de `database` et dans une copie homonyme du Secret de `middle`. La ConfigMap est également copiée dans `middle` : une ConfigMap ou un Secret ne peut être référencé directement que dans son propre namespace.

```powershell
kubectl apply -f TP/database/mysql-secret.yml
kubectl apply -f TP/database/mysql-configmap.yml
kubectl apply -f TP/database/mysql-service.yml
kubectl apply -f TP/database/mysql-statefulset.yml
kubectl get pods,pvc,svc -n database
kubectl exec -n database statefulset/mysql -- printenv MYSQL_DATABASE MYSQL_USER MYSQL_PASSWORD MYSQL_ROOT_PASSWORD
```

### WordPress

Le PV `wordpress-pv` utilise le chemin demandé sur le nœud Minikube. Le PVC et WordPress sont dans `middle`. Le nom DNS complet de MySQL est `mysql.database.svc.cluster.local:3306`.

```powershell
kubectl apply -f TP/middle/mysql-secret.yml
kubectl apply -f TP/middle/mysql-configmap.yml
kubectl apply -f TP/middle/wordpress-pv.yml
kubectl apply -f TP/middle/wordpress-pvc.yml
kubectl apply -f TP/middle/wordpress-deployment.yml
kubectl apply -f TP/middle/wordpress-service.yml
kubectl get pods,pvc,svc -n middle
kubectl exec -n middle deployment/wordpress -- printenv WORDPRESS_DB_USER WORDPRESS_DB_NAME WORDPRESS_DB_HOST WORDPRESS_DB_PASSWORD
```

Les étapes WordPress peuvent être appliquées après MySQL. Le PVC doit être `Bound` avant que le pod WordPress puisse démarrer correctement.

### NGINX comme reverse proxy

La ConfigMap NGINX résout WordPress via le DNS inter-namespace `wordpress.middle.svc.cluster.local`. Le Service NGINX est publié localement avec un port-forward :

```powershell
kubectl apply -f TP/front/nginx-configmap.yml
kubectl apply -f TP/front/nginx-deployment.yml
kubectl apply -f TP/front/nginx-service.yml
kubectl port-forward -n front svc/nginx 8080:8383
```

Ouvrir `http://localhost:8080` et compléter l'installation initiale de WordPress si elle s'affiche.

### Vérifications

- **Les ressources sont-elles dans le bon namespace ?** `kubectl get all -A` affiche les ressources namespacées avec leur namespace; `kubectl get pods,svc,configmap,secret -n <namespace>` permet de vérifier un namespace précis. `kubectl get pv` affiche les PV, qui sont globaux au cluster.
- **Pourquoi recopier le Secret et la ConfigMap dans `middle` ?** Un pod ne peut référencer que des Secrets et ConfigMaps de son propre namespace. MySQL consomme leurs objets dans `database`; WordPress consomme donc leurs équivalents dans `middle`.
- **Les pods sont-ils démarrés ?** `kubectl get pods -n <namespace>`; attendre `READY` et `STATUS=Running`.
- **Comment diagnostiquer un pod en erreur ?** `kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp`, `kubectl describe pod <pod> -n <namespace>` et `kubectl logs <pod> -n <namespace>`. Pour le conteneur précédent après un redémarrage : `kubectl logs <pod> -n <namespace> --previous`.

### Captures à fournir

1. `kubectl exec` affichant les quatre variables MySQL.
2. `kubectl exec` affichant les quatre variables WordPress.
3. La page d'installation de WordPress obtenue à travers le port-forward NGINX.

## Partie 3 — CronJob et HPA

### CronJob

`TP/cronjob.yml` crée un Job toutes les cinq minutes et exécute `date` :

```powershell
kubectl apply -f TP/cronjob.yml
kubectl get cronjobs,jobs,pods
kubectl logs job/<nom-du-job>
```

### HPA

`TP/application.yml` définit `mon-application` avec requests et limits CPU/mémoire; `TP/hpa-application.yml` cible le Deployment, maintient entre un et trois pods et vise 30 % de CPU moyen par rapport aux requests. Le résultat attendu est l'augmentation du nombre de réplicas lorsque la charge dépasse la cible, puis leur retour vers le minimum lorsque la charge baisse. La création effective dépend des métriques disponibles et de la charge réellement générée.

```powershell
minikube addons enable metrics-server
kubectl apply -f TP/application.yml
kubectl expose deployment mon-application --port=80 --type=LoadBalancer
kubectl apply -f TP/hpa-application.yml
kubectl get hpa
kubectl describe hpa mon-application
kubectl get pods -w
```

Les colonnes usuelles de `kubectl get hpa` sont : `NAME` (HPA), `REFERENCE` (cible), `TARGETS` (métrique actuelle/cible), `MINPODS`, `MAXPODS`, `REPLICAS` (réplicas demandés actuellement) et `AGE`. `TARGETS` peut afficher `<unknown>` tant que Metrics Server n'a pas encore publié de mesures.

Pour générer du trafic, ouvrir un shell BusyBox dans le cluster puis lancer les requêtes vers le Service :

```powershell
kubectl run -it --rm simulation-utilisateur --image=busybox:1.36 --restart=Never -- /bin/sh
```

Dans le shell :

```sh
while true; do wget -q -O- http://mon-application.default.svc.cluster.local; done
```

Un simple flux de requêtes HTTP ne garantit pas de consommer assez de CPU pour franchir le seuil. Contrôler `kubectl get hpa -w`, `kubectl top pods` et les événements; le délai de stabilisation de l'HPA est normal.

## Partie 4 — QCM Kubernetes

| Question | Réponse | Justification |
|---|---|---|
| 1 | B | Un Pod regroupe un ou plusieurs conteneurs étroitement liés partageant réseau et volumes. |
| 2 | B | `kubectl create namespace <nom>` est la commande complète. |
| 3 | A | Le kubelet communique avec l'API Server pour signaler et maintenir l'état des pods de son nœud. |
| 4 | B | Une ConfigMap contient des paramètres non sensibles. |
| 5 | B | Un CronJob déclenche des Jobs selon un calendrier. |
| 6 | B | Un Deployment gère les ReplicaSets et les mises à jour des pods. |
| 7 | B | kube-proxy met en œuvre les règles réseau des Services sur les nœuds. |
| 8 | A | `kubectl logs <pod-name>` lit les journaux du conteneur. |
| 9 | A | Un contrôleur tel qu'un Deployment recrée un pod pour rétablir l'état désiré. |
| 10 | C | Un Secret sert aux données sensibles; son encodage base64 n'est pas un chiffrement. |
| 11 | B | Un namespace fournit un cloisonnement logique des noms et des ressources namespacées. |
| 12 | C | Un PersistentVolumeClaim demande une capacité et des modes d'accès. |
| 13 | C | L'API Server expose l'API Kubernetes et constitue le point d'entrée de contrôle. |
| 14 | B | RBAC et NetworkPolicies contribuent aux contrôles d'accès et à l'isolation; les PSP sont obsolètes dans les versions récentes, remplacées notamment par Pod Security Admission. |
| 15 | A | Les requests influencent le placement; des limites CPU trop basses peuvent provoquer du throttling. |
| Bonus | C | Les Secrets sont encodés en base64 dans etcd et ne sont pas chiffrés par défaut sans configuration additionnelle. |

## Partie 5 — application « production-ready »

Les manifests sont dans `TP/partie5`. Le Deployment `hello-kubernetes` démarre trois réplicas et son Service ClusterIP écoute sur 8080.

```powershell
kubectl apply -f TP/partie5/namespace.yml
kubectl apply -f TP/partie5/app-deployment-initial.yml
kubectl apply -f TP/partie5/app-service.yml
kubectl get pods,svc -n prod-app
kubectl port-forward -n prod-app svc/hello-kubernetes 8080:8080
```

Ouvrir `http://localhost:8080` et actualiser plusieurs fois. `Pod Name` peut changer entre les réponses : le Service distribue les connexions entre les pods prêts; il ne garantit pas l'affinité vers un pod unique.

Appliquer ensuite le manifest final pour ajouter les limites, les probes, le hook d'arrêt et l'image `1.10.1` :

```powershell
kubectl apply -f TP/partie5/app-deployment.yml
kubectl get pods -n prod-app -w
```

Cette application remplace l'image initiale par `1.10.1` avec une stratégie RollingUpdate. La commande `kubectl get pods -w` permet d'observer le remplacement. La liveness probe et la readiness probe apparaissent avec cette version finale du Deployment.

### Ressources et QoS

Les requests sont `100m` CPU et `64Mi`; les limits sont `200m` CPU et `128Mi`. La QoS est **Burstable**, car requests et limits sont définies mais ne sont pas égales. Au minimum Burstable, les demandes sont prises en compte par le scheduler et le conteneur dispose de plafonds; en situation de pression mémoire, BestEffort est évincé avant Burstable.

### Probes

La liveness probe interroge `/healthz`; si elle échoue assez de fois, kubelet redémarre le conteneur. Elle détecte notamment un deadlock que le processus ne signale pas par sa sortie. La readiness probe interroge `/ready` après dix secondes; lorsqu'elle échoue, le pod reste en fonctionnement mais devient non prêt. L'EndpointSlice du Service cesse alors de le proposer comme endpoint prêt. La capture des événements doit montrer l'état de la sonde après l'application du manifest :

```powershell
kubectl get pods -n prod-app
kubectl describe pod <nom-du-pod> -n prod-app
```

### RollingUpdate et arrêt propre

L'image de départ de ce dépôt est `1.10.1`. Pour observer une mise à jour en revenant à la version indiquée dans la consigne :

```powershell
kubectl apply -f TP/partie5/app-deployment-initial.yml
kubectl get pods -n prod-app -w
kubectl apply -f TP/partie5/app-deployment.yml
kubectl get pods -n prod-app -w
```

La readiness probe empêche l'ancien pod d'être retiré du Service avant que le nouveau soit prêt; `maxUnavailable: 0` demande de préserver le nombre de réplicas disponibles pendant la mise à jour.

Le hook `preStop` attend dix secondes afin de laisser finir le retrait du pod et la propagation de la suppression des endpoints avant l'arrêt. En production, l'application devrait cesser d'accepter de nouvelles requêtes, terminer celles en cours et libérer proprement ses ressources.

### Débogage éphémère

L'image applicative n'a pas nécessairement `curl`. Le conteneur de débogage éphémère évite d'ajouter des outils inutiles à l'image de production et de les maintenir ou de les exposer en permanence :

```powershell
kubectl debug -it <nom-du-pod> -n prod-app --image=busybox:1.36 --target=hello-kubernetes
```

Dans son shell :

```sh
wget -O - http://localhost:8080
```

Cette méthode réduit la taille et la surface d'attaque de l'image applicative. L'ajout d'un conteneur éphémère nécessite toutefois les permissions RBAC appropriées et l'image de débogage doit elle aussi provenir d'une source de confiance.

### Captures demandées

À produire uniquement après démarrage réel du cluster :

1. Page `hello-kubernetes` sur `localhost:8080` avec plusieurs valeurs `Pod Name`.
2. Événements de la readiness probe dans `kubectl describe pod`.
3. Remplacement progressif des pods pendant le RollingUpdate (`kubectl get pods -w`).
4. Réponse `wget` depuis le conteneur éphémère.

Ne pas présenter les schémas de consigne de `img/` comme des captures d'exécution. Enregistrer les captures réelles dans `img/` avec des noms explicites, par exemple `partie5-pods.png`.

## Limites d'exécution constatées

À la rédaction de ce compte-rendu, Docker Desktop ne répond pas, aucun profil Minikube n'existe et le projet n'a pas encore de dépôt Git local. Les manifests et les réponses sont préparés, mais aucune capture de déploiement ni validation sur un cluster vivant ne peut être revendiquée avant de démarrer Docker et Minikube et d'exécuter les commandes ci-dessus.
