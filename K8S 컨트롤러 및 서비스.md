# Learning Kubernetes

[toc]

## 3. Replication Controller

### 3.1 Replication Controller 생성

아래와 같이 template를 작성합니다.

```{yaml}
apiVersion: v1
kind: ReplicationController
metadata:
  name: goapp-rc
spec:
  replicas: 3
  selector:
    app: goapp
  template:
    metadata:
      name: goapp-pod
      labels:
        tier: forntend
        app: goapp
        env: prod
        priority:  high
    spec:
      containers:
      - name: goapp-container
        image: chungsju/goapp
        ports:
        - containerPort: 8080
```

### 3.2 Replication Controller  생성

```{bash}
kubectl create -f ./rc-goapp.yaml
```

### 3.3 Pod 생성 확인

```{bash}
kubectl get po

NAME             READY   STATUS    RESTARTS   AGE
goapp-rc-9q689   1/1     Running   0          39s
goapp-rc-d5rnf   1/1     Running   0          39s
goapp-rc-fm7kr   1/1     Running   0          39s
```

### 3.4 Replication Controller 확인

```{bash}
kubectl get rc

NAME       DESIRED   CURRENT   READY   AGE
goapp-rc   3         3         3       58s

kubectl get rc -o wide

NAME       DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES           SELECTOR
goapp-rc   3         3         3       72s   goapp-container   chungsju/goapp   app=goapp

```

### 3.5 특정 Pod 삭제하고 변화 확인하기

아래와 같이 3개의 Pod 중에 하나를 선택해서 삭제 합니다.

```{bash}
kubectl deleete pod goapp-rc-9q689
```

```{bash}
kubectl get pod

NAME             READY   STATUS    						RESTARTS   AGE
goapp-rc-d5rnf   1/1     Running   						0          2m
goapp-rc-fm7kr   1/1     Running   						0          2m
goapp-rc-szv2r   1/1     ContainerCreating   	0          6s
```

기존 컨테이너를 Terminating 하고 새로운 컨테이너를 생성하는 것을 볼 수 있습니다.

### 3.6 Pod 정보를 라벨과 함께 출력해보기

```{bash}
kubectl get pod --show-labels

NAME             READY   STATUS    RESTARTS   AGE     LABELS
goapp-rc-d5rnf   1/1     Running   0          7m26s   app=goapp
goapp-rc-fm7kr   1/1     Running   0          7m26s   app=goapp
goapp-rc-szv2r   1/1     Running   0          4m51s   app=goapp
```

### 3.7 Pod 라벨을 변경해보기

기존 "app=nginx" 라는 label 을 "app=goapp-exit" 로 변경 합니다.

```{bash}
kubectl label pod goapp-rc-szv2r app=goapp-exit --overwrite
```

아래와 같이 pod 를 조회 해봅니다.

```{bash}
kubectl get po

NAME             READY   STATUS              RESTARTS   AGE
goapp-rc-d5rnf   1/1     Running             0          8m49s
goapp-rc-fm7kr   1/1     Running             0          8m49s
goapp-rc-mmn2b   0/1     ContainerCreating   0          5s
goapp-rc-szv2r   1/1     Running             0          6m14s
```

기존 3개의 Pod 중 하나의 Label을 변경하면 기존 app=goapp 에는 2개의 Pod 만 남기 때문에 Replication Controller 는 **추가적으로 하나의 Pod 를 생성** 합니다.

```{bash}
NAME             READY   STATUS    RESTARTS   AGE     LABELS
goapp-rc-d5rnf   1/1     Running   0          9m27s   app=goapp
goapp-rc-fm7kr   1/1     Running   0          9m27s   app=goapp
goapp-rc-mmn2b   1/1     Running   0          43s     app=goapp
goapp-rc-szv2r   1/1     Running   0          6m52s   app=goapp-exit
```

### 3.8 Pod Template 변경 해보기

아래와 같이 Pod Template의 spec ➢ spec ➢ containers ➢ image 항목을 nginx:1.20 로 변경 합니다.

```{bash}
kubectl edit rc nginx
```

```{yaml}
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: ReplicationController
metadata:
  creationTimestamp: "2020-04-01T09:32:23Z"
  generation: 1
  labels:
    app: goapp
  name: goapp-rc
  namespace: default
  resourceVersion: "405444"
  selfLink: /api/v1/namespaces/default/replicationcontrollers/goapp-rc
  uid: 17198300-d964-4de6-a160-825a7a9c16bf
spec:
  replicas: 3
  selector:
    app: goapp
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: goapp
      name: goapp-pod
    spec:
      containers:
      - image: nginx:1.20 # 이부분을 변경 합니다.(기존 : chungsju/goapp)
        imagePullPolicy: Always
        name: goapp-container
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  fullyLabeledReplicas: 3
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
```

