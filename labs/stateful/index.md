# StatefulSets
StatefulSets manage the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods, suitable for applications that require one or more of the following.
* Stable, unique network identifiers
* Stable, persistent storage
* Ordered, graceful deployment and scaling
* Ordered, automated rolling updates

In this lab you are deploying a MySQL database using `StatefulSet` and Azure Managed Disks as a PersistentVolume.

### Setup
Create a working directory for all of our manifest files.
```sh
mkdir -p ${HOME}/environment/azure_statefulset
cd ${HOME}/environment/azure_statefulset
```

### Create the mysql Namespace
We will create a new Namespace called `mysql` that will host all the components.

```sh
kubectl create namespace mysql
```

### Create ConfigMap
A ConfigMap allows you to decouple configuration artifacts and secrets from image content to keep containerized applications portable. Using ConfigMaps, you can independently control the MySQL configuration.

Run the following commands to create the ConfigMap.

```sh
cat << EoF > ${HOME}/environment/azure_statefulset/mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # Apply this config only on the leader.
    [mysqld]
    log-bin
  slave.cnf: |
    # Apply this config only on followers.
    [mysqld]
    super-read-only
EoF
```

The ConfigMap stores `master.cnf`, `slave.cnf` and passes them when initializing leader and follower pods defined in `StatefulSet`:
* **master.cnf** is for the MySQL leader pod which has binary log option (log-bin) to provides a record of the data changes to be sent to follower servers.
* **slave.cnf** is for follower pods which have super-read-only option.

Create `mysql-config` ConfigMap.
```sh
kubectl apply -f ${HOME}/environment/azure_statefulset/mysql-configmap.yaml
```

### Create Services
Services can be exposed in different ways by specifying a `type` in the `serviceSpec`. `StatefulSet` currently requires a Headless Service to control the domain of its Pods, directly reach each Pod with stable DNS entries.

By specifying **"None"** for the clusterIP, you can create a Headless Service.

Create the mysql services file:
```sh
cat << EoF > ${HOME}/environment/azure_statefulset/mysql-services.yaml
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
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
# For writes, you must instead connect to the leader: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
EoF
```

You can see the **mysql** service is for DNS resolution so that when pods are placed by StatefulSet controller, pods can be resolved using ``pod-name.mysql``. **mysql-read** is a client service that does load balancing for all followers.

Create service `mysql` and `mysql-read` by executing the following command
```sh
kubectl apply -f ${HOME}/environment/azure_statefulset/mysql-services.yaml
```

### Create StatefulSet
StatefulSet consists of serviceName, replicas, template and volumeClaimTemplates:

* **serviceName** is "mysql", headless service we created in previous section
* **replicas** is 3, the desired number of pod
* **template** is the configuration of pod
* **volumeClaimTemplates** is to claim volume for pod

Create the StatefulSet manifest:
```sh
cat << 'EoF' > ${HOME}/environment/azure_statefulset/mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: mysql
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 2
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
          [[ `uname -n` =~ -([0-9]+)$ ]] || exit 1
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
          # Skip the clone on leader (ordinal index 0).
          [[ `uname -n` =~ -([0-9]+)$ ]] || exit 1
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
            command: 
            - sh
            - -c
            - "[ ! -f /tmp/mysql-not-ready ] && mysql -h 127.0.0.1 -e 'SELECT 1'"
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
            # because we're cloning from an existing follower.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from leader. Parse binlog position.
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
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-premium
      resources:
        requests:
          storage: 10Gi
EoF
```

Note that we're using the `managed-premium` storage class, which is available in AKS for Azure Managed Disks.

Apply the StatefulSet:
```sh
kubectl apply -f ${HOME}/environment/azure_statefulset/mysql-statefulset.yaml
```

Watch StatefulSet deployment status 
```sh
kubectl -n mysql rollout status statefulset mysql
```

It will take few minutes for pods to initialize and the `StatefulSet`  to be created.

Output:
```
Waiting for 2 pods to be ready...
Waiting for 1 pods to be ready...
partitioned roll out complete: 2 new pods have been updated...
```

Open another terminal and watch the progress of pods creation using the following command.
```sh
kubectl -n mysql get pods -l app=mysql --watch
```

