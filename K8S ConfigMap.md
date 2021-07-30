# Learning Kubernetes

[toc]


## 13. ConfigMap

#### 13.1 도커에서 매개변수 전달

##### 13.1.1 디렉토리 생성 및 소스 코드 작성

```{bash}
mkdir -p ./configmap/docimg/indocker
mkdir -p ./configmap/kubetmp

cd ./configmap/docimg/indocker
vi fortuneloop.sh
```

```{bash}
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1 #파라메터로 첫번째 매개 변수를 INTERVAL 로 저장
echo "Configured to generate neew fortune every " $INTERVAL " seconds"
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep $INTERVAL
done
```

```{bash}
chmod 755 fortuneloop.sh
```

##### 13.1.2 Docker 이미지 생성

> Docker CMD 는 3가지 형태로 사용 가능 합니다.
>
> - CMD [ "실행파일" ,  "파라메터1" , "파라메터2" ] → **실행파일 형태**
> - CMD [ "파라메터1" , "파라메터2"]  → **ENTRYPOINT 의 디펄트 파라메터**
> - CMD command 파라메터1 파라매터2 → **쉘 명령어 형태**

```{bash}
$ vi Dockerfile
```

```{dockerfile}
FROM ubuntu:latest
RUN apt-get update;  apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]  # args가 없으면 10초
```

```{bash}
$ docker build -t chungsju/fortune:args .
$ docker push chungsju/fortune:args
```

> 테스트를 위해 다음과 같이 수행 가능
>
> $ docker run -it chungsju/fortune:args # 기본 10초 수행
>
> $ docker rm -f [docker-id]
>
> $ docker run -it chungsju/fortune:args 15 # 15초로 매개변수 전달
>
> $ docker rm -f [docker-id]

##### 13.1.3 Pod생성

파일명 : config-fortune-pod.yaml

```{bash}
cd ../../kubetmp
vi config-fortune-indocker-pod.yaml
```

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune5s
spec:
  containers:
  - image: chungsju/fortune:args
    args: ["5"]
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

```{bash}
kubectl apply -f ./config-fortune-indockeer-pod.yaml
```

#### 13.2 Yaml 파일을 통한 매개변수 전달

##### 13.2.1 fortuneloop.sh 작성

```{bash}
cd ../
mkdir -p ./docimg/inyaml
cd ./docimg/inyaml
vi fortuneloop.sh
```

```{bash}
#!/bin/bash
# 매개변수르 받지 않고 환경변로에서 바로 참조
trap "exit" SIGINT
echo "Configured to generate neew fortune every " $INTERVAL " seconds"
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune  > /var/htdocs/index.html
    sleep $INTERVAL
done
```

##### 13.2.2 Dockerfile 작성 및 build

```{bash}
$ vi dockerfile
```

```{dockerfile}
FROM ubuntu:latest
RUN apt-get update;  apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]  # args가 없으면 10초
```

```{bash}
$ docker build -t chungsju/fortune:env .
$ docker push chungsju/fortune:env
```

##### 13.2.2 Pod 생성및 매개변수 전달을 위한 yaml 파일 작성

```{bash}
cd ../../kubetmp
vi config-fortune-inyaml-pod.yaml
```

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune30s
spec:
  containers:
  - image: chungsju/fortune:env
    env:
    - name: INTERVAL 
      value: "30"
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

#### 13.3 ConfigMap 을 통한 설정

##### 13.3.1 ConfigMap 생성 및 확인

- configMap 생성

```{bash}
$ kubectl create configmap fortune-config --from-literal=sleep-interval=7
```

> ConfigMap 생성시 이름은 영문자,숫자, 대시, 밑줄, 점 만 포함 할 수 있습니다.
>
> 만약 여러개의 매개변수를 저장 할려면 --from-literal=sleep-interval=5 --from-literal=sleep-count=10 와 같이 from-literal 부분을 여러번 반복해서 입력 하면 됩니다.

- configMap 확인

```{bash}
$ kubectl get cm fortune-config

NAME             DATA   AGE
fortune-config   1      15s

$ kubectl get configmap fortune-config. -o yaml

apiVersion: v1
data:
  sleep-interval: "7"
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-16T09:09:33Z"
  name: fortune-config
  namespace: default
  resourceVersion: "1044269"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 85723e0a-1959-4ace-9365-47c101ebef82
```

##### 13.3.2 ConfigMap을 환경변수로 전달한는 yaml 파일 작성

- Yaml 파일 생성

