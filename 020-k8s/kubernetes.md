# K8s

Au lieu de se connecter à un serveur spécifique, de télécharger la source, de construire une image, etc., nous allons plutôt déployer notre conteneur dans le nuage. Le nuage se chargera de faire fonctionner le conteneur pour nous !

Nous allons utiliser un cluster Kubernetes que je vous ai fourni pour déployer votre conteneur.

Je fournirai à chacun d'entre vous un fichier de configuration qui vous donnera un accès exclusif à mon cluster kubernetes.

```yml
apiVersion: v1
clusters:
- name: "hetic-devops"
  cluster:
    certificate-authority-data: LS0tLS1CRUdJTiB...
    server: https://96ff...
contexts:
- name: kevin-nguni-fr
  context:
    cluster: "hetic-devops"
    user: s-...
    namespace: s-...
current-context: s-...
kind: Config
preferences: {}
users:
- name: s-...
  user:
    token: eyJhbGciOiJS....
```

Retournez à votre DevContainer. J'ai déjà installé l'application `kubectl` dans votre DevContainer qui nous permet de communiquer avec un cluster Kubernetes.

Copiez le fichier `kubeconfig-....yaml` dans votre devcontainer, sous le dossier `k8s` (créez ce dossier s'il n'existe pas encore).

Dans le terminal de votre DevContainer, naviguer dans le dossier `k8s` :

```sh
cd k8s
```

Commencez par créer une variable d'environnement qui pointe vers votre fichier kubeconfig :

```sh
export KUBECONFIG=./kubeconfig.yaml 
```

Nous pouvons maintenant interagir avec le cluster Kubernetes, en listant tous les *pods* (ou conteneurs) qui sont en cours d'exécution :

```sh
kubectl get pods
```

## Deployment

Nous allons maintenant indiquer à k8s que nous voulons déployer notre API sur le cluster. Nous allons écrire un fichier de configuration de type "deployment".

```yaml
# k8s/deployment.k8s.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devopsapi
  labels:
    app: devopsapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devopsapi
  template:
    metadata:
      labels:
        app: devopsapi
    spec:
      containers:
        - name: devopsapi
          image: drkevinglass/devopsapi:1.0.0
          command: ["npm", "run", "start-api"]

```

Remplacez `drkevinglass/devopsapi:1.0.0` par le nom de votre image envoyé à Docker Hub.

Lancez votre image dans le cluster avec :

```sh
kubectl apply -f deployment.k8s.yaml 
```

Si tout se passe bien, l'image sera récupéré de Docker Hub, et lancé automatiquement sur mon cluster !

Vérifions :

```sh
kubectl get pods
# Vous devez voir  2 pods       

kubectl get deployments
# Vous devez vois 1 deployment

kubectl logs [ID DU POD]
# Vous devez voir les logs de votre API
```

A chaque fois qu'on veut modifier notre deploiement, par exemple, le nombre de copies, ou mettre à jour la version de notre API, on modifie le fichier `deployment.k8s.yaml`, et on éxecute `kubectl apply ...`

## Service

Pour l'instant, notre déploiement est en cours, mais il n'est pas disponible à l'extérieur. Nous devons établir ce lien. Il fonctionne comme suit :

Load-Balancer &rarr; Ingress &rarr; Service &rarr; Deployment &rarr; Pods

Nous avons déjà configuré le "Deployment" qui a généré automatiquement plusieurs Pods. 

Le LoadBalancer est une ressource payante de mon fournisseur de services que j'ai déjà configurée pour vous. Je vous donnerai l'adresse IP précise en classe.

Il ne reste plus qu'à connecter le Ingress et le Service.

Créez un nouveau fichier pour le service :

```yaml
# k8s/service.k8s.yaml

apiVersion: v1
kind: Service
metadata:
  name: devopsapi-svc
  labels:
    app: devopsapi
spec:
  ports:
    - port: 5055
      targetPort: 5055
      protocol: TCP
  selector:
    app: devopsapi
```

Ensuite, déployez le service vers le cluster :

```sh
kubectl apply -f service.k8s.yaml 
```

## Ingress

Nous voulons maintenant indiquer au cluster comment rediriger les requêtes entrantes. Pour ce faire, nous utilisons un contrôleur *Ingress*.

Comme nous partageons tous le même cluster (et donc la même adresse IP), nous allons distinguer nos déploiements par un sous-chemin.

Par exemple, une requête à 

```
http://cluster.mt.glassworks.tech/kevin-nguni-fr 
```

