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

You can see the **mysql** service is for DNS resolution so that when pods are placed by StatefulSet controller, pods can be resolved using `pod-name.mysql`. **mysql-read** is a client service that does load balancing for all followers.

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
cat << EoF > ${HOME}/environment/azure_statefulset/mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword"
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
      volumes:
      - name: conf
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

It will take a few minutes for pods to initialize and the `StatefulSet` to be created.

Open another terminal and watch the progress of pods creation using the following command.
```sh
kubectl -n mysql get pods -l app=mysql --watch
```

You can see ordered, graceful deployment with a stable, unique name for each pod.

Check the dynamically created PVC
```sh
kubectl -n mysql get pvc -l app=mysql
```

### Test MySQL
You can use **mysql-client** to send some data to the leader, **mysql-0.mysql** by running the following command.

```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql -proot<<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello, from mysql-client');
EOF
```

Run the following to test follower `mysql-read` received the data.

```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -it --rm --restart=Never --\
  mysql -h mysql-read -proot -e "SELECT * FROM test.messages"
```

To test load balancing across followers, run the following command.
```sh
kubectl -n mysql run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-read -proot -e 'SELECT @@server_id,NOW()'; done"
```

Each MySQL instance is assigned a unique identifier, and it can be retrieved using `@@server_id`. It will print the server id serving the request and the timestamp.

### Test Scaling 
Scale the replicas to 5:

```sh
kubectl -n mysql scale statefulset mysql --replicas=5
```

Watch the progress of ordered and graceful scaling.

```sh
kubectl -n mysql rollout status statefulset mysql
```

### Clean Up

To remove the resources we created in the cluster, you can delete the namespace:

```sh
kubectl delete namespace mysql
```

## Congratulations!

You've successfully deployed and tested a StatefulSet in AKS!