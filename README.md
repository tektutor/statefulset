# Statefulset - mysql db cluster

Paste the below in mysql-statefulset.yml file

```
---
# 1️⃣ Secret for MySQL root password
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: root@123

---
# 2️⃣ PersistentVolumes (static, replace NFS paths if needed)
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
    server: 192.168.10.200
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
    server: 192.168.10.200
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
    server: 192.168.10.200
  persistentVolumeReclaimPolicy: Retain

---
# 3️⃣ StatefulSet for 3-node MySQL Group Replication
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
        image: docker.io/bitnamilegacy/mysql:8.0.39
        imagePullPolicy: IfNotPresent
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: root-password
          - name: MYSQL_REPLICATION_USER
            value: repl_user
          - name: MYSQL_REPLICATION_PASSWORD
            value: repl_pass
          - name: MYSQL_REPLICATION_MODE
            value: "group"
          - name: MYSQL_GROUP_NAME
            value: "rps-mysql-grp"
          - name: MYSQL_MASTER_HOST
            value: "mysql-0.mysql"
        ports:
          - name: mysql
            containerPort: 3306
          - name: mysqlx
            containerPort: 33060
        volumeMounts:
          - name: mysql-data
            mountPath: /bitnami/mysql
        readinessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -uroot
            - -p$(MYSQL_ROOT_PASSWORD)
          initialDelaySeconds: 15
          periodSeconds: 10
      initContainers:
        - name: setup-replication
          image: bitnami/mysql:8.0
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            - name: MYSQL_REPLICATION_USER
              value: repl_user
            - name: MYSQL_REPLICATION_PASSWORD
              value: repl_pass
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          command:
            - bash
            - -c
            - |
              # Wait for MySQL to be ready
              until mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1;" &>/dev/null; do sleep 5; done

              # Install group_replication plugin if not present
              PLUGIN=$(mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -sse "SELECT PLUGIN_NAME FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME='group_replication';")
              if [ -z "$PLUGIN" ]; then
                mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "INSTALL PLUGIN group_replication SONAME 'group_replication.so';"
              fi

              # Create replication user
              mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "CREATE USER IF NOT EXISTS '$MYSQL_REPLICATION_USER'@'%' IDENTIFIED BY '$MYSQL_REPLICATION_PASSWORD';"
              mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "GRANT REPLICATION SLAVE ON *.* TO '$MYSQL_REPLICATION_USER'@'%'; FLUSH PRIVILEGES;"

              # Bootstrap first pod
              if [[ "$POD_NAME" == "mysql-0" ]]; then
                mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SET GLOBAL group_replication_bootstrap_group=ON;"
              fi
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```
Run it
```
oc apply -f mysql-statefulset.yml
```