devrait être dirigée vers mon API.

Par contre, une requête vers 

```
http://cluster.mt.glassworks.tech/t_joubert-hetic-eu
```

devrait être redirigée vers l'API de cet étudiant.

Crée un nouveau fichier :

```yaml
# k8s/ingress.k8s.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /MON_CHEMIN_UNIQUE(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: devopsapi-svc
            port:
              number: 5055
    
```

Remplacez `MON_CHEMIN_UNIQUE` par un chemin qui vous identifie, par exemple, votre adresse e-mail (ayant remplacé le `@` et les `.` par des tirés `-`)

Déployez cette configuration avec :

```sh
kubectl apply -f ingress.k8s.yaml 
```

## Tester

Vous devriez pouvoir envoyer une requête au serveur maintenant !

```sh
# Remplacez l'adress IP et par l'adresse fourni du cluster
# Remplacez MON_CHEMIN_UNIQUE par le chemin indique dans le ingress
curl http://cluster.mt.glassworks.tech/MON_CHEMIN_UNIQUE/info

# Resultat :
#{"title":"DevOps Code Samples API","host":"devopsapi-7774dbf9fb-sb5js","platform":"linux","type":"Linux"}%  
```

Félicitations ! Vous avez réussi à mettre en place votre environnement de production !


## Variables d'environnement (chapitre facultatif)

Comment préciser les variables d'environnement pour nos deploiements ? Crée un **config-map** : 

```yaml
# k8s/api-cfg.k8s.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devopsapi-cfg
data:
  PORT: "5001"
```

Ensuite, l'appliquer :

```bash
kubectl apply -f ./api-cfg.k8s.yaml
```

Il faut ensuite préciser au déploiement d'utiliser le config-map et appliquer les variables d'environnement :

```yaml
# k8s/deployment.k8s.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devopsapi
  labels:
    app: devopsapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devopsapi
  template:
    metadata:
      labels:
        app: devopsapi
    spec:
      containers:
        - name: devopsapi
          image: drkevinglass/devopsapi:1.0.0
          command: ["npm", "run", "start-api"]
          envFrom:
          - configMapRef:
              name: devopsapi-cfg

```

Appliquer la modification : 

```sh
kubectl apply -f deployment.k8s.yaml 
```

Attention, on ne stocke jamais les secrets dans un config-map, car les valeurs sont stockées en texte claire !


## Secrets (chapitre facultatif)

Comment les secrets sont-ils stockés dans Kubernetes, si ce n'est via les ConfigMaps ?

Nous devons ponctuellement créer des secrets à l'aide d'une commande unique qui n'est pas stockée dans un fichier ou dans GIT. Nous essayons également de ne jamais mettre un secret directement sur la ligne de commande, car n'importe qui pourrait récupérer notre historique.

Tout d'abord, nous créons un fichier `.env` qui stocke nos secrets localement, et qui est ignoré par GIT (il doit se trouver dans le fichier `.gitignore`)


```bash
export DB_DATABASE=
export DB_PASSWORD=
...
```

Nous chargeons ensuite ces variables dans le shell actuel:

```bash
source .env
```

Testez que vos variables sont chargées : 

```bash
echo $DB_DATABASE
```

Vous devriez voir votre valeur affichée dans le shell.

Ensuite, on va créer les secrets sur le cluster :

```bash
kubectl delete secret devopsapi-secrets
kubectl create secret generic devopsapi-secrets  \
  --from-literal=DB_DATABASE="$DB_DATABASE" \
  --from-literal=DB_PASSWORD="$DB_PASSWORD"
```

Il faut ensuite préciser au déploiement d'utiliser les secrets :

```yaml
# k8s/deployment.k8s.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devopsapi
  labels:
    app: devopsapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devopsapi
  template:
    metadata:
      labels:
        app: devopsapi
    spec:
      containers:
        - name: devopsapi
          image: drkevinglass/devopsapi:1.0.0
          command: ["npm", "run", "start-api"]
          envFrom:
          - configMapRef:
              name: devopsapi-cfg
          - secretRef:
              name: devopsapi-secrets

```

Appliquer la modification : 

```sh
kubectl apply -f deployment.k8s.yaml 
```

## Relancer vos services

Après avoir modifié les variables d'environnement et/ou secrets, pour les appliquer aux deploiements existants, il suffit de juste forcer le redemarrage des pods, en scalant à 0 et puis en les recréant :

```bash
kubectl scale --replicas=0 deployment/devopsapi
kubectl scale --replicas=2 deployment/devopsapi
```
