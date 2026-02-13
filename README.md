# Statefulset - mysql db cluster

Paste the below in mysql-statefulset.yml file

```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "Root@123"

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-0
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.10.200
    path: /var/nfs/jegan/mysql/mysql-0
  persistentVolumeReclaimPolicy: Retain

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-1
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.10.200
    path: /var/nfs/jegan/mysql/mysql-1
  persistentVolumeReclaimPolicy: Retain

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-2
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.10.200
    path: /var/nfs/jegan/mysql/mysql-2
  persistentVolumeReclaimPolicy: Retain

---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
    - port: 3306
      name: mysql
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
      securityContext:
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: mysql
          image: image-registry.openshift-image-registry.svc:5000/openshift/mysql:8.0
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_REPLICATION_MODE
              value: "master"
            - name: MYSQL_REPLICATION_USER
              value: "repl_user"
            - name: MYSQL_REPLICATION_PASSWORD
              value: "Repl@123"
            - name: MYSQL_DATABASE
              value: "mydb"
            - name: MYSQL_USER
              value: "appuser"
            - name: MYSQL_PASSWORD
              value: "App@123"
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql/data
      initContainers:
        - name: init-mysql-permissions
          image: busybox:1.36
          command: ["sh", "-c", "chown -R 1001:1001 /var/lib/mysql/data"]
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql/data
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi

```
Run it
```
oc apply -f mysql-statefulset.yml
```