저장후 편집기를 종료 합니다.

> 리눅스 편집기에는 다양한 종유가 있습니다. 만약 기본 편집기를 변경하고 싶으면
>
> KUBE_EDITOR="/bin/nano" 를 $HOME/.bashrc 파일에 선언 해주면 원하는 편집기를 사용 할수 있습니다.
>


### 3.9 Pod Template 를 적용하기 위해 임의의 Pod 삭제하기

```{bash}
kubectl get pod

goapp-rc-mvw57   1/1     Running   0          3h6m    10.36.0.2   worker02.acorn.com
goapp-rc-qkrpw   1/1     Running   0          6m26s   10.32.0.2   worker01.acorn.com
goapp-rc-x6q4d   1/1     Running   0          3h6m    10.36.0.1   worker02.acorn.com
```

Pod 삭제

```{bash}
kubectl delete pod goapp-rc-mvw57

pod "goapp-rc-mvw57" deleted
```

Pod 확인

```{bash}
kubectl get pod

NAME             READY   STATUS              RESTARTS   AGE
goapp-rc-bf2xk   0/1     ContainerCreating   0          8s
goapp-rc-qkrpw   1/1     Running             0          7m5s
goapp-rc-x6q4d   1/1     Running             0          3h7m

kubectl get pod -o wide

NAME             READY   STATUS    RESTARTS   AGE     IP          NODE                 NOMINATED NODE   READINESS GATES
goapp-rc-bf2xk   1/1     Running   0          21s     10.32.0.3   worker01.acorn.com
goapp-rc-qkrpw   1/1     Running   0          7m18s   10.32.0.2   worker01.acorn.com
goapp-rc-x6q4d   1/1     Running   0          3h7m    10.36.0.1   worker02.acorn.com
```

접속 해보기

```{bash}
# 기존 pod의 이미지 확인
kubectl get  pod goapp-rc-x6q4d -o jsonpath="{..image}"

# 새로 생성된 pod의 이미지 확인
kubectl get  pod goapp-rc-bf2xk -o jsonpath="{..image}"
```

### 3.10 Pod 스케일링

- Template 변경을 통한 스케일링

  아래와 같이 goapp-rc 를 edit 명령으로 수정 합니다. (replicas 항목을 3에서 4로 수정)

```{bash}
kubectl edit rc goapp-rc
```

```{yaml}
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: ReplicationController
metadata:
  creationTimestamp: "2020-04-01T09:51:49Z"
  generation: 3
  labels:
    app: goapp
  name: goapp-rc
  namespace: default
  resourceVersion: "416408"
  selfLink: /api/v1/namespaces/default/replicationcontrollers/goapp-rc
  uid: 23f58f51-88ab-4828-9a76-cde8a646fff4
spec:
  replicas: 4  # 이부분을 변경 합니다. (기존 : 3)
  selector:
    app: goapp
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: goapp
      name: goapp-pod
    spec:
      containers:
      - image: nginx:1.20
        imagePullPolicy: Always
        name: goapp-container
        ports:
```

저장 한다음 Pod 및 RC 확인

```{bash}
kubectl get pod

NAME             READY   STATUS              RESTARTS   AGE
goapp-rc-bf2xk   1/1     Running             0          19m
goapp-rc-mr6kb   0/1     ContainerCreating   0          7s
goapp-rc-qkrpw   1/1     Running             0          26m
goapp-rc-x6q4d   1/1     Running             0          3h26m
```

```{bash}
kubectl get rc

NAME       DESIRED   CURRENT   READY   AGE
goapp-rc   4         4         4       4h17m
```

- 명령어를 통한 스케일링

명령어를 이용해서 스케일링을 수행 할 수 있습니다.

```{bash}
kubectl scale rc goapp-rc --replicas=5
```

실제로 Pod 가 늘어 났는지 확인해봅니다.

```{bash}
kubectl get pod

NAME             READY   STATUS              RESTARTS   AGE
goapp-rc-bf2xk   1/1     Running             0          72m
goapp-rc-dlgfc   0/1     ContainerCreating   0          4s
goapp-rc-mr6kb   1/1     Running             0          53m
goapp-rc-qkrpw   1/1     Running             0          79m
goapp-rc-x6q4d   1/1     Running             0          4h19m
```

