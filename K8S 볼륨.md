# Learning Kubernetes

[toc]

## 10.Volums

### 10.1 EmptyDir 

#### 10.1.1 Docker 이미지 만들기

아래와 같이 폴더를 만들고 ./fortune_emptyDir 폴더로 이동합니다.

```{bash}
$ mkdir ./fortune_emptyDir
```

아래와 같이 docker 이미지를 작성하기 위해 bash 로 Application을 작성 합니다.

파일명 : fortuneloop.sh

```{bash}
#!/bin/bash
trap "exit" SIGINT
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep 10
done
```

Dockerfile 을 작성 합니다.

```{dockerfile}
FROM ubuntu:latest
RUN apt-get update; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
RUN chmod 755 /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh
```

Dcoker 이미지를 만듭니다.

```{bash}
$ docker build -t chungsju/fortune .
```

Docker 이미지를 Docker Hub 에 push 합니다.

```{bash}
$ docker login
$ docker push chungsju/fortune
```

#### 10.1.2 Deployment 작성

fortune APP을 적용하기 위해 Deployment 를 작성 합니다.

```{bash}
vi fortune-deploy.yaml
```

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortune-deployment
  labels:
    app: fortune
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fortune
  template:
    metadata:
      labels:
        app: fortune
    spec:
      containers:
      - image: chungsju/fortune
        name: html-generator
        volumeMounts:
        - name: html
          mountPath: /var/htdocs
      - image: nginx:alpine
        name: web-server
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
        ports:
          - containerPort: 80
            protocol: TCP
      volumes:
      - name: html
        emptyDir: {}
```

> html 볼륨을 html-generator 및 web-seerver 컨테이너에 모두 마운트 하였습니다.
>
> html 볼륨에는 /var/htdocs 및 /usr/share/nginx/html 이름 으로 서로 따른 컨테이너에서 바라 보게 됩니다.
>
> 다만, web-server 컨테이너는 읽기 전용(reeadOnly) 으로만 접근 하도록 설정 하였습니다.

> emptDir 을 디스크가 아닌 메모리에 생성 할 수도 있으며, 이를 위해서는 아래와 같이 설정을 바꾸어 주면 됩니다.
>
> emptyDir:
>
>  medium: Memory

#### 10.1.3 LoadBalancer 작성

```{bash}
vi fortune-lb.yaml
```

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name: fortune-lb
spec:
  selector:
    app: fortune
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

#### 10.1.4 Deployment 및 Loadbalancer 생성

```{bash}
$ kubectl apply -f ./fortune-deploy.yaml
$ kubectl apply -f ./fortune-lb.yaml
```

#### 10.1.5. 서비스 확인

```{bash}
$ kubectl get service fortune-lb
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
fortune-lb   LoadBalancer   10.92.12.153   34.64.155.68   80:30373/TCP   57s

$curl http://34.64.155.68
```

### 10.2 Git EmptyDir

#### 10.2.1 웹서비스용 Git 리포지토리 생성

Github 계정 생성 (혹은 테스트로 https://github.com/chungsju/k8s-web.git 사용 가능)

#### 10.2.2 Deployment 용 yaml 파일 작성

```{bash}
$ mkdir ./gitvolume
$ cd ./gitvolume
$ vi gitvolume-deploy.yaml
```

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitvolume-deployment
  labels:
    app: gitvolume
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gitvolume
  template:
    metadata:
      labels:
        app: gitvolume
    spec:
      containers:
      - image: nginx:alpine
        name: web-server
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
        ports:
          - containerPort: 80
            protocol: TCP
      volumes:
      - name: html
        gitRepo:
          repository: https://github.com/chungsju/k8s-web.git
          revision: main
          directory: .
```

#### 10.2.3 Deployment 생성

```{bash}
$ kubectl apply -f ./gitvolume-deploy.yaml
```

#### 10.2.4 Pod 서비스 접속

```{bash}
$ kubectl get pods -l app=gitvolume
NAME                                    READY   STATUS    RESTARTS   AGE
gitvolume-deployment-7fbf7b46c5-5vb2n   1/1     Running   0          5m51s
gitvolume-deployment-7fbf7b46c5-8p4nc   1/1     Running   0          5m51s
gitvolume-deployment-7fbf7b46c5-mnkpj   1/1     Running   0          5m51s


$ kubectl port-forward gitvolume-deployment-7fbf7b46c5-5vb2n 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```
> port-forward를 통해 특정 Pod에 포트포워딩 하여 접속합니다.
> 웹브라우저 localhost:8080 으로 접속하여 보기

### 10.3 GCE Persisteent DISK 사용하기

#### 10.3.1. Persistent DISK 생성

- 리전/존 확인

```{bash}
$ gcloud container clusters list
NAME       LOCATION           MASTER_VERSION   MASTER_IP     MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
myk8stest  asia-northeast3-a  1.19.9-gke.1900  34.64.81.177  e2-medium     1.19.9-gke.1900  3          RUNNING
```

