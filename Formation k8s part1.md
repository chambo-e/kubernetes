---------------------------------------------------------------------------------------------------------------
# Formation Kubernetes
---------------------------------------------------------------------------------------------------------------


---------------------------------------------------------------------------------------------------------------
## Les Nodes
---------------------------------------------------------------------------------------------------------------
Exemple pour la création d'un objet k8s de type node:
```yaml
apiVersion: v1
kind: node
metadata:
   name: < ip address of the node>
   labels:
      name: <lable name>
```
Information: Les noeuds Kubernetes peuvent être programmés sur Capacité. Les pods peuvent utiliser toute la capacité disponible sur un nœud par défaut : https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/




---------------------------------------------------------------------------------------------------------------
## Kubectl
---------------------------------------------------------------------------------------------------------------
### Installation Kubectl:

1/ Télécharger le binaire:
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

2/ Rendre le binaire kubectl exécutable:
```bash
 chmod +x ./kubectl
```

3/ Déplacez le binaire dans le PATH:
```bash
 sudo mv ./kubectl /usr/local/bin/kubectl
```

4/ Activer de l'auto-complétion en exécutant :
```bash
$ source <(kubectl completion bash)
```

Pour ajouter l'autocomplétion à votre profil:
```bash
 echo "source <(kubectl completion bash)" >> ~/.bashrc 
```

5/ vérifier la version et l'aide Kubectl:
```bash
$ kubectl version
$ kubectl -h
```