### 3.11 Replication Controller 삭제

Replication Controller 와 POD 모두 삭제

```{bash}
kubectl delete rc goapp-rc
```

Replication Controller 만 삭제. POD 는 그대로 유지 합니다.

```{bash}
kubectl delete rc goapp-rc --cascade=false
```

## 4.ReplicaSet

### 4.1 RS 생성

Selector 를 작성 할때 **ForntEnd** 이고 **운영계** 이면서 중요도가 **High** 인 POD 에 대해 RS 를 생성 합니다.

```{yaml}
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: env, operator: In, values: [prod]}
      - {key: priority, operator: NotIn, values: [low]}
  template:
    metadata:
      labels:
        tier: frontend
        env: prod
        priority: high
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

### 4.2 RS 확인

```{bash}
$ kubectl get pod -o wide

NAME             READY   STATUS    RESTARTS   AGE   IP          NODE
frontend-bstms   1/1     Running   0          53s   10.32.2.7   gke-gke1-default-pool-ad44d907-cq8j
frontend-d4znb   1/1     Running   0          53s   10.32.2.9   gke-gke1-default-pool-ad44d907-cq8j
frontend-rv9bl   1/1     Running   0          53s   10.32.2.8   gke-gke1-default-pool-ad44d907-cq8j
```

```{bash}
$ kubectl get rs -o wide

AME             READY   STATUS    RESTARTS   AGE   IP          NODE
frontend-bstms   1/1     Running   0          68s   10.32.2.7   gke-gke1-default-pool-ad44d907-cq8j
frontend-d4znb   1/1     Running   0          68s   10.32.2.9   gke-gke1-default-pool-ad44d907-cq8j
frontend-rv9bl   1/1     Running   0          68s   10.32.2.8   gke-gke1-default-pool-ad44d907-cq8j

```

```{bash}
$ kubectl get pod --show-labels

NAME             READY   STATUS    RESTARTS   AGE    LABELS
frontend-bstms   1/1     Running   0          107s   env=prod,priority=high,tier=frontend
frontend-d4znb   1/1     Running   0          107s   env=prod,priority=high,tier=frontend
frontend-rv9bl   1/1     Running   0          107s   env=prod,priority=high,tier=frontend
```

### [[Exercise]]

1. Nginx:1.21.1 Pod 3개로 구성된 Replication Controller를 작성 하세요
2. Replication Controller 만 삭제 하세요(Pod 는 유지)   (--cascade=false 옵션 사용)
3. 남겨진 Nginx Pod를 관리하는 ReplicaSet 을 작성하고 replica 4개로 구성 하시요
4. Nginx Pod 를 6개로 Scale Out 하세요



## 5. Deployment

### 5.1 Deployment 생성

아래와 같이 nginx 를 서비스 하고 replica 가 3개인 Deployment 를 작성합니다.(nginx-deploy.yaml)

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20.1
        ports:
        - containerPort: 80

```

```{bash}
kubectl apply -f ./nginx-deploy.yaml
```

### 5.2 Deployment 확인

```{bash}
kubectl get pod,rs,deploy
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-54f57cf6bf-dpsn4   1/1     Running   0          30s
pod/nginx-deployment-54f57cf6bf-ghfwm   1/1     Running   0          30s
pod/nginx-deployment-54f57cf6bf-rswwk   1/1     Running   0          30s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-54f57cf6bf   3         3         3       30s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           30s
```

### 5.3 이미지 업데이트

- 기본 방식

```{yaml}
kubectl --record deployment.apps/nginx-deployment set image deployment.v1.apps/nginx-deployment nginx=nginx:1.20.2
```

- 단축 방식

```{bash}
kubectl set image deployment/nginx-deployment nginx=nginx:1.20.2 --record
```

- yaml 파일 수정(.spec.template.spec.containers[0].image 변경)

```{bash}
kubectl edit deployment.v1.apps/nginx-deployment
```

- 롤아웃

```shell
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.21.1 --record
```

```shell
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.21.5 --record
```

- 롤아웃 히스토리 확인

```shell
kubectl rollout history deployment/nginx-deployment
```

- 롤백

```shell
kubectl rollout undo deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment

kubectl rollout undo deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment

kubectl rollout undo deployment node-js-deploy --to-revision 3
kubectl rollout history deployment/nginx-deployment
```


