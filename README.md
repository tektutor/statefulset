# Statefulset - mysql db cluster

Paste the below in mysql-statefulset.yml file

```
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: root@123
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: mysql-nfs
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
    - noatime
  nfs:
    path: /var/nfs/jegan/mysql/mysql-0
    server: 192.168.10.200
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-1
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: mysql-nfs
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
    - noatime
  nfs:
    path: /var/nfs/jegan/mysql/mysql-1
    server: 192.168.10.200
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-2
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: mysql-nfs
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
    - noatime
  nfs:
    path: /var/nfs/jegan/mysql/mysql-2
    server: 192.168.10.200
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
      name: mysql
    - port: 33061
      name: mysql-gr
  clusterIP: None
  selector:
    app: mysql
---
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
          image: mysql:8.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
              name: mysql
            - containerPort: 33061
              name: mysql-gr
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
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - "mysqladmin ping -uroot -p$MYSQL_ROOT_PASSWORD"
            initialDelaySeconds: 30
            periodSeconds: 10
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: mysql-nfs
        resources:
          requests:
            storage: 10Gi
```
Run it
```
oc apply -f mysql-statefulset.yml
```
