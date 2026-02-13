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
  root-password: Root@123
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-0
spec:
  capacity:
    storage: 2Gi
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
  name: mysql-pv-1
spec:
  capacity:
    storage: 2Gi
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
  name: mysql-pv-2
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.10.200
    path: /var/nfs/jegan/mysql/mysql-2
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-0
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  volumeName: mysql-pv-0
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  volumeName: mysql-pv-1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc-2
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  volumeName: mysql-pv-2
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
        runAsUser: 0   # Must run as root for OpenShift
      containers:
      - name: mysql
        image: docker.io/mysql:8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_REPLICATION_USER
          value: repl
        - name: MYSQL_REPLICATION_PASSWORD
          value: repl@123
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        command:
          - bash
          - -c
          - |
            # Install group replication plugin if not present
            /entrypoint.sh mysqld & 
            sleep 15
            mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "INSTALL PLUGIN group_replication SONAME 'group_replication.so';"
            mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SET GLOBAL group_replication_bootstrap_group=ON;"
            fg
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc-0
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 2Gi

```
Run it
```
oc apply -f mysql-statefulset.yml
```
