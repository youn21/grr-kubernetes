# TP Kubernetes

[TOC]

## Prise en main

### Découverte de PLMshift

#### Interface web

https://plmshift.math.cnrs.fr/
https://anf2024.math.cnrs.fr/


#### Ligne de commande

##### Installation

https://plmshift.math.cnrs.fr/command-line-tools
https://anf2024.math.cnrs.fr/command-line-tools

##### Connection

```bash
# Pour plmshift
oc login --web https://api.math.cnrs.fr
# Pour plmvshift (attention à rajouter .apps)
oc login --web https://api.apps.vodka.math.cnrs.fr
# Pour anfshift
oc login --web https://api.anf.math.cnrs.fr:644
```

```bash
oc status
```

### Premiers pas

crétion d'un projet ? `oc new-project tp`

Commencez par créer un simple pod NGINX, et vérifiez que vous pouvez bien l'atteindre depuis un autre pod :

```bash
kubectl run --image nginxinc/nginx-unprivileged:latest nginx-pod
kubectl wait --for=condition=Ready pod/nginx-pod
kubectl expose pod nginx-pod --port 8080
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
kubectl run --rm --restart=Never -it --image busybox test -- wget -O- nginx-pod:$port
```

### Nettoyage

```bash
kubectl delete service nginx-pod
kubectl delete pod nginx-pod
```

## Déploiement de GRR

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
kubectl logs -l app=mariadb --tail -1
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

