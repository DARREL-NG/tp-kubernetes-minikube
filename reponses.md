# Réponses — TP Kubernetes

## 1. Pod autonome vs Pod géré par un Deployment

Un **Pod autonome** n'est surveillé par aucun contrôleur : s'il est supprimé ou s'il plante, il n'est pas recréé (après `kubectl delete pod whoami-pod`, `kubectl get pods` renvoie *No resources found*).

Un **Pod géré par un Deployment** appartient à un ReplicaSet qui maintient le nombre de réplicas déclaré : un Pod supprimé est automatiquement remplacé. Le Deployment permet aussi de changer d'échelle (`kubectl scale`) et de faire des mises à jour progressives (rolling update).

## 2. Rôle du ReplicaSet dans l'auto-réparation

Le ReplicaSet compare en boucle l'état réel (nombre de Pods portant le label `app: whoami`) à l'état voulu (`replicas: 3`). Dès qu'un écart apparaît, il le corrige : il crée un nouveau Pod si un Pod disparaît, ou en supprime si leur nombre dépasse la cible. Dans le TP, la suppression de `whoami-c6f799d48-6vcrt` a immédiatement entraîné la création de `whoami-c6f799d48-fxnzj`.

## 3. Rôle du Service et vérification de sa cible

Les Pods sont éphémères et changent d'IP. Le **Service** fournit un nom DNS et une IP stables (`whoami-service`, type ClusterIP) et répartit le trafic entre les Pods sélectionnés par son `selector` (`app: whoami`).

Pour vérifier qu'il cible les bons Pods :
- `kubectl get endpoints whoami-service` (ou `kubectl get endpointslices -l kubernetes.io/service-name=whoami-service`) liste les IP des Pods ciblés : `10.244.0.10`, `.11`, `.12` ;
- on compare ces IP avec `kubectl get pods -o wide` et on vérifie les labels avec `kubectl get pods --show-labels` ;
- on teste l'accès HTTP : `kubectl port-forward service/whoami-service 8080:80` puis `curl http://localhost:8080`.

## 4. Pourquoi `minikube image load` et `imagePullPolicy: Never` ?

L'image `monapp:1.0` est construite dans le Docker de la machine hôte, alors que Minikube possède son propre runtime de conteneurs (containerd), qui ne voit pas les images locales de l'hôte. `minikube image load` copie donc l'image dans le nœud Minikube.

`imagePullPolicy: Never` force Kubernetes à utiliser uniquement l'image déjà présente sur le nœud. Sans cela, il tenterait de télécharger `monapp:1.0` depuis Docker Hub, où elle n'existe pas, et le Pod tomberait en `ErrImagePull` / `ImagePullBackOff`.