### 5.4 Deployment ScaleOut

```{bash}
kubectl edit deploy goapp-deployment

kubectl scale deploy nginx-deployment --replicas=4
goapp-deployment-5857594fbb-2hhnv   1/1     Running   0          32m
goapp-deployment-5857594fbb-6s9lx   1/1     Running   0          6s
goapp-deployment-5857594fbb-7nqsg   1/1     Running   0          32m
goapp-deployment-5857594fbb-9b28s   1/1     Running   0          32m
```

### 5.4 deployment 확인 하기

- 롤아웃 확인

```{bash}
kubectl rollout status deployment.v1.apps/nginx-deployment
```

- deployment 상세 확인

```{bash}
kubectl describe deploy nginx-deployment
```

## 6.DeamonSet

### 6.1 데몬셋 생성

goapp-ds.yaml 이라는 이름으로 아래 파일을 작성 합니다.

```{yaml}
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: goapp-on-ssd
spec:
  selector:
    matchLabels:
      app: goapp-pod
  template:
    metadata:
      labels:
        app: goapp-pod
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: goapp-container
        image: chungsju/goapp
```

데몬셋을 생성 합니다.

```{bash}
$ kubectl create -f ./goapp-ds.yaml
```

Pod 와 데몬셋을 조회 합니다.

```{bash}
$ kubectl get pod

$ kubectl get ds

NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
goapp-on-ssd   0         0         0       0            0           disk=ssd        <nome>
```

조회 하면 Pod 도 존재하지 않고 데몬셋 정보를 조회 해도 모두 0 으로 나옵닏다. 노드에 disk=ssd 라벨이 없기 때문입니다.

이제 라벨을 추가 합니다.

```{bash}
$ kubectl label node worker01.acorn.com disk=ssd

$ kubectl get pod
NAME                 READY   STATUS    RESTARTS   AGE
goapp-on-ssd-vwvks   1/1     Running   0          7s

$ kubectl label node worker02.acorn.com disk=ssd

$ kubectl get pod
NAME                 READY   STATUS    RESTARTS   AGE
goapp-on-ssd-nbnwz   1/1     Running   0          7s
goapp-on-ssd-vwvks   1/1     Running   0          36s

$ kubectl get ds -o wide
AME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
goapp-on-ssd   2         2         2       2            2           disk=ssd        10m
```

## 7. Job / CronJob 컨트롤러

### 7.1 Job

Pi-job.yaml 파일을 생성합니다.

```{yaml}
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

- Job 실행

```{bash}
kubectl apply -f ./Pi-job.yaml 
```

- Job 확인

```{bash}
kubectl get jobs/pi
kubectl describe jobs/pi
```

### 7.2 CronJob 

date-cronjob.yaml 파일을 생성합니다.

```{yaml}
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

```{bash}
kubectl apply -f ./date-cronjob.yaml

kubectl get jobs
kubectl get cronjobs
```

## 8. Horizontal Pod Auto-Scaler

[[Exercise]]

Nginx 웹서비스를 하는 Deployment :  name : web-deploy , Replica =3 를 생성하세요.

CPU 사용률 50%에 따른 HPA 설정
```{bash}
kubectl autoscale deployment web-deploy --cpu-percent=50 --min=1 --max=8

kubectl get hpa web-deploy -o wide

#조금 후에, 다시 
kubectl get hpa web-deploy -o wide
```

## 9. 서비스 (Service)

### 9.1 ClusterIP

#### 9.1.1 nodes app 생성 (실습용 Docker Image 내용 : chungju/nodeapp에서 내려받을수 있음)

app.js 파일 생성

```{javascript}
const http = require('http');
const os = require('os');

console.log("Kubia server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("V1 : You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

```{bash}
# FROM 으로 BASE 이미지 로드
FROM node:7

# ADD 명령어로 이미지에 app.js 파일 추가
ADD app.js /app.js

# ENTRYPOINT 명령어로 node 를 실행하고 매개변수로 app.js 를 전달
ENTRYPOINT ["node", "app.js"]
```



#### 9.1.2 Deployment 생성

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeapp-pod
  template:
    metadata:
      labels:
        app: nodeapp-pod
    spec:
      containers:
      - name: nodeapp-container
        image: chungsju/nodeapp
        ports:
        - containerPort: 8080
```