Note: dans la suite, de nombreuses commande utilisent [yq](https://mikefarah.gitbook.io/yq) pour modifier les fichiers YAML. La syntaxe étant au début un peu absconse, vous pouvez utilisez `diff` ou un autre logiciel de votre choix pour voir l'effet de la commande :

```bash
# Affiche la différence
yq '.kind = "StatefulSet"' mariadb.yml|diff -y mariadb.yml
# L'option -i (in-place) modifie le fichier
yq -i '.kind = "StatefulSet"' mariadb.yml
```

Un peu de magie [yq](https://mikefarah.gitbook.io/yq) pour tranformer notre deployment en statefulset...

```bash
# Remplace le champ kind
yq -i '.kind = "StatefulSet"' mariadb.yml
# Supprime les champ strategy et status
yq -i 'del(.spec.strategy)' mariadb.yml
yq -i 'del(.status)' mariadb.yml
# Ajoute le champ serviceName
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
kubectl create service clusterip --tcp=3306:3306 --clusterip='None' --dry-run=client -o yaml mariadb > mariadb-service.yml
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
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
# ou alors oc whoami
cat <<EOF > ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grr
  annotations:
    route.openshift.io/termination: edge
spec:
  rules:
  - host: grr-$oc_user.apps.anf.math.cnrs.fr
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
# plmshift
firefox http://grr-$oc_user.apps.anf.math.cnrs.fr
# plmvshift
firefox http://grr-$oc_user.apps.vodka.math.cnrs.fr
# anfshift
firefox http://grr-$oc_user.apps.anf.math.cnrs.fr
```

#### Healthcheck ####

Kubernetes utilise trois type de sondes pour évaluer l'état d'une application. Elles sont configurables au niveau de chaque conteneur.

* Les [Liveness Probes](https://kubernetes.io/fr/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#d%C3%A9finir-une-commande-de-liveness) permettent de détecter une application qui ne répond plus et qu'on doit redémarrer.
* Les [Readiness Probes](https://kubernetes.io/fr/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#d%C3%A9finir-les-readiness-probes) permette de détecter quand une application peut accepter du trafic. 
* Les [Startup Probes](https://kubernetes.io/fr/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes) permettent de laisser le temps à une application lente à démarrer.

Nous nous intéresserons ici aux deux premiers types.

##### Liveness probe #####

Par défaut, Kubernetes considère le pod en vie tant que tous les conteneurs le composant tournent. Nous pouvons affiner ce comportement en vérifiant qu'une requête HTTP aboutit. Si celle-ci n'aboutit pas, le pod sera tué.

Ajoutez la *Liveness Probe* suivante au déploiement GRR : 

```bash
          livenessProbe:
            httpGet:
              path: /
              port: http
```

##### Readiness probe #####

De la même manière, par défaut, Kubernetes considère que le pod est prêt à recevoir du trafic dès et tant que tous les conteneurs le composant tournent.
Affinons ici en utilisant un [endpoint spécifique](https://plmlab.math.cnrs.fr/anf2024/grr/-/blob/v4.3.5-docker/docker/nginx-unit/healthz.php?ref_type=heads) qui vérifie que la connexion à la base de donnée aboutit. Si celle-ci échoue, le pod ne recevra plus de trafic le temps que la connexion soit rétablie.

Ajoutez la *Readiness Probe* suivante au déploiement GRR : 

```bash
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
```

Après avoir appliqué vos changements, supprimer le service MariaDB et observer l'état du pod GRR :

```bash
kubectl delete svc mariadb
kubectl get pods --watch
```


#### Volume ####

En l'état, les données stockées dans la base MariaDB sont éphémères : à chaque redémarrage du pod MariaDB, elles sont perdues. La problématique du stockage persistant est résolue avec Kubernetes grâce à l'abstraction [Volume](https://kubernetes.io/docs/concepts/storage/volumes/). Notez que Kubernetes est capable de communiquer avec le cloud ou le matériel sous-jacent (Cinder, dans le cas d'OpenStack). On peut alors utiliser du stockage dynamique via l'objet [PersitentVolumeClaim](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/).


Définissez l'objet PersitentVolumeClaim :

``` bash
cat <<EOF > pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
kubectl apply -f pvc.yml
# En fonction de la configuration de la storageclasse, la PVC peut n'être créée
# qu'au moment ou elle est attachée à un pod (VOLUMEBINDINGMODE WaitForFirstConsumer)
kubectl describe pvc mariadb
```

Une fois le volume disponible, attachez-le au pod du StatefulSet MariaDB, au point de montage `/var/lib/mysql`. Verifiez ensuite qu'un redémarrage du pod préserve les données.

```bash
# yq magie !
yq -i '.spec.template.spec |= ({"volumes": [{"name": "mariadb", "persistentVolumeClaim": {"claimName": "mariadb"}}]} + .)'  mariadb.yml
yq -i '.spec.template.spec.containers[0] |= ({"volumeMounts": [{"mountPath": "/var/lib/mysql", "name": "mariadb"}]} + .)'  mariadb.yml
kubectl apply -f mariadb.yml
kubectl wait --for=condition=Ready pod/mariadb-0
# On vérifie l'état de la PVC
kubectl describe pvc mariadb
# On vérifie le point de montage
kubectl exec -i -t mariadb-0 -- df -h /var/lib/mysql
```

#### Ressources ####

#### Secrets ####

#### NetworkPolicy ####

#### Helm ####

[Helm](https://helm.sh) est un moteur de template spécialisé dans la gestion du déploiement d'applications sur Kubernetes. Les *charts* Helm permettent de décrire les applications et de les distribuer. À ce titre, il est parfois considéré comme le *package manager* de Kubernetes.

L'objectif est ici d'écrire un *chart* Helm qui permettra de faciliter le déploiement de GRR. Nous nous baserons sur un squelette (*starter*) afin d'en faciliter l'écriture.


##### Récupération du template et initialisation du *chart* #####

Clonez le dépôt pour récupérer le "starter" template.

```bash
mkdir ~/.local/share/helm/starters/
git  -C  ~/.local/share/helm/starters/ clone git@plmlab.math.cnrs.fr:anf2024/grr-kubernetes.git
```
Créez un nouveau chart à l'aide de ce template et inspectez-le :

```bash
helm create grr -p grr-kubernetes/helm/starter/
# Fichier de description du chart, dont la version de l'application empaquetée
less grr/Chart.yaml
# Valeurs par défaut
less grr/values.yaml
# Fichiers de templates
less grr/templates
```

##### Premiers tests #####

Inspectez la configuration du *chart* et générez les ressources. La commande `template` permet de rendre localement et d'afficher les ressources générées sans les déployer

```bash
# Inspection des valeurs de configuration par défaut
helm show values grr/
# Le premier argument est le nom de votre déploiement, le second le nom du chart. ici on travaille avec un chart local
helm template my-grr-deployment grr/
```
Mettez à jour la version de nginx et déployez :

```bash
yq -i '.appversion="1.27"' grr/Chart.yaml
helm install my-grr-deployment grr/
```
Par défaut, le *chart* ne déploie pas de règle d'ingress. on peut en ajouter une en positionnant les variables à l'aide de l'option `--set`.

```bash
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
helm upgrade my-grr-deployment grr/
 --set ingress.enabled=true --set ingress.hosts[0].host=$oc_user-grr-helm.apps.anf.math.cnrs.fr --set ingress.hosts[0].paths[0].path=/ --set ingress.hosts[0].paths[0].pathType=Prefix --set ingress.annotations."route\.openshift\.io/termination"=edge
curl https://$oc_user-grr-helm.apps.anf.math.cnrs.fr
```

Vous conviendrez aisément que passer tout les paramètres via la ligne de commande est pénible. Soyez déclaratifs !

```bash
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
cat <<EOF > my-values.yaml
ingress:
  enabled: true
  annotations:
    route.openshift.io/termination: edge
  hosts:
    - host: $oc_user-grr-helm.apps.anf.math.cnrs.fr
      paths:
        - path: /
          pathType: Prefix
EOF
```
Vous pouvez désormais utiliser l'option `-f my-values.yaml`. Les valeurs présentes dans ce fichier surchargeront celles pas défaut.

##### GRR Helm Chart ! #####

Adaptez maintenant le *chart* Helm pour GRR.

```bash
yq -i '.appVersion="v4.3.5-docker-15"' grr/Chart.yaml
yq -i '.image.repository="registry.plmlab.math.cnrs.fr/anf2024/grr"' grr/values.yaml
helm upgrade my-grr-deployment grr/ -f my-values.yaml
```

Il vous faut à présent ajouter le service MariaDB. Pour cela, Helm offre un mécanisme de dépendances. Nous nous appuierons sur le [*chart* MariaDB offert par Bitnami](https://artifacthub.io/packages/helm/bitnami/mariadb).

```bash
# yq magie !
yq -i '.dependencies |= ([{"name": "mariadb", "version": "19.0.5", "repository": "https://charts.bitnami.com/bitnami", "condition": "mariadb.enabled"}] + .)' grr/Chart.yaml
helm dependency update grr/
# on vérifie que la dépendance a bien été téléchargée
ls grr/charts/
```

Ajoutez les valeurs par défaut nécessaires au *chart* de dépendance. On utilise pour cela le nom de la dépendance déclarée dans l'étape précédente :

```bash
# yq magie !
yq -i '. |= ({"mariadb": {"auth": {"database": "grr", "rootPassword": "grr_root_password", "username": "grr_user", "password": "grr_password"}}} + .)' grr/values.yaml
```

Maintenant, à vous de jouer ! Utilisez les valeurs par défaut du *chart* pour injecter les variables d'environnement au fichier `grr/templates/deployment.yaml`. Dans un template, les valeurs par défaut sont accessibles à l'aide de la notation `{{ .Values.mariadb.auth.* }}`

Rappel : les variables sont `DB_NAME`, `DB_USER`, `DB_PASSWORD` et `DB_HOST`.

Pour cette dernière, il faut obtenir le nom du service MariaDB généré par le subchart. C'est possible via la notation `{{ template "mariadb.primary.fullname" .Subcharts.mariadb }}`.

Vérifiez vos modifications et appliquez :

```bash
helm template my-grr-deployment grr/ -f my-values.yaml
helm upgrade my-grr-deployment grr/ -f my-values.yaml
```

##### Les *Hooks* #####

Il vous faut maintenant lancer les migrations de base. Avec Helm, nous utiliserons le mécanisme de [Hooks](https://helm.sh/docs/topics/charts_hooks/) au lieu d'un *initContainer*, en utilisant la primitive Kubernetes [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) qui représente une tâche ponctuelle. Les annotations permettent de d'ordonnancer les exécutions, ici à la fin du déploiement (il faut que le pod MariaDB soit prêt).

```bash
cat <<EOF > grr/templates/job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "grr.fullname" . }}-migrate-{{ now | unixEpoch }}
  labels:
    {{- include "grr.labels" . | nindent 4 }}
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  backoffLimit: 8
  template:
    metadata:
      name: {{ .Chart.Name }}-migrate
      labels:
        {{- include "grr.labels" . | nindent 8 }}
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}
          command: ["/bin/bash", "-c", "--"]
          args: ["echo 'Hello, will sleep 60'; sleep 60"]
EOF
# On vérifie et on applique
helm template my-grr-deployment grr/ -f my-values.yaml
helm upgrade my-grr-deployment grr/ -f my-values.yaml
```
Les jobs hooks sont bloquants. Vous pouvez voir l'effet dans un autre terminal:

```bash
kubectl get jobs
kubectl get pods
kubectl logs -f -l batch.kubernetes.io/job-name=(kubectl get job -o jsonpath='{.items[0].metadata.name}')
```
Modifiez maintenant le *hook* pour qu'il lance les migrations. Appuyez-vous sur l'*initContainer* de la première partie et n'oubliez pas les variables d'environnement !

Une fois le hook en place, vous devez pouvoir vous connecter à l'instance déployée avec Helm !


##### Publiez votre *chart* #####

Pour finir, vous allez publier votre chart. Ici, nous utiliserons la [possibilité offerte](https://docs.gitlab.com/ee/user/packages/helm_repository/) par Gitlab.

Avant toutes choses, vérifiez que le chart est correct et que la construction du paquet se déroule correctement :

```bash
# Vérifie la syntaxe
helm lint grr
# Construit le paquet -- format tgz
helm package grr
```

Créez maintenant un nouveau projet Gitlab en forkant [grr-helm](https://plmlab.math.cnrs.fr/anf2024/grr-helm), soit via l'interface web soit à l'aide de [glab](https://docs.gitlab.com/ee/editor_extensions/gitlab_cli/)

```bash
# Pour les inconditionnel·les de la CLI !
glab repo fork anf2024/grr-helm
```

Ajoutez-y le repertoire `grr` contenant le chart, observez le fichier `.gitlab-ci.yaml`. Une fois les changements poussés, déclenchez la CI via l'interface web... ou glab.

Note: le déclenchement de la CI est manuelle dans cet exemple afin de ne pas complexifier le propos.

Une fois le pipeline terminé, vous pouvez ajouter le dépôt nouvellement créé à la configuration de Helm :

```bash
# On récupère l'id du projet (visible dans le menu kebab en haut à droite de la page d'accueil)
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
project_id=$(curl  https://plmlab.math.cnrs.fr/api/v4/projects/$oc_user%2Fgrr-helm|jq -r '.id')
helm repo add grr https://plmlab.math.cnrs.fr/api/v4/projects/$project_id/packages/helm/stable
helm repo update
```

Vous pouvez maintenant installer une nouvelle instance simplement !

```bash
helm install my-new-deployment grr/grr  --set ingress.enabled=true --set ingress.hosts[0].host=$oc_user-grr-helm-new.apps.anf.math.cnrs.fr --set ingress.hosts[0].paths[0].path=/ --set ingress.hosts[0].paths[0].pathType=Prefix --set ingress.annotations."route\.openshift\.io/termination"=edge
```

#### gitops ####
