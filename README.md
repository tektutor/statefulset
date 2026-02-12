# Statefulset - mysql db cluster

Create a file named mysql-svc.yml
<pre>
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
    - port: 3306
      name: mysql
    - port: 33061
      name: group-replication
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

    # Replication
    log_bin=binlog
    server_id=1
    loose-group_replication_group_name="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
    loose-group_replication_start_on_boot=off
    loose-group_replication_local_address="$(hostname -f):33061"
    loose-group_replication_group_seeds="mysql-0.mysql:33061,mysql-1.mysql:33061,mysql-2.mysql:33061"
    loose-group_replication_bootstrap_group=off
</pre>

Create a mysql-secret.yml
<pre>
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: mysqlpassword  
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

Now let's deploy it
```
oc apply -f mysql-svc.yaml
oc apply -f mysql-cm.yaml
oc apply -f mysql-secrets.yaml
oc apply -f mysql-ss.yaml
```
