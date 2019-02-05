---------------------------------------------------------------------------------------------------------------
# Stateful à instance unique
---------------------------------------------------------------------------------------------------------------
Exécuter une application Stateful à instance unique:
exécuter une application stateful à instance unique dans Kubernetes en utilisant un volume PersistentVolume et un déploiement. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

lister les pods:
```
$ kubectl get pods -l app=mysql
```

Inspect the PersistentVolumeClaim:
```
$ kubectl describe pvc mysql-pv-claim
```


Accéder à l'instance MySQL:
Le fichier YAML précédent crée un service qui permet à d'autres modules du cluster d'accéder à la base de données. 
L'option de service clusterIP: None permet au service DNS de se résoudre directement en adresse IP du pod. 
C'est optimal quand vous n'avez qu'un Pod derrière un Service et que vous n'avez pas l'intention d'augmenter le nombre de Pods.

Exécutez un client MySQL pour vous connecter au serveur:
```
$  kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -ppassword 
```

Cette commande crée un nouveau Pod dans le cluster qui exécute un client MySQL et le connecte au serveur via le Service. S'il se connecte, vous savez que votre base de données MySQL avec état est en cours d'exécution.
Waiting for pod default/mysql-client-274442439-zyp6i to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.
mysql>




*Utiliser la strategy: type: Recreate dans le fichier YAML de configuration de déploiement. Cela indique à Kubernetes de ne pas utiliser les mises à jour tournantes. Les mises à jour tournantes ne fonctionneront pas, car vous ne pouvez pas faire tourner plus d'un pod à la fois. La stratégie Recreate arrêtera le premier pod avant d'en créer un nouveau avec la configuration mise à jour.




Supprimer un déploiement:
```
$ kubectl delete deployment,svc mysql
$ kubectl delete pvc mysql-pv-claim
```
Si vous avez manuellement provisionné un PersistentVolume, vous devez également le supprimer manuellement et libérer la ressource sous-jacente. Si vous avez utilisé un provisionneur dynamique, il supprime automatiquement le volume PersistentVolume lorsqu'il voit que vous avez supprimé PersistentVolumeClaim. 




---------------------------------------------------------------------------------------------------------------
## application Stateful répliquée
---------------------------------------------------------------------------------------------------------------
Exécuter une application Stateful répliquée:
Cette page montre comment exécuter une application dynamique répliquée à l'aide d'un contrôleur StatefulSet . L'exemple est une topologie à un seul maître MySQL avec plusieurs esclaves exécutant une réplication asynchrone.

L'exemple de déploiement de MySQL se compose d'un fichier ConfigMap, de deux services et d'un StatefulSet.
Créez le fichier ConfigMap :

```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # Apply this config only on the master.
    [mysqld]
    log-bin
  slave.cnf: |
    # Apply this config only on slaves.
    [mysqld]
    super-read-only

```

Ce ConfigMap fournit des substitutions my.cnf qui vous permettent de contrôler indépendamment la configuration sur le maître MySQL et les esclaves. Dans ce cas, vous souhaitez que le maître puisse servir les journaux de réplication aux esclaves et que vous souhaitiez que les esclaves rejettent les écritures qui ne proviennent pas de la réplication.

Chaque Pod détermine la portion à regarder dans le configmap lors de l'initialisation, en fonction des informations fournies par le contrôleur StatefulSet.



Créez les services :

```yaml
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the master: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```

Le service Headless héberge les entrées DNS que le contrôleur StatefulSet crée pour chaque pod faisant partie de l'ensemble. Comme le service Headless est nommé mysql , les pods sont accessibles en résolvant <pod-name>.mysql depuis n'importe quel autre pod du même cluster et espace-noms Kubernetes.

Le service client, appelé mysql-read , est un service normal avec son propre IP de cluster qui distribue les connexions sur tous les pods MySQL déclarant être prêts. L'ensemble des points d'extrémité potentiels comprend le maître MySQL et tous les esclaves.

Notez que seules les requêtes en lecture peuvent utiliser le service client à charge équilibrée. Parce qu'il n'y a qu'un seul maître MySQL, les clients doivent se connecter directement au Pod principal MySQL (via son entrée DNS dans le Headless Service) pour exécuter des écritures.


```yaml
StatefulSet:
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on master (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing slave.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from master. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi

          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```


Vous pouvez regarder la progression du démarrage en exécutant:
$ kubectl get pods -l app=mysql --watch


Ce manifeste utilise une variété de techniques pour gérer les Pods dynamiques dans le cadre d'un StatefulSet. 


Compréhension de l'initialisation dynamique du Pod:
Le contrôleur StatefulSet démarre les Pods une à la fois, dans l'ordre par leur index ordinal. Il attend que chaque Pod indique être prêt avant de commencer le suivant.De plus, le contrôleur attribue à chaque Pod un nom unique et stable de la forme <statefulset-name>-<ordinal-index> . 

Générer la configuration:


https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/
