# Learning Kubernetes

[toc]

## 12 Ingress 

### 12.1 Ingress 컨트롤러 생성

- nginx Ingress 컨트롤러 생성
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/cloud/deploy.yaml
```

- 생성 확인
```
kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/name=ingress-nginx --watch
```
ingress-nginx-controller pod가 running이면 정상 설치 완료

### 12.2 Deployment 생성

- nginx / goapp deployment 생성

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
      - name: nginx-container
        image: nginx:1.7.9
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapp-deployment
  labels:
    app: goapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: goapp
  template:
    metadata:
      labels:
        app: goapp
    spec:
      containers:
      - name: goapp-container
        image: chungsju/goapp
        ports:
        - containerPort: 8080
```

### 12.3 Service 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name:  nginx-svc
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx

---
apiVersion: v1
kind: Service
metadata:
  name:  goapp-svc
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: goapp
```

### 12.4 Ingress 생성

```{yaml}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-goapp-ingress
spec:
  rules:
  - host: nginx.acorn.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port: 
              number: 80
  - host: goapp.acorn.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: goapp-svc
            port: 
              number: 80
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port: 
              number: 80   
      - path: /goapp
        pathType: Prefix
        backend:
          service:
            name: goapp-svc
            port: 
              number: 80
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port: 
              number: 80        
```

### 12.5 Ingress 조회

```{bash}
kubectl get ingress

NAME                  HOSTS                             ADDRESS   PORTS     AGE
nginx-goapp-ingress   nginx.acorn.com,goapp.acorn.com             80, 443   15s
```

> Ingress 가 완전히 생성되기 까지 시간이 걸립니다. 2~5분 소요

다시 조회 합니다

```{bash}
kubectl get ingress

NAME                  HOSTS                             ADDRESS          PORTS     AGE
nginx-goapp-ingress   nginx.acorn.com,goapp.acorn.com   35.227.227.127   80, 443   13m
```

### 12.6 /etc/hosts 파일 수정 (C:\Windows\System32\drivers\etc\hosts 파일 수정)

```{bash}
sudo vi /etc/hosts

35.227.227.127 nginx.acorn.com goapp.acorn.com
```

### 12.7 서비스 확인

```{bash}
$ crul http://35.227.227.127/

$ crul http://35.227.227.127/nginx  #동작 여부 확인, 안된다면 왜 안되는지?

$ crul http://35.227.227.127/goapp

$ curl http://goapp.acorn.com
hostname: goapp-deployment-d7564689f-6rrzw

$ curl http://nginx.acorn.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>

```