- Disk 생성

```{bash}
$ gcloud compute disks create --size=10GiB --zone asia-northeast3-a  mongodb
# 삭제
# gcloud compute disks delete mongodb --zone asia-northeast3-a
```

#### 10.3.2 Pod 생성을 위한 yaml 파일 작성

- 파일명 : gce-pv.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
  - name: mongodb-data
    gcePersistentDisk:
      pdName: mongodb
      fsType: ext4
  containers:
  - image: mongo
    name:  mongodb
    volumeMounts:
    -  name: mongodb-data
       mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```

- Pod 생성

```{bash}
$ kubectl apply -f ./gce-pv.yaml

$ kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
mongodb   1/1     Running   0          8m42s
```

- Disk 확인

```{bash}
$ kubectl describe pod mongodb

...(중략)

Volumes:
  mongodb-data:
    Type:       GCEPersistentDisk (a Persistent Disk resource in Google Compute Engine)
    PDName:     mongodb  # 디스크이름
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
  default-token-dgkd5:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-dgkd5
    Optional:    false

...(중략)

```

#### 10.3.3 Mongodb 접속 및 데이터 Insert

- 접속

```{bash}
kubectl exec -it mongodb mongo
```

- 데이터 Insert

```{bash}
> use mystore
> db.foo.insert({"first-name" : "chungsju"})

> db.foo.find()
{ "_id" : ObjectId("5f9c4127caf2e6464e18331c"), "first-name" : "chungsju" }

> exit
```

#### 10.3.4 MongoDB Pod 재시작

- MongoDB 중단

```{bash}
$ kubectl delete pod mongodb
```

- MongoDB Pod 재생성

```{bash}
$ kubectl apply -f ./gce-pv.yaml
```

- 기존에 Insert 한 데이터 확인

```{bash}
$ kubectl exec -it mongodb mongo

> use mystore

> db.foo.find()
{ "_id" : ObjectId("5e9684134384860bc207b1f9"), "first-name" : "chungsju" }
```

#### 10.3.5 Pod 삭제

```{bash}
$ kubectl delete po mongodb
```

### 10.4 PersistentVolume 및 PersistentVolumeClaim

#### 10.4.1 PersistentVolume 생성

- gce-pv2.yaml 로 작성

```{yaml}
apiVersion: v1
kind: PersistentVolume
metadata:
   name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
   pdName: mongodb
   fsType: ext4
```

```{bash}
kubectl apply -f ./gce-pv2.yaml
```

```{bash}
kubectl get pv
```

#### 10.4.2 PersistentVolumeClaim 생성

gce-pvc.yaml 로 작성

```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

```{bash}
kubectl apply -f ./gce-pvc.yaml
```

```{bash}
kubectl get pvc
```

#### 10.4.3 PV, PVC 를 이용한 Pod 생성

gce-pod.yaml 파일 생성

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

```bash
$ kubectl apply -f ./gce-pod.yaml
```

```{bash}
$ kubectl get po,pv,pvc
```

#### 10.4.4 Mongodb 접속 및 데이터 확인

```{bash}
$ kubectl exec -it mongodb -- mongo

> use mystore

> db.foo.find()

```

### 10.5 Persistent Volume 의 동적 할당

#### 10.5.1 StorageClass 를 이용해 스토리지 유형 정의

- 클라우드에서 제공하는 Default Storage Class  확인 해보기

```{bash}
kubectl get sc

NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
fast                 kubernetes.io/gce-pd    Delete          Immediate              false                  3m37s
premium-rwo          pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   10d
standard (default)   kubernetes.io/gce-pd    Delete          Immediate              true                   10d
standard-rwo         pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   10d
```

- 상세 내역 확인

```{bash}
kubectl describe sc standard

Name:                  standard
IsDefaultClass:        Yes
Annotations:           storageclass.kubernetes.io/is-default-class=true
Provisioner:           kubernetes.io/gce-pd
Parameters:            type=pd-standard # 일반 디스크
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>

kubectl describe sc premium-rwo

Name:                  premium-rwo
IsDefaultClass:        No
Annotations:           components.gke.io/component-name=pdcsi-addon,components.gke.io/component-version=0.8.7
Provisioner:           pd.csi.storage.gke.io
Parameters:            type=pd-ssd  # SSD 
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
```

- GCP 지원 디스크 종류 확인 하기

```{bash}
gcloud compute disk-types list | grep asia-northeast3

local-ssd    asia-northeast3-a          375GB-375GB
pd-balanced  asia-northeast3-a          10GB-65536GB
pd-extreme   asia-northeast3-a          500GB-65536GB
pd-ssd       asia-northeast3-a          10GB-65536GB
pd-standard  asia-northeast3-a          10GB-65536GB
```

- Stroage Class 생성 (파일명 : sc.yaml)

