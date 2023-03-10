# 세미 프로젝트 (1)

## 구현 목록 및 요구 사항

### wordpress APP

- Deployment
- Probe, Resource
- HPA
- ConfigMap
- Secret
- PVC
- Service/Ingress

### Mysql APP

- StatfulSet
- Probe, Resource
- Read Replica
- ConfigMap
- Secret
- PVC
- Service

helm 패키지 X 

Kustomize는 가능

# 사용법

### 테스트 환경

windows virtualbox 7

vm 4 (ubuntu 22.04.2 LTS)

![스크린샷 2023-03-10 오후 12.09.25.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b07d6b03-fdc3-4af5-8962-fb4ddc47071b/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-03-10_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_12.09.25.png)

use vagrant

## 1. storageclass 만들기

nfs-subdir-external-provisioner 이용 

[https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

```bash
cd nfs-subdir-external-provisioner-deploy
```

```bash
### deployment.yaml ###
		env:
      - name: PROVISIONER_NAME
        value: k8s-sigs.io/nfs-subdir-external-provisioner
      - name: NFS_SERVER
        value: 192.168.56.11
      - name: NFS_PATH
        value: /srv/nfs-volume
volumes:
  - name: nfs-client-root
    nfs:
      server: 192.168.56.11
      path: /srv/nfs-volume
```

ip, path경로 수정

```bash
kubectl patch storageclasses.storage.k8s.io nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

storage class default 설정 (pvc 할때 default로 사용 됨)

```bash
kubectl apply -k .
```

deployment.apps/nfs-client-provisioner 실행이 된다

```bash
kubectl get storageclasses.storage.k8s.io
```

nfs-client (default) 출력 되면 성공

<aside>
🙄 퍼블릭 클라우드를 이용한다면 지원되는 퍼블릭 클라우드 스토리지를 이용하여 storage class를 구현하는 것이 좋다

</aside>

## 2. db, wordpress 실행

```jsx
kubectl apply -k ./
```

실행중에 오류난다면 오류 로그 찾기

```bash
kubectl logs {pods} --all-containers
kubectl logs mydb-1 --all-containers
```

### db 테스트

*mydb-0.mydb => primary
mydb-1.mydb => replica*

dbclient 이미지 사용 (사용후 kubectl delete pods dbclient 로 제거)

primary 에 테스트정보 넣고 replica 로 출력해보는 작업

```bash
kubectl run dbclient --image ghcr.io/c1t1d0s7/network-multitool -it bash

mysql -h mydb-0.mydb -u root -p

SHOW DATABASES;
CREATE DATABASE testdb;
CREATE TABLE testdb.testtb (message VARCHAR(100));
INSERT INTO testdb.testtb VALUES ("Hello World");
exit

mysql -h mydb-1.mydb -u root -p -e 'SELECT * FROM testdb.testtb'
```

[https://www.youtube.com/watch?v=Pb0DwwvGKaA&list=PLqqWF8gP3uQbB2kMECrOwQwmKIADoXCii&index=1](https://www.youtube.com/watch?v=Pb0DwwvGKaA&list=PLqqWF8gP3uQbB2kMECrOwQwmKIADoXCii&index=1)

### 스테이트풀셋 레플리카 개수 변경

```bash
kubectl scale statefulset mydb  --replicas=3
```

[![스테이트풀셋](http://img.youtube.com/vi/xfVLrjpzEi0/0.jpg)](https://www.youtube.com/watch?v=xfVLrjpzEi0&list=PLqqWF8gP3uQbB2kMECrOwQwmKIADoXCii&index=2)

### hpa 모니터링

```bash
watch -n1 -d kubectl get hpa
watch -n1 -d kubectl top pods
```

- cpu 부하주기

```bash
kubectl exec {pod name} -- sha256sum /dev/zero
```

[https://www.youtube.com/watch?v=y-PfrbRSm2k&list=PLqqWF8gP3uQbB2kMECrOwQwmKIADoXCii&index=3](https://www.youtube.com/watch?v=y-PfrbRSm2k&list=PLqqWF8gP3uQbB2kMECrOwQwmKIADoXCii&index=3)

## 3. 접속 테스트

```bash
kubectl get ing
```

ing  ip로 접속

![스크린샷 2023-03-10 오후 12.23.28.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4b129bb0-05fe-4671-83d2-859345b94caf/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-03-10_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_12.23.28.png)

접속 후 기본 설치

![스크린샷 2023-03-10 오후 12.19.09.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c78b3ab6-388f-412b-b233-dcbcf8422cd4/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-03-10_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_12.19.09.png)

![스크린샷 2023-03-10 오후 12.22.18.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/80738c32-30d3-48c1-8b2a-2e51aa685a67/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-03-10_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_12.22.18.png)

## 전부 삭제

```jsx
kubectl delete -f ./
kubectl delete pvc --all
kubectl delete secret --all
kubectl delete cm --all

```

참고

[https://freedeveloper.tistory.com/430](https://freedeveloper.tistory.com/430)

[https://github.com/kubernetes/examples](https://github.com/kubernetes/examples)

[https://github.com/kubernetes/examples/tree/master/mysql-wordpress-pd](https://github.com/kubernetes/examples/tree/master/mysql-wordpress-pd)

[https://kubernetes.io/ko/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/](https://kubernetes.io/ko/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)

[https://nearhome.tistory.com/108](https://nearhome.tistory.com/108)

[https://do-hansung.tistory.com/57](https://do-hansung.tistory.com/57)

[https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)

[https://github.com/kubernetes/website/issues/21417](https://github.com/kubernetes/website/issues/21417)

[https://stackoverflow.com/questions/51905060/kubernetes-mysql-statefulset-with-root-password](https://stackoverflow.com/questions/51905060/kubernetes-mysql-statefulset-with-root-password)