```{bash}
vi config-fortune-mapenv-pod.yaml
```

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune7s
spec:
  containers:
  - image: chungsju/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
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

##### 13.3.3 Pod 생성 및 확인

- Pod 생성

```{bash}
$ kubectl create -f ./config-fortune-mapenv-pod.yaml
```

- Pod 생성 확인

```{bash}
$ kubectl get po, cm

$ kubectl describe cm

Name:         fortune-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
sleep-interval:
----
7
Events:  <none>
```

- configMap 삭제

```{bash}
$ kubectl delete configmap fortune-config
```

#### 13.4 ConfigMap Volume 사용 

##### 13.4.1 ConfigMap 생성

```{bash}
mkdir config-dir
cd config-dir
vi custom-nginx-config.conf
```

파일명 : custom-nginx-config.conf

```{conf}
server {
	listen				8080;
	server_name		www.acron.com;

	gzip on;
	gzip_types text/plain application/xml;
	location / {
		root	/usr/share/nginx/html;
		index	index.html index.htm;
	}
}
```

```{bash}
$ kubectl create configmap fortune-config --from-file=config-dir
```

##### 13.4.2 configMap volume 이용 Pod 생성

- Yaml 파일 작성 : config-fortune-mapvol-pod.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configvol
spec:
  containers:
  - image: nginx:1.7.9
    name: web-server
    volumeMounts:
    - name: ls
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ports:
    - containerPort: 8080
      protocol: TCP
  volumes:
  - name: config
    configMap:
      name: fortune-config
```

> 서버에 접속해서 디렉토리 구조를 한번 보는것이 좋습니다.
>
> kubectl exec -it nginx-pod bash

```{bash}
$ kubectl apply -f ./config-fortune-mapvol-pod.yaml
```

##### 13.4.3 ConfigMap Volume 참조 확인

```{bash}
$ kubectl get pod -o wide

AME              READY   STATUS    RESTARTS   AGE     IP          NODE
nginx-configvol   1/1     Running   0          9m25s   10.32.0.2   worker01.acorn.com
```

- Response 에 압축(gzip)을 사용 하는지 확인

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080
```

- 마운트된 configMap 볼륨 내용확인

```{bash}
$ kubectl exec nginx-configvol -c web-server ls /etc/nginx/conf.d

nginx-config.conf
sleep-interval

$ kubectl exec -it nginx-configvol sh
```

##### 13.4.4 ConfigMap 동적 변경

- 사전 테스트

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080

HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
Content-Encoding: gzip
```

- configMap 변경

```{bash}
$ kubectl edit cm fortune-config


```

**gzip on** 부분을 **gzip off ** 로 변경 합니다.

```{bash}
apiVersion: v1
data:
  nginx-config.conf: "server {\n  listen\t80;\n  server_name\tnginx.acorn.com;\n\n
    \ gzip off;\n  gzip_types text/plain application/xml;\n  location / {\n    root\t/usr/share/nginx/html;\n
    \   index\tindex.html index.htm;\n  }  \n}\n\n\n"
  nginx-ssl-config.conf: '##Nginx SSL Config'
  sleep-interval: |
    25
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-16T15:58:42Z"
  name: fortune-config
  namespace: default
  resourceVersion: "1115758"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 182302d8-f30f-4045-9615-36e24b185ecb
```

- 변경후 테스트

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080

HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
Content-Encoding: gzip
```

- reload 로 변경 사항 컨테이너에 알리기

```{bash}
$ kubectl  exec nginx-configvol -c web-server -- nginx -s reload
```

- 최종 테스트

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080

HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Fri, 17 Apr 2020 06:10:32 GMT
Content-Type: text/html
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: W/"5e5e6a8f-264"
```

#### 13.5 ConfigMap Volume(파일 추가)

##### 13.5.1 configMap volume 이용 Pod 생성

- Yaml 파일 작성 : config-fortune-mapvol-pod2.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configvol
spec:
  containers:
  - image: nginx:1.7.9
    name: web-server
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx-config.conf # 주의 : configmap 의 key 와 파일명이 일치 해야합니다.
      readOnly: true
    ports:
    - containerPort: 8080
      protocol: TCP
  volumes:
  - name: config
    configMap:
      name: fortune-config
      defaultMode: 0660
```

> nginx 1.7.9 이상 버전, 예를 들면 nginx:latest 로 하면 /etc/nginx/conf.d 폴더내에 default.conf 파일만 존재 합니다. Example_cat tssl.conf 파일은 없습니다. 테스트를 위해서 nginx:1.7.9 버전으로 설정 한것입니다.

