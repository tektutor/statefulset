# Statefulset - mysql db cluster

Create a file named mysql-svc.yml
<pre>
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  clusterIP: None
  ports:
    - port: 3306
      name: mysql
    - port: 33061
      name: grp-repl
  selector:
    app: mysql
</pre>

Create a file named mysql-cm.yml
<pre>
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    bind-address=0.0.0.0
    default_storage_engine=InnoDB
    binlog_format=ROW
    gtid_mode=ON
    enforce_gtid_consistency=ON
    master_info_repository=TABLE
    relay_log_info_repository=TABLE
    transaction_write_set_extraction=XXHASH64

    log_bin=binlog
    log_slave_updates=ON

    loose-group_replication_group_name="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
    loose-group_replication_start_on_boot=off
    loose-group_replication_bootstrap_group=off
    loose-group_replication_group_seeds="mysql-0.mysql:33061,mysql-1.mysql:33061,mysql-2.mysql:33061"
</pre>

Create a mysql-secret.yml
<pre>
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: root@123
</pre>

Create a mysql-ss.yml
<pre>
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
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
          image: image-registry.openshift-image-registry.svc:5000/openshift/mariadb:1.0
          ports:
            - containerPort: 3306
              name: mysql
            - containerPort: 33061
              name: group-replication
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
            - name: config
              mountPath: /etc/mysql/conf.d
          command:
            - bash
            - "-c"
            - |
              HOSTNAME=$(hostname)
              ORDINAL=${HOSTNAME##*-}
              export MYSQL_SERVER_ID=$((100 + ORDINAL))

              echo "server-id=${MYSQL_SERVER_ID}" >> /etc/mysql/conf.d/server-id.cnf
              exec docker-entrypoint.sh mysqld
      volumes:
        - name: config
          configMap:
            name: mysql-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
</pre>

Create a file named pv.yml
<pre>
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-0
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /var/nfs/jegan/mysql/mysql-0
    server: 192.168.10.201
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /var/nfs/jegan/mysql/mysql-1
    server: 192.168.10.201
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /var/nfs/jegan/mysql/mysql-2
    server: 192.168.10.201
  persistentVolumeReclaimPolicy: Retain  
</pre>

Now let's deploy it
```
oc apply -f mysql-pv.yml
oc apply -f mysql-svc.yaml
oc apply -f mysql-cm.yaml
oc apply -f mysql-secrets.yaml
oc apply -f mysql-ss.yaml
```

Now get inside master mysql instance
```
oc exec -it mysql-0 -- mysql -u root -p root@123
SET SQL_LOG_BIN=0;
CREATE USER 'repl'@'%' IDENTIFIED BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;

CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='replpass' FOR CHANNEL 'group_replication_recovery';

INSTALL PLUGIN group_replication SONAME 'group_replication.so';

SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
```

In mysql-1
```
oc exec -it mysql-1 -- mysql -u root -p root@123
START GROUP_REPLICATION;
```

In mysql-2
```
oc exec -it mysql-2 -- mysql -u root -p root@123
START GROUP_REPLICATION;
```

Note NFS should have
```
mkdir -p /var/nfs/jegan/mysql/mysql-0
mkdir -p /var/nfs/jegan/mysql/mysql-1
mkdir -p /var/nfs/jegan/mysql/mysql-2
chmod -R 777 /var/nfs/jegan/mysql
```