### Configurer Kubectl:
Il faut configurer kubectl en local pour pouvoir interagir avec le cluster Kubernetes (IP de l'apiserver, certificats client, Token,informations d'identification utilisateur. Spécifier une option qui existe déjà fusionnera de nouveaux champs avec les valeurs existantes pour ces champs. Par défaut, la configuration de kubectl est située à ~/.kube/config.

Pour modifier le fichier kubeconfig:
```bash
$ kubectl config -h
$ kubectl <command> --help
$ kubectl config –-kubeconfig <String of File name>
```
Pour obtenir une liste d’options globales:
```bash
$ kubectl options
```

1/ Définir une entrée de cluster dans Kubeconfig :
kubectl config set-cluster NAME [--server=server] [--certificate-authority=...] [--insecure-skip-tls-verify=true] [options]
*--insecure-skip-tls-verify=true(Désactive la vérification de certification)

```bash
$ kubectl config get-clusters
$ kubectl config set-cluster $CLUSTER_NAME --certificate-authority=ca.pem --embed-certs=true --server=https://$MASTER_IP
```
Pour supprimer le cluster:
```bash
$ kubectl config delete-cluster <$CLUSTER_NAME>
```

2/ Définir les credentials (entrée utilisateur):
```bash
$ kubectl config set-credentials -h
$ kubectl config set-credentials $USER --client-certificate=$CLI_CERT --client-key=admin-key.pem --embed-certs=true --token=$TOKEN
ou
$ kubectl config set-credentials default-admin --certificateauthority = ${CA_CERT} --client-key = ${ADMIN_KEY} --clientcertificate = ${ADMIN_CERT}
```

3/ Définissez le context par défaut dans Entrypoint k8s:
```bash
$ kubectl config get-contexts
$ kubectl config set-context $CONTEXT_NAME --cluster=$CLUSTER_NAME --user=$USER --namespace=$namespace
$ kubectl config use-context $CONTEXT_NAME
$ kubectl config current-context

Pour supprimer un contexte spécifié de kubeconfig:
$ kubectl config delete-context <Context Name>
```

3/ Vérifier la configuration :
```bash
$ kubectl config view
```

4/ Afficher les versions d'API prises en charge sur le cluster.
```bash
$ kubectl api-versions
```



### Vérification l'état du cluster:
1/ Affiche les informations du cluster.
```bash
 $ kubectl cluster-info 
 *L’URL indique que kubectl est correctement configuré pour accéder au cluster (Apiserver)
 ```
Dans le cas contraire, vérifiez que celui-ci est correctement configuré:
Affiche les informations pertinentes concernant le cluster pour le débogage et le diagnostic.
```bash
 $ kubectl cluster-info dump
 $ kubectl cluster-info dump --output-directory=/path/to/cluster-state
```

2/ Vérifiez le status de chaque composant:
```bash
$ kubectl get cs
$ kubectl get all --all-namespaces
$ kubectl get pods --all-namespaces
```

5/ Obtenez des informations sur les nodes du cluster:
```bash
$ kubectl get nodes
$ kubectl describe nodes
```

6/ Exécute kubectl en mode reverse-proxy et tester le serveur API et l'authentification : 
```bash
$ kubectl proxy --port=8080 &
$ curl http://localhost:8080/
```

7/ Obtenez une liste et l'URL du reverse Proxy de l'ensemble des services demarré sur le cluster:
```bash
$ kubectl get all
$ kubectl get services --namespace=kube-system 
```

8/ Baculer un node en mode maintenance ou normal:
```bash
$ kubectl drain $NODENAME
$ kubectl uncordon $NODENAME
```
Pour vider en toute sécurité un node tout en respectant les SLO d'applicationVoir: 
https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/




### Utiliser Kubectl:
Lorsque d'une opération sur plusieurs ressources, on peut spécifier chaque ressource par type d'objet et nom ou spécifier un ou plusieurs fichiers:

1/ spécifier des ressources du même type:
```bash
$ kubectl get pod example-pod1 example-pod2
```

2/ spécifier plusieurs types de ressources individuellement: 
```bash
$ kubectl get pod/example-pod1 replicationcontroller/example-rc1
```

3/ spécifier les ressources avec un ou plusieurs fichiers: 
```bash
$ kubectl get pod -f ./pod.yaml
```



### Migration de commandes
Migration des commandes impératives vers la configuration d'objet impérative:

1/ Exportez l'objet live dans un fichier de configuration d'objet yaml:
```bash
$ kubectl get <kind>/<name> -o yaml --export > xxx.yaml
```

2/ Modifer manuellement les champs du nouveau fichier.

3/ Pour la gestion d'objet ultérieure, utilisez replace exclusivement: 
```bash
$ kubectl replace -f xxx.yaml 
```



---------------------------------------------------------------------------------------------------------------
## Manifeste Template:
---------------------------------------------------------------------------------------------------------------
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monpod
  namespace: namespace1
  annotations:
    description: my frontend
  labels:
    environment: "production"
    tier: "frontend"
spec:
  restartPolicy: Always
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: container1
    image: ubuntu
    imagePullPolicy: Always
    env:
    -name: envname
     value: "valenv"
    ressources:
     limits:
      memory: "200Mi"
      cpu: "1"
      ephemeral-storage: "4Gi"
     requests: 
      memory: "100Mi"
      cpu: "0.5"
      ephemeral-storage: "2Gi"
    securityContext:
     allowPrivilegeEscalation: false
    livenessProbe:
     initialDelaySeconds: 15
     timeoutSeconds: 1
     httpGet:
      path: /site
      port: 8080
      httpHeaders:
      - name: X-Custom-Header
        value: Awesome
    command: ['sh','-c','echo Hello onepoint! && sleep ]
    #args: -cpus "2"
    
    *Args:-cpus "2" dit au conteneur d'essayer d'utiliser 2 cpus. Mais le conteneur est seulement autorisé à utiliser environ 1 CPU. 
```



---------------------------------------------------------------------------------------------------------------
## Traduire un fichier Docker Compose en Kompose
---------------------------------------------------------------------------------------------------------------
Traduire un fichier Docker Compose en ressources Kubernetes (Kubernetes + Compose = Kompose)
C'est un outil de conversion (Docker Compose) pour les orchestrateurs de conteneurs (Kubernetes ou OpenShift).
http://kompose.io/

Installation kompose:
```bash
curl -L https://github.com/kubernetes/kompose/releases/download/v1.17.0/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
```

1/ Utilisation en trois étapes :
 - Accédez au répertoire contenant votre fichier "docker-compose.yaml"
 - Exécutez "kompose up" dans le même répertoire
 - Vérifiez le déploiement des containers dans le cluster Kubernetes "kubectl get po"

2/ Alternativement: vous pouvez exécuter kompose convert pour générer un fichier à utiliser avec kubectl.
```bash
$ kompose --file docker-convert.yml convert
$ kompose --provider openshift --file docker-convert.yml convert
```

Voir: https://kubernetes.io/docs/tools/kompose/user-guide/



---------------------------------------------------------------------------------------------------------------
## NAMESPACE:
---------------------------------------------------------------------------------------------------------------
1/ Lister tous les namespaces:
```bash
$ kubectl get namespaces
$ kubectl get namespaces --show-labels
```

2/ Créer un namespace:
```bash
$ kubectl create namespace tst
$ kubectl create –f namespace-tst.yaml
```
```yaml  
apiVersion: v1
kind: Namespace
metadata:
  name: tst
```

3/ Obtenir des informations sur un namespace:
```bash
$ kubectl describe namespaces tst
$ kubectl get all --namespace=tst
$ kubectl get <type> --namespace=tst 
```

4/ Définir un namespace pour un objet:
```bash
$ kubectl --namespace=tst run nginx --image=nginx (obsolete)
```

5/ Définir le namespace par defaut pour toutes les commandes kubectl:
```bash
$ kubectl config set-context $(kubectl config current-context) --namespace=tst
$ kubectl config view | grep namespace
```

6/ Supprimer un namespace :
```bash
$ kubectl delete namespace tst
```

---------------------------------------------------------------------------------------------------------------
## ResourceQuota:
---------------------------------------------------------------------------------------------------------------
1/ Créer un ResourceQuota
```yaml  
 apiVersion:   v1 
  kind:   ResourceQuota 
  metadata: 
    name:   quota
  spec: 
    hard: 
      requests.cpu:   "1" 
      requests.memory:   1Gi 
      limits.cpu:   "5" 
      limits.memory:   10Gi 
```
```bash
$ kubectl create -f resourcequota.yaml --namespace=tst
```

2/ Afficher les ResourceQuota:
```bash
$ kubectl get resourcequota quota --namespace=tst --output=yaml 
$ kubectl describe namespaces tst
```

3/ Modifier le fichier yaml et remplacer la configuration:
```yaml 
apiVersion:   v1 
kind:   ResourceQuota 
metadata: 
  name:   quota
spec: 
  hard: 
    limits.cpu:   "5" 
    limits.memory:   10Gi 
```

```bash
kubectl replace -f resourcequota.yaml
```

voir : https://kubernetes.io/docs/concepts/policy/resource-quotas/




---------------------------------------------------------------------------------------------------------------
## Les Pods
---------------------------------------------------------------------------------------------------------------
1/ Création d’un fichier manifest (YAML) pour un Pod à un container :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod1
  namespace: tst
spec:
  containers:
  - name: container1
    image: centos
    env:
    - name: MESSAGE
      value: "Hello Word"
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo '$(MESSAGE)'; sleep 10;done"]
    #command: ["printenv"]
    #args: ["HOSTNAME", "KUBERNETES_PORT", "$(MESSAGE)"]
    resources:  #Si absent: error from server (Forbidden) car ResourceQuota present
     limits:
      memory: "200Mi"
      cpu: "1"
     requests: 
      memory: "100Mi"
      cpu: "0.5"
```


2/ Création du Pod à partir du manifest:
```bash
$ kubectl create -f simplepod1.yaml --record --namespace=tst
(--record enregistre la commande en cours dans les annotations. Utile pour une révision ultérieure)
$ kubectl describe pod simplepod1
```
*A la différences des "Multi container pod", les "Single Container Pod" peuvent être créés avec la commande kubctl run.
```bash
$ kubectl run tomcat --image=tomcat:8.0 (obsolete)
```

3/ Afficher les informations sur les Pods en cours d’execution:
```bash
$ kubectl get pod --namespace=tst
$ kubectl get pod --namespace=tst -o wide
$ kubectl get pod simplepod1 --namespace=tst -o wide
 *(-o wide permet d'afficher le Node auquel le pod a été assigné)
 $ kubectl get pod --namespace=tst -o yaml
$ kubectl get pod simplepod1 --namespace=tst -o yaml
 *(-o yaml "--output=yaml" spécifie d’afficher la configuration complette de l'objet)
```

4/ Voir des informations détaillées sur l'histoire et le status du Pod:
```bash
$  kubectl describe pod simplepod1 --namespace=tst
```

5/ Voir le résultat de la commande exécuté dans le conteneur, affichez les journaux du pod:`
```bash
$ kubectl log simplepod1
```

6/ S'attacher à un conteneur en cours d'exécution dans un Pod.
```bash
$ kubectl attach simplepod1
$ kubectl attach simplepod1 -c container1
```

7/ Exécuter une commande dans un conteneur:
 - Récupère le résultat de l'exécution de 'date' à partir du pod "simplepod1"
```bash
$ kubectl exec simplepod1 date
```
 - Récupère le résultat de l'exécution de 'date' dans le conteneur "container1" du pod "simplepod1"
```bash
$ kubectl exec simplepod1 -c container1 date
```
 - Récupérer la stdin de "container1" à partir du simplepod1 :
 ```bash
 $ kubectl exec -it simplepod1 -- /bin/bash
 $ kubectl exec simplepod1 -c container1 -it -- bash -il
```
*Dans le shell, exécutez la commande "printenv" pour répertorier les variables d’environnement.


8/ Supprimez un Pod:
```bash
$ kubectl delete pod simplepod1 --namespace=tst
$ kubectl delete --grace-period=0 --force pod simplepod1 --namespace=tst
 *(Remplacer la valeur de grace par défaut "La valeur 0 force la suppression du Pod")
```


---------------------------------------------------------------------------------------------------------------
## LimitRange & Ressources
---------------------------------------------------------------------------------------------------------------
1/ Créer un objet LimitRange
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limitrange
spec:
  limits:
  - default:
      cpu: 3
      memory: "1000Mi"
    defaultRequest:
      cpu: 1.5
      memory: "500Mi"
    type: Container
```

2/ Créez le LimitRange dans l'espace de noms
```bash
$ kubectl create -f limitrange.yaml --namespace=tst
```

3/ Créer un simplepod3 sans spécifier de ressources:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod3
  namespace: tst
spec:
  restartPolicy: Always
  containers:
  - name: container3
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```   
```bash
$ kubectl create -f limitrange.yaml --namespace=tst
```

Vérifier les ressources attribuer pour le "pod", "limirange", "ResourceQuota" et "namespace":
```bash
$ kubectl describe po simplepod3
$ kubectl describe limitrange
$ kubectl get resourcequota quota --output=yaml 
$ kubectl describe namespaces tst
```

Si un conteneur est créé dans un namespace et qu'il ne spécifie pas ses propres valeurs pour la demande de resources, le conteneur reçoit une demande par défaut correspondant au LimiRange associé a ce namespace.



---------------------------------------------------------------------------------------------------------------
## QoS pod
---------------------------------------------------------------------------------------------------------------

1/ Assigné une classe Guaranteed, Burstable et BestEffort au Pod et afficher le résultat:
```bash
$ kubectl get pod |grep -i qosClass
```



---------------------------------------------------------------------------------------------------------------
## Labels & selector
---------------------------------------------------------------------------------------------------------------
1/ Afficher les labels générées pour chaque objets (pod, namespace, limitrange, resourcequota...) :
```bash
$ kubectl get pod --show-labels --namespace=tst
$ kubectl get namespaces --show-labels --namespace=tst
$ kubectl get limitrange --show-labels --namespace=tst
$ kubectl get resourcequota --show-labels --namespace=tst
```

2/ Attribuer des labels aux objets:
```bash
$ kubectl label pod simplepod1 environment=tst tier=frontend type=pod
$ kubectl label namespace tst environment=tst type=namespace
```


3/ Vérifier la présence des labels sur les objets (pod, namespace, limitrange, resourcequota...):
```bash
$ kubectl get pod --show-labels
```

3/ Effectuer une recherche avec le selector equality-base:
```bash
$ kubectl get pods -l environment=tst,tier=frontend
```
*equality-base permet de filtrer par clé et par valeur. Les objets correspondants doivent satisfaire à tous les labels spécifiées.

4/ Effectuer une recherche avec le selector set-based:
```bash
$ kubectl get pods -l 'environment in (tst),tier in (frontend)'
```

5/ Supprimer un objet par une recherche avec le selector equality-base:
```bash
$ kubectl describe po simplepod1
$ kubectl delete pods -l environment=tst,tier=frontend
```

6/ Attacher une anotation à un Pod:
```bash
$ kubectl annotate pods simplepod3 description='my frontend'
```

7/ Recréer plusieurs Pods avec des nom, labels et anotations différentes 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod1
  namespace: tst
  annotations:
    description: my frontend
  labels:
    environment: "tst"
    tier: "frontend"
...
```  

---------------------------------------------------------------------------------------------------------------
## Affectation Pod
---------------------------------------------------------------------------------------------------------------
1/ répertoriez les nodes disponibles sur le cluster:
```bash
$ kubectl get nodes 
```

2/ Ajouter les labels au node:
```bash
$ kubectl label nodes node1 affect=node1
```

3/ Vérifiez que le node possède bien le nouveau label : 
```bash
$  kubectl get nodes --show-labels 
```

4/ Créer un pod "simplepod4" planifié sur le node choisi grace au sélecteur de node (node qui a un label label1=var1):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod4
  namespace: tst
spec:
  restartPolicy: Never
  nodeSelector:
    affect: node1
  containers:
  - name: container4
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```
```bash
$ kubectl create -f simplepod4.yaml
```

5/ Vérifiez que le pod est bien en cours d'exécution sur le node choisi:
```bash
$ kubectl get pods --output=wide 
```



---------------------------------------------------------------------------------------------------------------
## TAINTS:
---------------------------------------------------------------------------------------------------------------

1/ Ajoutez un taints à un node:
```bash
$ kubectl taint nodes node1 Mykey=Myvalue:NoSchedule
$ kubectl describe no node1 | grep Taints
```
*L'affectation "Mykey=Myvalue:NoSchedule" au node signifie qu'aucun pod ne pourra être planifier sur node1, à moins d'avoir une tolérance correspondante.


2/ Lancer un nouveau pod "simplepod5" sur le node1 et vérifier le status:
```bash
$ kubectl describe po  simplepod5
```

3/ Spécifiez l'une des tolérances ci-dessous dans le PodSpec du simplepod5 (Operateur: Equal ou Exist) afin que celui-ci soit capable d'être programmé sur node1 :
```yaml
spec:
  tolerations:
  - key: "Mykey"
    operator: "Equal"
    value: "Myvalue"
    effect: "NoSchedule"

ou

tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```
*Une tolérance "correspond" à une taint si les clés sont les mêmes et les effets sont les mêmes
*Deux cas spéciaux:
 - Une key vide avec opérateur "Exists" correspond à toutes les clés, valeurs et effets, ce qui signifie que tout sera toléré.
```yaml
tolerations:
- operator: "Exists"
```

 - Un effect vide correspond à tous les effets avec la key “Mykey” .
```yaml
tolerations:
- key: "Mykey"
  operator: "Exists"
```


2/ Supprimer l'altération sur le node:
```bash
$ kubectl taint nodes node1 Mykey:NoSchedule-
```


Voir: https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/


---------------------------------------------------------------------------------------------------------------
## Les sondes (probes)
---------------------------------------------------------------------------------------------------------------
1/ Définir un pod avec une sonde d'activité qui utilise une requête EXEC:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod6
  namespace: tst
spec:
  containers:
  - name: container6
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "touch /tmp/test; sleep 60; rm -rf /tmp/healthy; sleep 60"]
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/test
      initialDelaySeconds: 10
      periodSeconds: 5
```
- periodSeconds: spécifie que kubelet doit effectuer une sonde d'activité (cat /tmp/tst) toutes les 5 secondes (La valeur minimale est 1).
- initialDelaySeconds: indique a kubelet qu'il doit attendre 10 secondes avant d'effectuer la première sonde. 
- command: kubelet exécute la commande "cat /tmp/test" dans le conteneur.  Si la commande réussit, elle renvoie 0

Vérifier le status du pod:
```bash
$ kubectl describe po simplepod6
$ tail -f /var/log/message | grep -i simplepod6
```
*kubelet tue le conteneur et il est soumis à sa politique de redémarrage. La sortie "kubectl get" montre que RESTARTS a été incrémenté:


 
 
Ajouter les options suivantes:
 - timeoutSeconds: Nombre de secondes après lequel la sonde arrive à expiration (Valeur minimal par défaut "1sce").
 - successThreshold: Succès consécutifs minimum pour que la sonde soit considérée comme ayant réussi après avoir échoué. La valeur minimal par défaut est 1 (doit être 1 pour la vivacité).
 - failureThreshold: Quand un pod démarre et que la sonde échoue, Kubernetes essaiera le seuil d'échec avant d'abandonner. Abandonner en cas d’analyse signifie relancer le pod. En cas de test de disponibilité, le pod sera marqué comme étant non prêt. La valeur par défaut est 3. La valeur minimale est 1.



2/ Définir un nouveau pod avec une sonde d'activité qui utilise une requête HTTPGET:
```yaml
apiVersion: v1
apiVersion: v1
kind: Pod
metadata:
  name: simplepod7
  namespace: tst
spec:
  containers:
  - name: container7
    image: nginx
    imagePullPolicy: Always
    #command: 
    #args: 
    livenessProbe:
      httpGet:
        #path: /usr/share/nginx/html/index.html
        port: 80
        #httpHeaders:
        #- name: X-Custom-Header
        #  value: Awesome
      initialDelaySeconds: 10
      periodSeconds: 5
```
Les sondes HTTP ont des champs supplémentaires qui peuvent être définis :
- host : nom d'hôte auquel se connecter, par défaut l'adresse IP du pod. 
- scheme : Schéma à utiliser pour se connecter à l'hôte (HTTP ou HTTPS). Par défaut à HTTP.
- path : Chemin d'accès sur le serveur HTTP.
- httpHeaders : en-têtes personnalisés à définir dans la requête. HTTP permet des en-têtes répétés.
- port : nom ou numéro du port auquel accéder sur le conteneur. Le nombre doit être compris entre 1 et 65535.


3/ Définir une sonde d'activité qui utilise une requête TCPSocket:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```
- Cet exemple utilise à la fois des sondes de disponibilité (Readiness) et de vivacité (Liveness). 
- kubelet envoie la première sonde de disponibilité 5 secondes après le démarrage du conteneur. 
- Il tente de se connecter au Pod sur le port 8080. Si la sonde réussit, le pod sera marqué comme prêt. 
- kubelet continuera à exécuter cette vérification toutes les 10 secondes.

- kubelet lance la première sonde de vivacité 15 secondes après le début du conteneur. 
- Tout comme la sonde de disponibilité, il tente de se connecter au conteneur goproxy sur le port 8080. 
- Si la sonde d'activité échoue, le conteneur sera redémarré.



4/ Utiliser un port nommé:
Utiliser un ContainerPort nommé pour les contrôles d'activité HTTP ou TCP:
```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
```

Voir: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/




---------------------------------------------------------------------------------------------------------------
##  volumes
---------------------------------------------------------------------------------------------------------------
###Volume emptyDir:

1/ Répertoriez les StorageClasses du cluster:
```bash
$ kubectl get storageclass
```

2/ Définir un volume emptyDir dans le Pod et le monter dans le container avec une limite.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod9
  namespace: tst
spec:
  containers:
  - image: nginx
    name: container9
    volumeMounts:
    - mountPath: /myvolume
      name: monvolume
    #ressources:
      #limits:
      #memory: "200Mi"
      #ephemeral-storage: 2GiB
      #requests: 
      #ephemeral-storage: 1GiB
  volumes:
  - name: monvolume
    emptyDir: {}
```
   
3/ Créer le Pod puis dans un autre terminal, lancer un shell sur le conteneur:
```bash
$ kubectl exec -it simplepod9 -- /bin/bash
```

4/ Dans le terminal d'origine, surveillez les modifications apportées
$ find / -name myvolume

### PV & PVC:
1/ Créer un "PersistentVolume"
```bash
$  mkdir /mnt/data 
```

2/ Dans le répertoire /mnt/data , créez un fichier index.html :
```bash
$ echo 'test volume persistant' > /mnt/data/index.html
```

3/ Créer un objet PersistentVolume à partir d'un nouveau manifest :
```yaml  
	kind: PersistentVolume
	apiVersion: v1
	metadata:
	  name: mypersvol
	  labels:
	    type: local
	spec:
	  storageClassName: manual
	  capacity:
	    storage: 10Gi
	  accessModes:
	    - ReadWriteOnce
	  hostPath:
	    path: "/mnt/data"
```
*("StorageClass"  sur manual pour un PersistentVolume est utilisé pour lier les requêtes PersistentVolumeClaim à ce PersistentVolume)
```bash
$ kubectl create –f pv.yaml
```

4/ Afficher des informations (status) sur le PersistentVolume:
```bash
$ kubectl get pv task-pv-volume
$ kubectl describe pv task-pv-volume
```

5/ Crée un PersistentVolumeClaim, qui sera automatiquement lié au PersistentVolume et qui demande un volume d'au moins trois gibibytes pouvant fournir un accès en r/w pour au moins un node:
```yaml
	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: mypersvolclaim
	spec:
	  storageClassName: manual
	  accessModes:
	    - ReadWriteOnce
	  resources:
	    requests:
	      storage: 3Gi
```
```bash
$ kubectl create –f pvc.yaml
$ kubectl get pvc
```


6/ Regardez à nouveau le PersistentVolume et vérifier que la sortie montre que PersistentVolumeClaim est lié à votre PersistantVolume:
```bash
$  kubectl get pv task-pv-volume
$ kubectl describe pv pv0001
```

7/ Créer un pod qui utilise PersistentVolumeClaim comme volume de stockage.
```yaml
...
	spec:
	  volumes:
	  - name: volume1
	    persistentVolumeClaim:
	    claimName: mypersvolclaim
	  containers:
	   ...
	    volumeMounts:
	    - mountPath: "/var/www/monsite"
	      name: volume1
```
Dans le code ci-dessus, nous avons défini -
∙ volumeMounts: → Il s'agit du chemin dans le conteneur sur lequel le montage aura lieu.
∙ Volume: → Cette définition définit la définition de volume que nous allons réclamer.
∙ persistentVolumeClaim: → Sous cela, nous définissons le nom du volume que nous allons utiliser dans le module défini.


8/ Obtenez un shell sur le conteneur et vérifiez le montage:
```bash
$  kubectl exec -it task-pv-pod -- /bin/bash 
$  kubectl get pod task-pv-pod 
```



### Projected Volume

1/ Créer un pod pour utiliser un "Projected Volume" et pour monter les secrets (ci-dessous) dans un même répertoire partagé:
```yaml
	spec:
	  containers:
	  ...
	    volumeMounts:
	    - name: all-in-one
	      mountPath: "/projected-volume"
	      readOnly: true
	  volumes:
	  - name: all-in-one
	    projected:
	      sources:
	      - secret:
		  name: user
	      - secret:
		  name: pass
```

2/ Observez les modifications apportées au pod:
```bash
$  kubectl get --watch pod test-projected-volume 
```

3/ Dans un autre terminal vérifiez que le répertoire contient vos sources projetées
```bash
$ kubectl exec -it test-projected-volume -- /bin/sh 
$ ls /projected-volume/ 
```



---------------------------------------------------------------------------------------------------------------
## Contrôle d'accès:
---------------------------------------------------------------------------------------------------------------
Le stockage configuré avec un ID de groupe (GID) permet d'écrire uniquement par Pods en utilisant le même GID. Les GID non concordants ou manquants provoquent des erreurs d'autorisation refusées. Pour réduire le besoin de coordination avec les utilisateurs, un administrateur peut annoter un PersistentVolume avec un GID. Ensuite, le GID est automatiquement ajouté à tout Pod qui utilise le PersistentVolume.

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv1
  annotations:
    pv.beta.kubernetes.io/gid: "1234"
```    
Quand un Pod consomme un PersistentVolume qui a une annotation GID, le GID annoté est appliqué à tous les Conteneurs dans le Pod de la même manière que les GID spécifiés dans le contexte de sécurité du Pod.




---------------------------------------------------------------------------------------------------------------
## Les secrets
---------------------------------------------------------------------------------------------------------------

### Créer des secrets:

Il existe de nombreuses façons de créer des secrets dans Kubernetes.

1/ méthode 1: Création à partir de fichiers txt
-Créer des fichiers contenant le nom d'utilisateur et mot de passe:
```bash
$ echo -n "admin" > ./username.txt
$ echo -n "azerty" > ./password.txt
ou
echo -n '123456dfg45' | base64 >> ./password.txt
```
- Emballez ces fichiers dans des secrets:
```bash
$ kubectl create secret generic user --from-file=./username.txt
$ kubectl create secret generic pass --from-file=./password.txt
```

2/ mMéthode 2: Créer à partir d'une commande k8s
```bash
$ kubectl create secret generic monsecret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb'
```

3/ méthode 3: Créer à partir d'un fichier yaml
-Créer l'object secret à partir du fichier yaml:
```yaml
	apiVersion: v1
	kind: Secret
	metadata:
	  name: monsecret
	data:
	  username: admin
	  password: azerty
```

```bash
$ kubectl create –f Secret.yaml
```

4/ Afficher des informations sur le secret:
```bash
$  kubectl get secret  
$  kubectl describe secret monsecret -o yaml
```

### Utiliser des secrets
Une fois que nous avons créé les secrets, il peut être consommé dans un pod ou un contrôleur en tant que: 
 - Variable d'environnement
 - volume

5/ Créer un pod qui accède aux secret via un volume (tous les fichiers créés sur montage secret auront l'autorisation 0400):
```yaml
	...
	spec:
	  containers:
	    ...
	    volumeMounts:
	    - name: secret-volume
	      mountPath: /mnt/secret-volume
	      readOnly: true
	  volumes:
	    - name: secret-volume
	      secret:
	       secretName: monsecret
	       defaultMode: 256
```

6/ Spécifier un chemin particulier pour un item (/mnt/secret-volume/my-group/my-username à la place de /mnt/secret-volume/username) et spécifier des autorisations différentes pour différents fichiers (ici, la valeur d'autorisation de 0777): 

```yaml
	volumes:
	  - name: foo
	    secret:
	      secretName: mysecret
	      items:
	      - key: username
		path: my-group/my-username
		mode: 511
```

- Si spec.volumes[].secret.items est utilisé, seules les clés spécifiées dans les items sont projetées. 
- Pour consommer toutes les clés du secret, elles doivent toutes être répertoriées dans le champ des items.
- Toutes les clés listées doivent exister dans le secret correspondant. Sinon, le volume n'est pas créé.


7/ Créer un pod qui a accès aux secret via des variables d'environnement:
Afin d'utiliser la variable secrète comme variable d'environnement, nous utiliserons env dans la section spec du fichier pod yaml.
```yaml
spec:
  containers:
   ...
   env:
   - name: ENVSECRET1
     valueFrom:
      secretKeyRef:
       name: usersecret
       key: username
   - name: ENVSECRET2
     valueFrom:
       secretKeyRef:
       name: passsecret
       key: password
```


---------------------------------------------------------------------------------------------------------------
## Security Context
---------------------------------------------------------------------------------------------------------------
1/ Créer un Pod et configurer un contexte de sécurité (mode privilége) pour l'ensemble de ses pods:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod6
  namespace: tst
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
    #seLinuxOptions:
      #level: "s0:c123,c456"
      #supplementalGroups: [5678]
  containers:
  - name: container6
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```
*runAsUser: spécifie que pour tous les conteneur du Pod, le premier processus s'exécute avec l'ID utilisateur 1000. 
*fsGroup: spécifie que l'ID de groupe 2000 est associé à tous les Conteneurs du Pod (L'ID de groupe est également associé au volume monté).


2/ Obtenez un shell et dressez la liste des processus en cours pour vérifier que les processus s'exécutent en tant qu'utilisateur 1000
```bash
$  kubectl exec -it simplepod6 -- sh 
$  ps aux 
```

3/ Créer un fichier dans le répertoire de votre volume et vérifier la valeur de l'ID de groupe

4/ affichez les capacités du processus 1:
```bash
$ cd /proc/1
```


5/ Faites de même pour un contenair:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simplepod6
  namespace: tst
spec:
  containers:
  - name: container6
    image: centos
    securityContext:
      privileged: true
      runAsUser: 2000
      seLinuxOptions:
        level: "s0:c123,c456"
      allowPrivilegeEscalation: false
      capabilities:
	add: ["NET_ADMIN", "SYS_TIME"]
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```



---------------------------------------------------------------------------------------------------------------
##Controller
---------------------------------------------------------------------------------------------------------------

###Replication Controller
Avant que le déploiement et ReplicaSet soient ajoutés à Kubernetes, les applications répliquées étaient configurées à l'aide d'un ReplicationController.

1/ définiser un type Replication Controller ontrôleur:
```yaml
apiVersion: v1
kind: ReplicationController 
metadata:
   name: Name-ReplicationController
spec:
   replicas: 3 
   template:
      metadata:
         name: Name-ReplicationController
      labels:
         app: App
         component: neo4j
      spec:
         containers:
         - name: Nmae_Cntainer
         image: tomcat: 8.0
         ports:
            - containerPort: 7474 
```  

2/ Afficher les détails du contrôleur de réplication.:
```bash
$ kubectl get rc
```   
Voir comment effectuer une mise à jour progressive à l'aide d'un contrôleur de réplication:
https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/



###ReplicaSet

La principale différence entre le réplica Set et le Replication Controller est que le Replication Controller ne prend en charge que le sélecteur equality-based, alors que le jeu de réplicas prend en charge le sélecteur set-based selector.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: MyReplicaSet
spec:
  replicas: 3
  selector:
    matchLables:
      tier: Backend
    matchExpression:
      - { key: tier, operation: In, values: [Backend]}
  template:
    metadata:
      lables:
        app: Tomcat-ReplicaSet
        tier: Backend
    spec:
      containers:
      - name: Tomcat
        image: tomcat: 8.0
      ports:
      - containerPort: 7474
 ```  
*un modèle de pod dans un ReplicaSet doit spécifier les labels appropriées et une stratégie de redémarrage (la seule valeur autorisée est Always).
* Un ReplicaSet gère tous les pods avec des labels correspondant au sélecteur. Il ne fait pas la distinction entre les pods créés ou supprimés et les pods créés ou supprimés par une autre personne ou un autre processus. Cela permet de remplacer le ReplicaSet sans affecter les pods en cours d'exécution.
*vous ne devez pas créer de pod dont les labels correspondent à ce sélecteur, ni directement, ni avec un autre ReplicaSet, ni avec un autre contrôleur, tel qu'un déploiement. Si vous le faites, le ReplicaSet pense avoir créé les autres pods. Kubernetes ne vous empêche pas de le faire.
* Pour les Labels, veillez à ne pas les chevaucher avec d'autres contrôleurs.


2/ Afficher les détails du ReplicaSet:
```bash
$ kubectl describe rs/MyReplicaSet
```   


---------------------------------------------------------------------------------------------------------------
## SERVICES
---------------------------------------------------------------------------------------------------------------
1/ créer un service:
```yaml
apiVersion: v1
kind: Service
metadata:
   name: Name_Service
spec:
   selector:
      application: "My Application"  (*falcultatif)
   ports:
   - port: 8080
   targetPort: 31999
```
*Dans cet exemple, nous avons un sélecteur; Pour transférer le trafic, nous devons donc créer manuellement un noeud final.

2/ créer un EndPoint qui acheminera le trafic vers le noeud final défini comme "192.168.168.40:8080".
```yaml
apiVersion: v1
kind: Endpoints
metadata:
   name: Tutorial_point_service
subnets:
   address:
      "ip": "192.168.168.40" -------------------> (Selector)
   ports:
      - port: 8080
```

3/ Créer un service multi-ports:
```yaml
piVersion: v1
kind: Service
metadata:
   name: Tutorial_point_service
spec:
   selector:
      application: “My Application”
   ClusterIP: 10.3.0.12
   ports:
      -name: http
      protocol: TCP
      port: 80
      targetPort: 31999
   -name:https
      Protocol: TCP
      Port: 443
      targetPort: 31998
```      
*CLUSTERIP: aide à limiter le service au sein du cluster. Il expose le service au sein du cluster Kubernetes défini.      
      
      
4/ Créer un service complet avec le type de service NodePort: 
```yaml
apiVersion: v1
kind: Service
metadata:
   name: appname
   labels:
      k8s-app: appname
spec:
   type: NodePort
   ports:
   - port: 8080
      nodePort: 31999
      name: omninginx
   selector:
      k8s-app: appname
      component: nginx
      env: env_name
```   



---------------------------------------------------------------------------------------------------------------
##Controller Deployment
---------------------------------------------------------------------------------------------------------------
La méthode préférée pour créer une application répliquée consiste à utiliser un déploiement, qui à son tour utilise un ReplicaSet.

1/ Exécuter une application à l'aide d'un objet Kubernetes Deployment.
```yaml
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template: # create pods using pod definition in this template
    metadata:
      # unlike pod-nginx.yaml, the name is not included in the meta data as a unique name is
      # generated from the deployment name
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

2/ Créez un déploiement basé sur le fichier YAML:
```bash
$ kubectl apply -f https://k8s.io/docs/tasks/run-application/deployment.yaml
```
Creation le controller de type ‘deployment’:
$ kubectl run Name-Pod –image=Image-registry:tag 


3/ Afficher des informations sur le déploiement:
```bash
$ kubctl get deployments
$ kubectl describe deployment nginx-deployment 
```

4/ Vérifier l'état du déploiement
```bash
$ kubectl rollout status deployment/nginx-deployment
```

5/ Répertoriez les modules créés par le déploiement:
```bash
$  kubectl get pods -l app=nginx 
```

6/ Afficher des informations sur un pod:
```bash
$  kubectl describe pod <pod-name> 
```

7/ Mise à jour du déploiement par l'application d'un nouveau fichier YAML. 
```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8 # Update the version of nginx from 1.7.9 to 1.8
        ports:
        - containerPort: 80
```

Appliquez le nouveau fichier YAML:
```bash
$ kubectl apply -f deployment-update.yaml 
ou 
$ kubectl set image deployment/Deployment tomcat=tomcat:6.0
```

8/ Regardez le déploiement créer des pods avec de nouveaux noms et supprimer les anciens pods:
```bash
$  kubectl get pods -l app=nginx 
```

9/ Scaling de l'application en augmentant le nombre de réplicas (Pods):
```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4 # Update the replicas from 2 to 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
```

Appliquer les changements et vérifier les résultat:
```bash
kubectl apply -f deployment-scale.yaml
$ kubectl get pods -l app=nginx
```

10/ Supprimer un déploiement:
```bash
$  kubectl delete deployment nginx-deployment 
```

11/ Voir comment exécuter une application avec état à instance unique à l'aide de PersistentVolume et d'un déploiement.:
https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/

12/ Voir comment exécuter une application avec état répliquée à l'aide d'un contrôleur StatefulSet. L'exemple est une topologie mono-maître MySQL avec plusieurs esclaves exécutant une réplication asynchrone. Notez qu'il ne s'agit pas d'une configuration de production. 
https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/


Autre méthode:
supposons que nous souhaitons maintenant mettre à jour les modules nginx pour utiliser l'image nginx:1.9.1:
```bash
 $ kubectl set image deployment/nginx-deployment nginx = nginx:1.9.1 deployment "nginx-deployment" image updated 
 ```
Alternativement, nous pouvons edit le Déploiement et changer:
```bash
$ kubectl edit deployment/nginx
```

Pour voir l'état du déploiement, exécutez: 
```bash
$ kubectl rollout status deployment/nginx-deployment
```

Revenir au déploiement précédent
```bash
$ kubectl rollout undo deployment/Deployment –to-revision=2
```

La prochaine fois que nous voulons mettre à jour ces Pods, nous avons seulement besoin de mettre à jour le template de pod de Deployment.
voir 

Utiliser un correctif de fusion stratégique pour mettre à jour un déploiement:
https://kubernetes.io/docs/tasks/run-application/update-api-object-kubectl-patch/



---------------------------------------------------------------------------------------------------------------
## NETWORK POLICY:
---------------------------------------------------------------------------------------------------------------

$ kubectl create deployment nginx --image=nginx

*Un podSelector vide sélectionne tous les pods de l'espace de noms.
* chaque politique NetworkPolicy comprend une liste policyTypes pouvant inclure Ingress , Egress ou les deux.Si aucun type de policyTypes n'est spécifié sur un NetworkPolicy, par défaut, Ingress sera toujours défini et Egress sera défini si NetworkPolicy a des règles de sortie.

```yaml
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
   name: policyfrontend
   namespace: tst
spec:
   podSelector:
      matchLabels:
         role: backend
  policyTypes: 
  -   Ingress 
  -   Egress 
  ingress:	 
   - from:
      - podSelector:
         matchLabels:
            role: db
   ports:
      - protocol: TCP
         port: 6379
egress: 
  -   to: 
    -   ipBlock: 
        cidr:   10.0.0.0/24 
    ports: 
    -   protocol:   TCP 
      port:   5978
```

- La NetworkPolicy du nom de "policyfrontend", isole les pods identifiés par le label "role=frontend" dans le namespace "tst" pour le trafic entrant "Ingress" et sortant "Egress". Sur l'ensemble des pods selectionné, elle identifie ceux contenant le label "role=db" et leurs autorise les connexions entrante sur le port TCP 6379.

voir: https://kubernetes.io/docs/concepts/services-networking/network-policies/





---------------------------------------------------------------------------------------------------------------
##AutoScale
---------------------------------------------------------------------------------------------------------------
Mettre à l'échelle automatiquement les pods définis tels que Déploiement, Replica Set, Replication Controller.
```bash
$ kubectl autoscale (-f FILENAME | TYPE NAME | TYPE/NAME) [--min = MINPODS] --max = MAXPODS [--cpu-percent = CPU] [flags]
$ kubectl autoscale deployment foo --min=2 --max=10
```


---------------------------------------------------------------------------------------------------------------
##JOB
---------------------------------------------------------------------------------------------------------------

1/ Creer un Job
```yaml
apiVersion: v1
kind: Job   (création d’un Job de type Pod)
metadata:
   name: py
   spec:
   template:
      metadata
      name: py -------> 2
      spec:
         containers:
            - name: py ------------------------> 3
            image: python----------> 4
            command: ["python", "SUCCESS"]
            restartPocliy: Never --------> 5
```
∙ kind: Job → Nous avons défini le type de Job qui indiquera à kubectlt que le fichier yaml utilisé doit créer un module de type de travail.
∙ Nom: py → C'est le nom du modèle que nous utilisons et la spécification définit le modèle.
∙ nom: py → nous avons donné le nom py dans les spécifications de conteneur, ce qui permet d'identifier le pod qui sera créé.
Image: python → l'image que nous allons extraire pour créer le conteneur qui s'exécutera à l'intérieur du pod.
∙ restartPolicy: Jamais → Cette condition de redémarrage de l'image est donnée comme jamais, ce qui signifie que si le conteneur est tué ou s'il est faux, il ne redémarrera pas tout seul.


2/ SCréer un scheduled Job
```yaml
apiVersion: v1
kind: Job
metadata:
   name: py
spec:
   schedule: h/30 * * * * ? -------------------> 1
   template:
      metadata
         name: py
      spec:
         containers:
         - name: py
         image: python
         args:
/bin/sh -------> 2
-c
ps –eaf ------------> 3
restartPocliy: OnFailure
```

 - hschedule: Pour planifier l'exécution du Job toutes les 30 minutes.
 - /bin/sh: entrer dans le conteneur avec /bin/sh
 - ps –eaf: Exécute la commande ps -eaf sur la machine et répertorie tous les processus en cours d'exécution dans un conteneur.
 
 
Pour lister tous les pods appartenant à un job sous une forme lisible par une machine:
```bash
$ pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath={.items..metadata.name})
$ echo $pods
```

L'option --output=jsonpath spécifie une expression qui récupère juste le nom de chaque pod dans la liste renvoyée.
```bash
$ kubectl logs $pods
```


