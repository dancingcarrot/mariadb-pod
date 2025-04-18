# mariadb-pod
mariadb pod 생성

> 생성 조건

 1. 생성 조건
k8s : mariadb service deploy.sh (pod)구성<br>
k8s : mariadb Config

- init.sql 적용(ConfigMap) : board table 생성 sql 실행    > board table : no, title, contents, user_name, reg_date, update_date <br>
- store data pvc 연결 <br>
- mariadb backup script 생성 (CronJob) : 하루에 한번 오후 5시 <br>
- Remote Client Access : mariadb.((system_domain)) -> nodePort
