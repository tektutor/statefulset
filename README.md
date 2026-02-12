# Statefulset - mysql db cluster

Paste the below in mysql-statefulset.yml file

```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: root@123
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-0
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-1
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-2
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: install-group-replication
        image: bitnami/mysql:8.0
        command:
          - bash
          - -c
          - |
            mysql --user=root --password="$MYSQL_ROOT_PASSWORD" -e "INSTALL PLUGIN group_replication SONAME 'group_replication.so';" || true
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: root-password
      containers:
      - name: mysql
        image: bitnami/mysql:8.0
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: root-password
          - name: MYSQL_REPLICATION_MODE
            value: "group"
          - name: MYSQL_GROUP_NAME
            value: "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
          - name: MYSQL_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
        ports:
          - containerPort: 3306
            name: mysql
          - containerPort: 33060
            name: mysqlx
        volumeMounts:
          - name: data
            mountPath: /bitnami/mysql
        command:
          - bash
          - -c
          - |
            #!/bin/bash
            set -e
            # bootstrap first node only
            if [[ "$HOSTNAME" == "mysql-0" ]]; then
              echo "Bootstrapping first node"
              mysqld --user=mysql &
              sleep 15
              mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SET GLOBAL group_replication_bootstrap_group=ON; START GROUP_REPLICATION; SET GLOBAL group_replication_bootstrap_group=OFF;"
              fg
            else
              echo "Joining cluster"
              mysqld --user=mysql &
              sleep 15
              mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "START GROUP_REPLICATION;"
              fg
            fi
  volumeClaimTemplates:
    - metadata:
        name: data
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
