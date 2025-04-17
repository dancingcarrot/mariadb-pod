

## Table of Contents

1. [Mariadb pod 생성](#1)<br>
  1.1. [생성 조건](#1.1)<br>
  1.2. [pv 생성](#1.2)<br>
  1.3. [pvc 생성](#1.3)<br>
  1.4. [secret 생성](#1.4)<br>
  1.5. [configmap 생성](#1.5)<br>
  1.6. [service 생성](#1.6)<br>
  1.7. [pod 생성](#1.7)<br>
  1.8. [cronjob 생성](#1.8)<br>








# <div id='1'> 1. Mariadb Pod  생성

### <div id='1.1'> 1.1. 생성 조건


1. k8s : mariadb service deploy.sh (pod)구성

2. k8s : mariadb Config
- init.sql 적용(ConfigMap) : board table 생성 sql 실행 
  > board table : no, title, contents, user_name, reg_date, update_date 
- store data pvc 연결 
- mariadb backup script 생성 (CronJob) : 하루에 한번 오후 5시 
- Remote Client Access : mariadb.((system_domain)) -> nodePort




### <div id='1.2'> 1.2. pv 생성

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.20.0.149     #nfs 서버 ip 작성
    path: "/data/mysql"

```

### <div id='1.3'> 1.3. pvc 생성

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

```

### <div id='1.4'> 1.4. secret 생성

```
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
type: Opaque
data:
  password: MTIzNA==        
```

### <div id='1.5'> 1.5. configmap 생성

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-config
data:
  init.sql: |
    CREATE DATABASE IF NOT EXISTS sun;
    USE sun;
    CREATE TABLE IF NOT EXISTS board (
      no INT AUTO_INCREMENT,
      user_name VARCHAR(50),
      title VARCHAR(225) NOT NULL,
      contents VARCHAR(500),
      reg_date DATETIME,
      update_date DATETIME,
      primary key(no)
    );

```

### <div id='1.6'> 1.6. service 생성

```
apiVersion: v1
kind: Service
metadata:
  name: mariadb-service
spec:
  selector:
    app: mariadb
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 30000
  type: NodePort
```

### <div id='1.7'> 1.7. deployment 생성

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deployment
  labels:
    app: mariadb
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mariadb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:   
      - image: mariadb:10.7
        name: mariadb
        ports:
        - containerPort: 3306
          name: maraidb
        volumeMounts:
        - name: mariadb-config
          mountPath: /docker-entrypoint-initdb.d
        - mountPath: /var/lib/mysql
          name: mariadb-data
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: password       
      volumes:
      - name: mariadb-config
        configMap:
          name: mariadb-config
      - name: mariadb-data
        persistentVolumeClaim:
          claimName: nfs-pvc

```

### <div id='1.8'> 1.8. cronjob 생성

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: mariadb-backup-cronjob
spec:
  schedule: "0 17 * * *"
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup-container
            image: mariadb:10.7
            command: ["/bin/sh"]
            args: ["-c", "mysqldump -h mariadb-service -u root -p1234 sun > /data/tdb-$(date +%Y%m%d).sql"]
            volumeMounts:
              - name: mariadb-backup
                mountPath: /data
          restartPolicy: OnFailure
          volumes:
          - name: mariadb-backup
            persistentVolumeClaim:
              claimName: nfs-pvc

```