#### 9.1.3 yaml을 통한 ClusterIP 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: nodeapp-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nodeapp-pod
```

#### 9.1.4 서비스 상태 확인

```{bash}
kubectl get  po,deploy,svc

NAME                                      READY   STATUS    RESTARTS   AGE
pod/nodeapp-deploy-55688d9d4b-8pzsk   1/1     Running   0          2m45s
pod/nodeapp-deploy-55688d9d4b-pslvb   1/1     Running   0          2m46s
pod/nodeapp-deploy-55688d9d4b-whbk8   1/1     Running   0          2m46s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nodeapp-deploy   3/3     3            3           2m46s

NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
nodeapp-service   ClusterIP      10.101.249.42    <none>           80/TCP         78s
```

조회된 Pod 중 하나에  exec 옵션을 사용해 sh 로 접속 합니다.

```{bash}
kubectl exec -it nodeapp-deploy-55688d9d4b-8pzsk -- /bin/sh
```

curl 을 설치 하고 Cluster IP 로 접속합니다.

```{bash}
apt-get install curl

clusterIP로 여러번 호출
curl http://10.116.11.242 
서비스 이름으로 여러번 호출
curl http://nodeapp-service
```

#### 9.1.5 서비스 삭제

```{bash}
kubectl delete svc nodeapp-service
```



### 9.2 NodePort

#### 9.2.1 yaml 을 이용한 NodePort 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: node-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app:  nodeapp-pod
```

#### 9.2.2 NodePort 조회

```{bash}
kubectl get po,rs,svc

NAME                                      READY   STATUS    RESTARTS   AGE
pod/nodeapp-deployment-55688d9d4b-8pzsk   1/1     Running   0          145m
pod/nodeapp-deployment-55688d9d4b-pslvb   1/1     Running   0          145m
pod/nodeapp-deployment-55688d9d4b-whbk8   1/1     Running   0          145m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/nodeapp-deployment-55688d9d4b   3         3         3       145m

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes         ClusterIP   10.96.0.1      <none>        443/TCP        10d
service/nodeapp-nodeport   NodePort    10.108.30.68   <none>        80:30123/TCP   4m14s
```

#### 9.2.3 NodePort 를 통한 서비스 접속 확인(여러번 수행)

노드의 External 퍼블릭 IP얻기

```{bash}
kubectl get node -o wide # 결과에서 External-IP 를 참조

NAME                                      STATUS   ROLES    AGE   VERSION             INTERNAL-IP   EXTERNAL-IP     OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-istiok8s-default-pool-36a6222b-33vv   Ready    <none>   32h   v1.18.12-gke.1210   10.146.0.21   35.221.70.145   Container-Optimized OS from Google   5.4.49+          docker://19.3.9
gke-istiok8s-default-pool-36a6222b-6rhk   Ready    <none>   32h   v1.18.12-gke.1210   10.146.0.20   35.221.82.6     Container-Optimized OS from Google   5.4.49+          docker://19.3.9
gke-istiok8s-default-pool-36a6222b-xrzj   Ready    <none>   32h   v1.18.12-gke.1210   10.146.0.22   34.84.27.67     Container-Optimized OS from Google   5.4.49+          docker://19.3.9
```

```{bash}
$ curl http://35.221.70.145:30123
You've hit nodeapp-deploy-6dc7c5dd68-wcm7d

$ curl http://35.221.70.145:30123
You've hit nodeapp-deploy-6dc7c5dd68-lh26q

$ curl http://35.221.70.145:30123
You've hit nodeapp-deploy-6dc7c5dd68-r78cj
```

#### 9.2.4 NodePort 삭제

```{bash}
kubectl delete svc nodeapp-nodeport
```



### 9.3 LoadBalancer (GCP 에서 수행)

#### 9.3.1 LoadBalancer 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name:  nodeapp-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nodeapp-pod
```

#### 9.3.2 LoadBalancer 생성 확인

```{bash}
kubectl get svc

NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.36.0.1      <none>        443/TCP        7d21h
nodeapp-lb   LoadBalancer   10.36.14.234   <pending>     80:31237/TCP   33s
```

현재 pending 상태임 20초 정도 지나면

```{bash}
kubectl get svc

NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.36.0.1      <none>           443/TCP        7d21h
nodeapp-lb   LoadBalancer   10.36.14.234   35.221.179.171   80:31237/TCP   45s
```

#### 9.3.3 서비스 확인

```ㅌ{bash}
curl http://35.221.179.171

You've hit nodeapp-deployment-7d58f5d487-r8hq8
```