```{bash}
$ kubectl apply -f ./config-fortune-mapvol-pod.yaml
```

##### 13.5.2 서비스 및 ConfiMap 확인

- 서비스 확인

```{bash}
$ curl -H "Accept-Encoding: gzip" -I 10.32.0.2:8080
```

- configMap 확인

```{bash}
$ kubectl exec nginx-configvol -c web-server ls /etc/nginx/conf.d
```

##### 13.5.3 ConfigMap 추가

- configMap 추가

```{bash}
$ kubectl edit cm fortune-config
```

아래와 같이 nginx-ssl-config.conf 키값을 추가합니다.

```{bash}
apiVersion: v1
data:
  nginx-config.conf: "server {\n  listen\t80;\n  server_name\tnginx.acorn.com;\n\n
    \ gzip on;\n  gzip_types text/plain application/xml;\n  location / {\n    root\t/usr/share/nginx/html;\n
    \   index\tindex.html index.htm;\n  }  \n}\n\n\n"
  nginx-ssl-config.conf: ""##Nginx SSL Config" # 이부분이 추가됨
  sleep-interval: |
    25
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-16T15:58:42Z"
  name: fortune-config
  namespace: default
  resourceVersion: "1098337"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 182302d8-f30f-4045-9615-36e24b185ecb
```

##### 13.5.4 ConfigMap 추가된 Pod 생성

- 설정 파일 추가 하기위해 yaml 파일 수정

파일명 : config-fortune-mapvol-pod3.yaml

```{yaml}
kind: Pod
metadata:
  name: nginx-configvol
spec:
  containers:
  - image: nginx:1.7.9
    name: web-server
    volumeMounts:
    - name: config #default.conf 파일 교체
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx-config.conf
      readOnly: true
    - name: config #example_ssl.conf 파일 교체
      mountPath: /etc/nginx/conf.d/example_ssl.conf
      subPath: nginx-ssl-config.conf
      readOnly: true
    ports:
    - containerPort: 8080
      protocol: TCP
  volumes:
  - name: config
    configMap:
      name: fortune-config
      defaultMode: 0660
```

```{bash}
$ kubectl apply -f ./ config-fortune-mapvol-pod3.yaml
```

##### 13.5.6 추가된 ConfigMap 확인

- 서비스 확인

```{bash}
$ curl http://10.32.0.4
```

- configMap 확인

```{bash}
$ kubectl exec -it nginx-configvol bash

$ cd /etc/nginx/conf.d
$ ls -al

```

## 14. Secret

```{bash}
mkdir -p ./secret/cert
mkdir -p ./secret/config
mkdir -p ./secret/kubetmp
```

### 14.1 Secret 생성 (fortune-https)

```{bash}
cd ./secret/cert
```

```{bash}
openssl genrsa -out https.key 2048
openssl req -new -x509 -key https.key -out https.cert -days 360 -subj '/CN=*.acron.com'
```

```{bash}
kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert
```

### 14.2 SSL 용 niginx 설정 생성(fortune-config)

```{bash}
cd ./secret/config
vi custom-nginx-config.conf
```

```{bash}
server {
	listen				8080;
  listen				443 ssl;
	server_name		www.acron.com;
	ssl_certificate		certs/https.cert;
	ssl_certificate_key	certs/https.key;
	ssl_protocols		TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers		HIGH:!aNULL:!MD5;
	gzip on;
	gzip_types text/plain application/xml;
	location / {
		root	/usr/share/nginx/html;
		index	index.html index.htm;
	}
}
```

```{bash}
vi sleep-interval
7
```

```{bash}
kubectl create cm fortune-config --from-file=./config
```

### 14.3 POD 생성

- Pod 생성 : 8.3.2 의 yaml 파일 참조
- 파일명 : secret-pod.yaml

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
  - image: chungsju/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
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
    - name: config # 추가
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs # 추가
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443 # 추가
  volumes:
  - name: html
    emptyDir: {}
  - name: config # 추가
    configMap:
      name: fortune-config
      items:
      - key: custom-nginx-config.conf
        path: https.conf
  - name: certs  #추가
    secret:
      secretName: fortune-https
```

```{bash}
kubectl apply -f ./secret-pod.yaml
```

### 14.4 서비스 확인

```{bash}
kubectl port-forward fortune-https 8443:443 &
```

```{bash}
curl https://localhsot:8443 -k
curl https://localhost:8443 -k -v
```




