# TP Kubernetes

## Prise en main

### Découverte de PLMshift

#### Interface web

https://plmshift.math.cnrs.fr/


#### Ligne de commande

##### Installation

https://plmshift.math.cnrs.fr/command-line-tools

##### Connection

```
oc login --web https://api.math.cnrs.fr
```

```
oc status
```

### Premiers pas

crétion d'un projet ? `oc new-project tp`

Commencez par créer un simple pod NGINX, et vérifiez que vous pouvez bien l'atteindre depuis un autre pod :

```bash
kubectl run --image nginxinc/nginx-unprivileged:latest nginx-pod
kubectl expose pod nginx-pod --port 80
kubectl get pods -o wide
pod_ip=$(kubectl get pod nginx-pod -o jsonpath='{.status.podIP}')
kubectl get service
service_ip=$(kubectl get service nginx-pod -o jsonpath='{.spec.clusterIP}')
port=$(kubectl get service nginx-pod -o jsonpath='{.spec.ports[0].port}')
```

Essayez de l'atteindre depuis un autre pod, via son IP, l'IP du service, puis le nom DNS du service :
```bash
kubectl run --rm --restart=Never -it --image busybox test -- wget -O- $pod_ip:$port
kubectl run --rm --restart=Never -it --image busybox test -- wget -O- $service_ip:$port
kubectl run --rm --restart=Never -it --image busybox test -- wget -O- $pod_name:$port
```

### Nettoyage

```bash
kubectl delete service nginx-pod
kubectl delete pod nginx-pod
```

## GRR

Nous allons lancer deux pods : l'un pour l'application GRR, basé sur [NGINX Unit](https://unit.nginx.org/) et l'autre pour MariaDB.
Le premier est contrôlé par un [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) qui permet le passage à l'échelle et l'autoréparation.
Le second est contrôlé par un [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) qui est recommandé pour les charge de travail stateful, avec un besoin de stockage persistant stable par exemple.

### Mariadb ###

#### Premier déploiement
Commençons par déployer MariaDB avec un Deployment.

_Astuce: pour générer les manifestes yaml facilement, vous pouvez utiliser la commande `kubectl` avec les options `--dry-run=client -o yaml`_ :
```bash
kubectl create deployment mariadb --image quay.io/sclorg/mariadb-1011-c9s:c9s --dry-run=client -o yaml > mariadb.yml
```
Examiner le fichier créé et lancez le déploiement:
```bash
less mariadb.yml
kubectl apply -f mariadb.yml
```
Surveillez l'état du pod :
```bash
kubectl get pods -w
```
#### Debug ####
Le pod mariadb produit une erreur. Pour la comprendre :
```bash
kubectl describe pod -l app=mariadb
```
L'erreur est `Back-off restarting failed container`, ce qui signifie que le pod redémarre sans cesse. Pour examiner plus précisément l'activité d'un conteneur du pod :
```bash
kubectl logs -l app=mariadb
```
On voit qu'il manque des options (variables d'environnement) pour que le conteneur mariadb démarre correctement. Définissez les variables `MYSQL_ROOT_PASSWORD`, `MYSQL_USER`, `MYSQL_PASSWORD` et `MYSQL_DATABASE` dans le fichier `mariadb.yml` à l'aide du champ [env](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/). Profitez-en pour positionner `MYSQL_CHARSET` à la valeur `utf8mb4`.

_Dans la suite, nous verrons comment éviter les secrets en clair... Si nous avons le temps ;)_

Une fois les variables définies, réappliquez les changements et vérifiez que le pod tourne :
```bash
kubectl apply -f mariadb.yml
kubectl get pods -w
```

#### Tests ####

Pour tester facilement que le pod mariadb répond correctement, et pour les futurs debug, vous pouvez utiliser la commande `kubectl port-forward` :
```bash
mariadb_pod=$(kubectl get pods -l app=mariadb -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward $mariadb_pod 3306:3306
nc localhost 3306
```

Nous avons vu que les pods et leur configuration réseau sont temporaires. Afin de les exposer de manière pérenne, nous pourrions créer un service avec une IP interne comme dans la première partie. Dans le cas d'une application stateful, nous allons plutôt tirer parti des possibilités offertes par les  [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) et les [Headless services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).

Un peu de magie [yq](https://mikefarah.gitbook.io/yq) pour tranformer notre deployment en statefulset...

```bash
# Remplace le champ kind
yq -i '.kind = "StatefulSet"' mariadb.yml
# Supprime les champ strategy et status
yq -i 'del(.spec.strategy)' mariadb.yml 
yq -i 'del(.status)' mariadb.yml
# Ajout le champ serviceName
yq -i '.spec.serviceName = "mariadb"' mariadb.yml
# Vérification et déploiement
less mariadb.yml
kubectl apply -f mariadb.yml
kubectl get pods -w
# Suppression du Déployment
kubectl delete deployment  mariadb
```

Créez maintenant le service Headless. Il n'a pas d'IP et ne permet pas le Load-Balancing, mais un simple enregistrement DNS qui pointe directement sur l'IP du pod.

```bash
kubectl create service clusterip --tcp=3306:3306 --clusterip='None' --dry-run=client -o yaml mariadb-service > mariadb-service.yml
kubectl apply -f mariadb-service.yml
```

Vous pouvez créer un pod temporaire afin de vérifier la résolution DNS et le bon fonctionnement du serveur mariadb.
```bash
kubectl run --rm --restart=Never -i -t --image busybox test /bin/sh
nc mariadb 3306
```
### GRR ###

#### Déploiement de GRR (à vous de jouer) ####

Sur l'exemple des *manifests* mariadb, créez un *manifest* pour déployer les conteneurs GRR.

_Vous pouvez déclarer plusieurs objets dans le même fichier en les séparant par la ligne `---`_

Pensez à :
* Specifier l'image correspondante: registry.plmlab.math.cnrs.fr/anf2024/grr:stable 
* Positionner les variables *DB_HOST*, *DB_NAME*, *DB_USER*, *DB_PASSWORD*;
* Spécifier le port (8080);
* Créer le service corespondant.

Après avoir appliqué le *manifest*, vérifez que vous accédez au conteneur avec `kubectl port-forward`.


#### Migration de base ####

L'application ne marchera pas en l'état, puisqu'il faut lancer les migrations de base avant de lancer l'application GRR. Ajoutez un [initContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) au déploiement, qui utilise la même image et dont le rôle sera d'éxecuter la commande `/var/www/docker/nginx-unit/migrations.sh`

Verifiez le bon fonctionnement de l'application à l'aide de :
```bash
kubectl port-forward svc/grr 8080:8080
```

#### Ingress ####

Vous pouvez maintenant exposer l'application au monde extérieur grâce à l'objet [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).

Nous pouvons alors définir l'objet Ingress qui va diriger le trafic vers le service grr :

```bash
oc_user=$(oc auth whoami -o jsonpath='{.status.userInfo.username}')
cat <<EOF > ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grr
  annotations:
    route.openshift.io/termination: edge
spec:
  ingressClassName: openshift-default
  rules:
  - host: grr-$oc_user.apps.math.cnrs.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grr
            port:
              number: 8080
EOF
kubectl apply -f ingress.yml
```

Vous n'avez désormais plus besoin de *port-forward* (qui est un mécanisme prévu pr le debug) pour accéder à votre application :

```bash
firefox http://grr-$oc_user.apps.math.cnrs.fr
```

### 3. Aller plus loin ###

#### Ressources ####

#### Secrets ####

#### Volume ####

#### NetworkPolicy ####

#### Helm ####

#### GitOps ####