You can see ordered, graceful deployment with a stable, unique name for each pod.

```
NAME      READY   STATUS           RESTARTS    AGE
mysql-0   0/2     Init:0/2          0          16s
mysql-0   0/2     Init:1/2          0          17s
mysql-0   0/2     PodInitializing   0          18s
mysql-0   1/2     Running           0          19s
mysql-0   2/2     Running           0          25s
mysql-1   0/2     Pending           0          0s
mysql-1   0/2     Pending           0          0s
mysql-1   0/2     Init:0/2          0          0s
mysql-1   0/2     Init:1/2          0          10s
mysql-1   0/2     PodInitializing   0          11s
mysql-1   1/2     Running           0          12s
mysql-1   2/2     Running           0          16s
```

Press `Ctrl+C` to stop watching.

Check the dynamically created PVC
```sh
kubectl -n mysql get pvc -l app=mysql
```

We can see `data-mysql-0`, and `data-mysql-1` have been created.

Output:
```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
data-mysql-0   Bound    pvc-2a9bb222-3fbe-11ea-94be-0aff3e98c5a0   10Gi       RWO            managed-premium   22m
data-mysql-1   Bound    pvc-47076f1d-3fbe-11ea-94be-0aff3e98c5a0   10Gi       RWO            managed-premium   21m
```

### Test MySQL
You can use **mysql-client** to send some data to the leader, **mysql-0.mysql** by running the following command.

```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello, from mysql-client');
EOF
```

Run the following to test follower `mysql-read` received the data.

```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -it --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"
```

Output:
```
+--------------------------+
| message                  |
+--------------------------+
| hello, from mysql-client |
+--------------------------+
```

To test load balancing across followers, run the following command.
```sh
kubectl -n mysql run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```

Each MySQL instance is assigned a unique identifier, and it can be retrieved using `@@server_id`. It will print the server id serving the request and the timestamp.

```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2021-02-21 19:17:52 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2021-02-21 19:17:53 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2021-02-21 19:17:54 |
+-------------+---------------------+
```

Leave this open in a separate window while you test failure in the next section.

MySQL container uses readiness probe by running `mysql -h 127.0.0.1 -e 'SELECT 1'` on the server to make sure MySQL server is still active. Open a new terminal and simulate MySQL as being unresponsive.

```sh
kubectl -n mysql exec mysql-1 -c mysql -- touch /tmp/mysql-not-ready
```

This command creates a file `/tmp/mysql-not-ready` which will cause the readiness probe to fail. The readiness probe is configured to check for the absence of this file. During the next health check, the pod should report that its MySQL container is not ready.

```sh
kubectl -n mysql get pod mysql-1
```

Output:
```
NAME      READY     STATUS    RESTARTS   AGE
mysql-1   1/2       Running   0          12m
```

Notice only one container is in a `READY` state. 

**mysql-read** load balancer detects failures and takes action by not sending traffic to the failed container, `@@server_id 101`. You can check this by viewing the loop running in the separate window from previous section. The loop shows the following output.

```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 17:32:19 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 17:32:20 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 17:32:21 |
+-------------+---------------------+
```

Notice it does not read `@@server_id 101`

Fix the mysql server
```sh
kubectl -n mysql exec mysql-1 -c mysql -- rm /tmp/mysql-not-ready
```

Check the status again to see that both containers are running and healthy

```sh
kubectl -n mysql get pod mysql-1
```

Output:
```
NAME      READY     STATUS    RESTARTS   AGE
mysql-1   2/2       Running   0          5h
```

The loop in another terminal is now showing` @@server_id 101` is back and all servers are running. Press `Ctrl+C` to stop watching.

To simulate a failed pod, delete mysql-1
```sh
kubectl -n mysql delete pod mysql-1
```

```
pod "mysql-1" deleted
```

StatefulSet controller recognizes failed pod and creates a new one to maintain the number of replicas with the same name and link to the same `PersistentVolumeClaim`.

```sh
kubectl -n mysql get pod mysql-1 -w
```

