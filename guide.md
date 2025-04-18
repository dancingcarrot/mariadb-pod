

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






+ 가이드 순으로 생성을 진행해 주시기 바랍니다.

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

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl apply -f pv.yaml -n sun 
persistentvolume/nfs-pv created


ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get pv -n sun
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                      STORAGECLASS      VOLUMEATTRIBUTESCLASS   REASON   AGE
jenkins-pv                                 8Gi        RWO            Delete           Available                                                                <unset>                          13d
nfs-pv                                     10Gi       RWX            Retain           Available                                                                <unset>                          39s


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

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl apply -f pvc.yaml -n sun
persistentvolumeclaim/nfs-pvc created

ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get pvc -n sun
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
nfs-pvc   Bound    pvc-0ee91c0d-8c56-49dd-a287-ccefc677172e   10Gi       RWX            cp-storageclass   <unset>                 3s

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

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl apply -f secret.yaml  -n sun
secret/mariadb-secret created

ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get secret -n sun
NAME             TYPE     DATA   AGE
mariadb-secret   Opaque   1      4s

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

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ vi configmap.yaml
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl apply -f configmap.yaml -n sun
configmap/mariadb-config created

ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get configmaps -n sun
NAME               DATA   AGE
kube-root-ca.crt   1      13d
mariadb-config     1      8s

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

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl apply -f service.yaml -n sun
service/mariadb-service created

ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get svc -n sun
NAME              TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
mariadb-service   NodePort   10.233.53.92   <none>        3306:30000/TCP   3s

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
  replicas: 1
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

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl apply -f deployment.yaml  -n sun
Warning: would violate PodSecurity "restricted:v1.30": allowPrivilegeEscalation != false (container "mariadb" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "mariadb" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "mariadb" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "mariadb" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/mariadb-deployment created

ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get pods -n sun
NAME                                 READY   STATUS    RESTARTS   AGE
mariadb-deployment-9c8f6758d-85f5q   1/1     Running   0          4s


```
<details>

<summary>deployment에서 replicas를 3개로 지정한 후 배포후에 파드가 하나만 정상작동 할 경우</summary>

```
ubuntu@test-cluster-1:~$ kubectl get pods -n sun
NAME                                 READY   STATUS             RESTARTS      AGE
mariadb-deployment-9c8f6758d-7mlfn   0/1     CrashLoopBackOff   7 (42s ago)   16m
mariadb-deployment-9c8f6758d-ff8nc   1/1     Running            0             16m
mariadb-deployment-9c8f6758d-wqj89   0/1     CrashLoopBackOff   7 (56s ago)   16m
```

<br>
여러 pod가 동일한 pvc를 공유할 경우, 데이터 디렉토리에서 충돌이 발생할 수 있다.
Mariadb는 단일 인스턴스에서 데이터 디렉토리를 독점적으로 사용하도록 설계되어 있어, PVC를 공유하면 문제가 발생할 수 있다.

* 해결방법
  - PVC 분리: 각 MariaDB 파드가 독립적인 pvc를 사용하도록 설정한다. 이를 위해 statefulset 을 사용한다.

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
spec:
  serviceName: mariadb
  replicas: 3
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.7
        ports:
        - containerPort: 3306
          name: mariadb
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get pods -n sun
NAME        READY   STATUS    RESTARTS   AGE
mariadb-0   1/1     Running   0          7s
mariadb-1   1/1     Running   0          6s
mariadb-2   1/1     Running   0          4s

```

</details>


### <div id='1.8'> 1.8. cronjob 생성

```
apiVersion: batch/v1
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

```
ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl apply -f cronjob.yaml -n sun
Warning: would violate PodSecurity "restricted:v1.30": allowPrivilegeEscalation != false (container "backup-container" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "backup-container" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "backup-container" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "backup-container" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
cronjob.batch/mariadb-backup-cronjob created

ubuntu@test-cluster-1:~/workspace/sun/sub$ kubectl get cronjob -n sun
NAME                     SCHEDULE     TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
mariadb-backup-cronjob   0 17 * * *   <none>     False     0        <none>          61s


```