```{yaml}
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zone: asia-northeast3-a  #클러스터를 만든 지역으로 설정 해야함
```

```{bash}
$ kubectl apply -f ./sc.yaml
```

#### 10.5.2 Storage Class 이용한 PVC 생성

- gce-pvc-sclass.yaml

```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: mongodb-pvc-sc
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

```{bash}
$ kubectl apply -f ./gce-pvc-sclass.yaml
```



#### 10.5.3 PV 및 PVC 확인

- pvc 확인

```{bash}
$ kubectl get pvc mongdb-pvc
```

- pv 확인

```{bash}
$ kubectl get pv #Storage Class에 의해 자동 생성된 PV확인
```

#### 10.5.4 PVC 를 이용한 POD 생성

파일명 : gce-pod.yaml 파일 생성

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

## 11. StatefullSet 

### 11.1 애플리케이션 이미지 작성

#### 11.1.1 app.js 작성

```{javascript}
const http = require('http');
const os = require('os');
const fs = require('fs');

const dataFile = "/var/data/kubia.txt";
const port  = 8080;

// 파일 존재  유/무 검사
function fileExists(file) {
  try {
    fs.statSync(file);
    return true;
  } catch (e) {
    return false;
  }
}

var handler = function(request, response) {
//  POST 요청일 경우 BODY에 있는 내용을 파일에 기록 함
  if (request.method == 'POST') {
    var file = fs.createWriteStream(dataFile);
    file.on('open', function (fd) {
      request.pipe(file);
      console.log("New data has been received and stored.");
      response.writeHead(200);
      response.end("Data stored on pod " + os.hostname() + "\n");
    });
// GET 요청일 경우 호스트명과 파일에 기록된 내용을 반환 함
  } else {
    var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
    response.writeHead(200);
    response.write("You've hit " + os.hostname() + "\n");
    response.end("Data stored on this pod: " + data + "\n");
  }
};

var www = http.createServer(handler);
www.listen(port);
```

#### 11.1.2 Docker 이미지 만들기

- Dockerfile 작성

```{dockerfile}
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

- Docker 이미지 build 및 push

```{bash}
$ docker build chungsju/nodejs:sfs .
$ docker login
$ docker push chungsju/nodejs:sfs
```

### 11.2 Sotage Class  생성

```{yaml}
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-standard-retain
provisioner: kubernetes.io/gce-pd
reclaimPolicy: Retain
parameters:
  type: pd-ssd
  zone: asia-northeast3-a
```

### 11.3 서비스 및 Pod 생성

#### 11.3.1 로드밸런서 서비스 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
    name: nodesjs-sfs-lb
spec:
    type: LoadBalancer
    ports:
    - port: 80
      targetPort: 8080
    selector:
        app: nodejs-sfs
```

#### 11.3.2 Pod 생성

```{yaml}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nodejs-sfs
spec:
  selector:
    matchLabels:
      app: nodejs-sfs
  serviceName: nodejs-sfs
  replicas: 2
  template:
    metadata:
      labels:
        app: nodejs-sfs
    spec:
      containers:
      - name: nodejs-sfs
        image: chungsju/nodejs:sfs
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
      storageClassName: sc-standard-retain
```

```{bash}
kubectl apply -f nodejs-sfs.yaml
```

```{bash}
kubectl get pvc   

kubectl get pv

kubectl get svc

NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
service/kubernetes       ClusterIP      10.65.0.1      <none>         443/TCP        7h43m
service/nodesjs-sfs-lb   LoadBalancer   10.65.12.243   34.85.38.158   80:32280/TCP   120m

```

### 11.4 서비스 테스트

- 데이터 조회

```{bash}
curl http://34.85.38.158
```

- 데이터 입력

```{bash}
curl -X POST -d "hi, my name is chungsju-1" 34.85.38.158
curl -X POST -d "hi, my name is chungsju-2" 34.85.38.158
curl -X POST -d "hi, my name is chungsju-3" 34.85.38.158
curl -X POST -d "hi, my name is chungsju-4" 34.85.38.158
```

> 데이터 입력을 반복하에 두개 노드 모드에 데이터를 모두 저장 합니다. 양쪽 노드에 어떤 데이터가 입력 되었는지 기억 하고 다음 단계로 넘어 갑니다.

### 11.5 노드 삭제 및 데이터 보존 확인

- 노드 삭제 

```{bash}
kubectl scale statefulset nodejs-sfs --replicas=1
```

```{bash}
kubectl get pods #Pod가 삭제된 것을 확인
kubectl get pvc  #pvc 존재 여부 확인 
kubectl get pv
```

- 노드 재 확장
```{bash}
kubectl scale statefulset nodejs-sfs --replicas=2
```

- 데이터 보존 확인

```{bash}
curl http://34.85.38.158
```

> 노드가 삭제 되었다 재생성 되도 기존 디스크를 그대로 유지하는 것을 볼 수 있습니다. 