Output
```
NAME      READY   STATUS        RESTARTS   AGE
mysql-1   2/2     Terminating   0          15m
mysql-1   0/2     Terminating   0          16m
mysql-1   0/2     Terminating   0          16m
mysql-1   0/2     Terminating   0          16m
mysql-1   0/2     Pending       0          0s
mysql-1   0/2     Pending       0          0s
mysql-1   0/2     Init:0/2      0          0s
mysql-1   0/2     Init:1/2      0          11s
mysql-1   0/2     PodInitializing   0          12s
mysql-1   1/2     Running           0          13s
mysql-1   2/2     Running           0          18s
```

### Test Scaling 
More followers can be added to the MySQL Cluster to increase read capacity. 
```sh
kubectl -n mysql scale statefulset mysql --replicas=3
```

Watch the progress of ordered and graceful scaling.

```sh
kubectl -n mysql rollout status statefulset mysql
```

Output:
```
Waiting for 1 pods to be ready...
partitioned roll out complete: 3 new pods have been updated...
```

In another terminal watch the new pod come online
```sh
kubectl -n mysql get pods -l app=mysql --watch
```

To exit type `Ctrl+C`

If you stopped the loop start it again. 
```sh
kubectl -n mysql run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```

You will now see 3 servers running. 

Output:
```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 02:32:43 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         102 | 2020-01-25 02:32:44 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2020-01-25 02:32:45 |
+-------------+---------------------+
```

Verify if the newly deployed follower `mysql-2` has the same data set.
```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
 mysql -h mysql-2.mysql -e "SELECT * FROM test.messages"
```

It will show the same data that the leader has.

Output:
```
+--------------------------+
| message                  |
+--------------------------+
| hello, from mysql-client |
+--------------------------+
```

Scale the replicas to 2

```sh
kubectl -n mysql scale statefulset mysql --replicas=2
```

You can see that it removed the last added replica. 
```
kubectl -n mysql get pods -l app=mysql
```

Output:
```
NAME      READY     STATUS    RESTARTS   AGE
mysql-0   2/2       Running   0          1d
mysql-1   2/2       Running   0          1d
```
Confirm the pvcs still exist
```sh
kubectl -n mysql  get pvc -l app=mysql
```

Output:
```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-fdb74a5e-ba51-4ccf-925b-e64761575059   10Gi       RWO            managed-premium      18m
data-mysql-1   Bound    pvc-355b9910-c446-4f66-8da6-629989a34d9a   10Gi       RWO            managed-premium    17m
data-mysql-2   Bound    pvc-12c304e4-2b3e-4621-8521-0dc17f41d107   10Gi       RWO            managed-premium    9m35s
```

### Bonus (Change reclaim policy)
By default, deleting a `PersistentVolumeClaim` will delete its associated persistent volume. What if you wanted to keep the volume?

Change the reclaim policy:
Find the `PersistentVolume` attached to the `PersistentVolumeClaim` `data-mysql-2`

```sh
export pv=$(kubectl -n mysql get pvc data-mysql-2 -o json | jq --raw-output '.spec.volumeName')
echo data-mysql-2 PersistentVolume name: ${pv}
```

Now update the `ReclaimPolicy`

```sh
kubectl -n mysql patch pv ${pv} -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

Verify the `ReclaimPolicy` was updated

```sh
kubectl -n mysql get persistentvolume
```

Now, if you delete the `PersistentVolumeClaim` `data-mysql-2`, you can still see the Azure Managed Disk in your Azure portal, with its state as "available".

Let's change the reclaim policy back to "Delete" to avoid orphaned volumes:

```sh
kubectl -n mysql patch pv ${pv} -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
unset pv
```

Delete `data-mysql-2`

```sh
kubectl -n mysql delete pvc data-mysql-2
```

Output:
```
persistentvolumeclaim "data-mysql-2" deleted
```

## Cleanup
```sh
kubectl delete \
  -f ${HOME}/environment/azure_statefulset/mysql-statefulset.yaml \
  -f ${HOME}/environment/azure_statefulset/mysql-services.yaml \
  -f ${HOME}/environment/azure_statefulset/mysql-configmap.yaml \

# Delete the mysql namespace 
kubectl delete namespace mysql
```

## Congrats!